# [THM] Biohazard — Writeup
---

https://tryhackme.com/room/biohazard

**Difficulty:** Medium
**Tags:** Steganography, Cryptography, Web Enumeration, CTF Puzzle, Resident Evil
**Solved:** June 2026

---

## Overview

Biohazard is a puzzle-style CTF room built around the old Resident Evil survival horror games. Instead of a typical vuln-chain attack path, this room is a narrative scavenger hunt — every "item" you collect is a flag, and progressing through the story requires solving layered encoding puzzles, steganography, and a fair amount of patience with red herrings.

The room is split into clear story beats: the Mansion, the Guard House, the Revisit, and the Underground Laboratory — each gated behind items collected in the previous section.

---

## Introduction

```bash
sudo nmap -p- ip -sV --min-rate=1000 -T4
```

```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29
```

Three open ports. The web server on port 80 sets the tone immediately — the homepage reads like the opening cutscene of Resident Evil, introducing the **STARS Alpha team** searching for their missing Bravo team in the woods outside Raccoon City. The story leads us to `/mansionmain`.

---

## The Mansion

Each room of the mansion is a separate web endpoint, and progress is gated by item flags in the format `item_name{32-char-hash}`.

### Emblem Flag

Viewing the source of the main hall page reveals an HTML comment pointing to `/diningRoom/`. There, a wall emblem gives us our first flag directly:

```
emblem{***************************}
```

Trying to slot it into the wall's emblem slot fails — wrong emblem for this slot. Worth remembering for later.

### Lock Pick Flag

Back in the dining room's source code, a base64 comment is hiding:

```
SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=
→ "How about the /teaRoom/"
```

The tea room narrative has Jill fighting off an undead enemy and finding Barry, who hands over a lockpick page directly:

```
lock_pick{***************************}
```

### Music Sheet Flag

The lockpick opens the **bar room**. Inside, a "moonlight sonata" note points to an encoded music note:

```
NV2XG2LDL5ZWQZLFOR5TGNRSMQ3TEZDFMFTDMNLGGVRGIYZWGNSGCZLDMU3GCMLGGY3TMZL5
```

The "moonlight sonata" framing is a red herring — it's not a cipher key for anything. Running the string through CyberChef's Magic recipe identifies it as straight **Base32**:

```
music_sheet{***************************}
```

### Gold Emblem Flag

Entering the music sheet flag into the bar room's piano unlocks a secret area with a gold emblem:

```
gold_emblem{***************************}
```

### Shield Key Flag

The gold emblem doesn't fit back into the same slot — but it fits the **dining room's** original emblem slot from earlier. Submitting it reveals ciphertext with no obvious key:

```
klfvg ks r wimgnd biz mpuiui ulg fiemok tqod...
```

Going back and trying the *regular* emblem flag in the gold emblem's slot unlocks the missing piece — the cipher key, `rebecca`. Running the ciphertext through a **Vigenère cipher** with that key gives:

```
there is a shield key inside the dining room.
the html page is called the_great_shield_key
```

Visiting `/diningRoom/the_great_shield_key.html` hands over:

```
shield_key{***************************}
```

### Blue Gem Flag

The second-floor dining room has a statue with a gem just out of reach. The page source contains a ROT13-encoded comment:

```
Lbh trg gur oyhr trz...
→ "You get the blue gem by pushing the status to the lower floor.
   The gem is on the diningRoom first floor. Visit sapphire.html"
```

Visiting `/diningRoom/sapphire.html` gives:

```
blue_jewel{***************************}
```

---

## The Four Crests — A Layered Decoding Puzzle

Placing the blue gem into the **tiger statue room** unlocks four separate "crest" fragments, scattered across four rooms, each encoded a different number of times:

| Room | Crest | Encoding Layers |
|------|-------|-----------------|
| Tiger Statue Room | Crest 1 | Base64 → Base32 |
| Gallery Room | Crest 2 | Base32 → Base58 |
| Armor Room (needs shield key) | Crest 3 | Base64 → Binary → Hex |
| Attic (needs shield key) | Crest 4 | Base58 → Hex |

CyberChef's Magic recipe handles most of the heavy lifting for identifying each layer. The instructions specify the four crests must be **concatenated in order** (1+2+3+4) and decoded one final time as Base64.

Combined, the crests produce:

```
RlRQIHVzZXI6IGh1bnRlciwgRlRQIHBhc3M6IHlvdV9jYW50X2hpZGVfZm9yZXZlcg==
```

Decoding that Base64 string reveals our FTP credentials:

```
FTP user: ******
FTP pass: ***************************
```

---

## The Guard House

### Finding the Hidden Directory

Logging into FTP with the recovered credentials shows a handful of files, including `important.txt`:

```
Jill,
I think the helmet key is inside the text file, but I have no clue on
decrypting stuff. Also, I come across a /hidden_closet/ door but it was locked.
-Barry
```

This confirms the next target: `/hidden_closet/` — gated behind a key we don't have yet.

### Extracting the Hidden Key Fragments

Three images, three different steganography techniques:

**001-key.jpg — Steghide:**
```bash
steghide info 001-key.jpg
steghide extract -sf 001-key.jpg
```
Empty passphrase works. Extracts `key-001.txt`.

**002-key.jpg — EXIF metadata:**
```bash
exiftool 002-key.jpg
```
The `Comment` field contains a base64 fragment directly in the metadata.

**003-key.jpg — Embedded archive:**
```bash
binwalk 003-key.jpg
binwalk -e 003-key.jpg
```
Reveals a ZIP archive appended to the JPEG, containing `key-003.txt`.

### Assembling the Passphrase

Combining all three fragments in order gives a base64 string:

```
cGxhbnQ0Ml9jYW5fYmVfZGVzdHJveV93aXRoX3Zqb2x0
→ plant42_can_be_destroy_with_vjolt
```

### Decrypting the Helmet Key

That passphrase decrypts the GPG file found earlier:

```bash
gpg --decrypt helmet_key.txt.gpg
```

```
helmet_key{***************************}
```

---

## The Revisit

With the helmet key in hand, two previously locked doors open up.

### SSH Username — Study Room

The study room hands over a `doom.tar.gz` archive:

```bash
tar -xvzf doom.tar.gz
cat eagle_medal.txt
```

```
SSH user: ***************
```

### SSH Password — Hidden Closet

The hidden closet reveals a major story beat — Jill finds the dying STARS Bravo leader, Enrico, who's shot before he can reveal the traitor's name. Two items are left behind: an encrypted MO disk and a wolf medal.

The MO disk content is Vigenère-encrypted:

```
wpbwbxr wpkzg pltwnhro, txrks_xfqsxrd_bvv_fy_rvmexa_ajk
```

We don't have the key yet — this gets resolved later in the Underground Laboratory section. The wolf medal, however, gives the SSH password directly:

```
SSH password: ***********
```

**Bravo team leader:** Enrico (explicitly stated in the dialogue).

---

## Underground Laboratory

Logging in over SSH as `***************` / `***********`.

### Finding Chris

```bash
ls -la
```

A hidden `.jailcell` directory stands out. Reading it plays out a reunion scene between Jill and Chris — Chris reveals **Wesker** is the traitor, and hands over **MO Disk 2**, whose content is the literal Vigenère key: `albert`.

### Decrypting MO Disk 1

Using `albert` as the Vigenère key against the MO Disk 1 text from the Guard House section:

```
weasker login password, stars_members_are_my_guinea_pig
```

This confirms the traitor (**Wesker**) and gives us his login credentials.

### The Ultimate Lifeform

Logging in as `weasker`, a note (`weasker_note.txt`) plays out the climax — Wesker reveals his creation, the **Tyrant**, right before it kills him.

### Root Flag

```bash
id
# uid=1000(weasker) groups=...,27(sudo),...
```

`weasker` is in the `sudo` group:

```bash
find / -perm /4000 2>/dev/null
# /usr/bin/sudo

sudo su
```

Reading `root.txt` closes out the story with Jill, Barry, and Chris escaping the mansion's self-destruct sequence:

```
flag: ********************************
```

---

## Key Takeaways

- This room is a good exercise in *not* over-trusting in-story hints — the "moonlight sonata" reference and the `Automation`-style red herrings exist specifically to waste your time chasing the wrong cipher
- CyberChef's Magic recipe is invaluable when you're stacking multiple unknown encodings — manually guessing Base32 vs Base58 vs Base64 by eye is slow and error-prone
- Steganography tasks rarely use just one technique — `steghide`, `exiftool`, and `binwalk` each catch a different hiding method, so running all three as a habit saves time
- Vigenère ciphers are easy to brute-force once you suspect the key is a name dropped earlier in the narrative — keep track of every character name mentioned

---

*Tags: TryHackMe · Steganography · Cryptography · CyberChef · Vigenère Cipher · GPG · CTF Puzzle*