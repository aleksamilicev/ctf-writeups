# [THM] Cache Me Outside — Writeup
---

https://tryhackme.com/room/cachemeoutside

**Difficulty:** Medium  
**Tags:** OSINT, Email, GitHub, Geolocation
**Solved:** June 2026

---

## Overview

Cache Me Outside is an OSINT-focused room where you're tasked with tracking down a retired hacker who has left traces of his identity scattered across the open internet. Starting from a leaked conversation screenshot, the challenge is to follow the digital breadcrumbs — public profiles, forgotten metadata, and small operational mistakes — until you've built a complete picture of who he is and where he's been.

The tasks are:

1. What is the retired hacker's full name?
2. What email address did he accidentally expose?
3. What is his phone number?
4. In which city is he located?
5. What is the name of the tram station where he got off on the 7th of May, 2026?

---

## Step 1 — Full Name (Komoot)

The initial image provided in the room contained a social profile link. Following it led to a **Komoot** profile:

```
https://www.komoot.com/user/5667624959835
```

The profile displayed the name **Jim Lee**, along with a bio describing an ex-hacker trying to turn his life around through outdoor activities and running. The profile also linked out to a GitHub account: `github.com/jiml33t`.

**Answer: Jim Lee**

---

## Step 2 — Email Address (GitHub Commit Metadata)

The room is named *"Cache Me Outside"* — a strong hint toward cached or archived data. GitHub commit `.patch` files are publicly accessible and expose the author's name and email from their Git config, even when the repository itself reveals nothing.

By fetching the patch file for the initial commit on the `jiml33t/jiml33t` profile repository:

```
https://github.com/jiml33t/jiml33t/commit/<SHA>.patch
```

The output revealed:

```
From: jimleepro1-cell <jimleepro1@gmail.com>
Date: Thu, 16 Apr 2026 03:27:19 -0400
Subject: [PATCH] Initial commit
```

The email was leaked through Git commit metadata — a classic OSINT finding. The commit was authored under an older GitHub account, `jimleepro1-cell`, which gave us an additional username to track.

**Answer: jimleepro1@gmail.com**

---

## Step 3 — Phone Number (Out-of-Office Email)

The Komoot profile had a hidden feature: a **"Secret Menu"** accessible via a password dialog. This was a direct nod to the room name — something cached/hidden that needs to be unlocked.

However, the phone number was found through a simpler technique: **sending an email to the leaked address**.

Jim Lee had configured an **out-of-office auto-reply**, which responded with:

```
Good day,
I will be absent from the office while I prepare for a marathon. 
You can contact me on my phone for anything urgent.

Best Regards,
JL
0x4A4C

JIM LEE
// CYBERSECURITY CONSULTANT
✉ jimleepro1@gmail.com
✆ +40 743 321 239
L33T SECURITY
[ PENTESTING · RED TEAM · CONSULTING ]
```

Note the `0x4A4C` easter egg — the hex encoding of "JL" (Jim Lee's initials). Classic hacker signature. The country code `+40` confirms Romania, consistent with later geolocation findings.

**Answer: +40 743 321 239**

---

## Step 4 — City (Geolocation + OSINT)

The Threads profile `@jiml33t` contained a post with an attached photo. The image showed a road with a visible billboard reading **"IRIGAȚII.RO"** — a Romanian irrigation company.

A quick search confirmed that **Irigații.ro** is located on **Calea Stan Vidrighin (formerly Calea Buziașului)** in **Timișoara, Romania** — notably described on their own site as being *"at the end of tramway line 4."*

**Answer: Timișoara**

---

## Step 5 — Tram Station (Photo Analysis + Local Transit Research)

The same Threads post that contained the photo also included the caption:

> *"Just finished my last run before the big day, hopping on the tram for my well-deserved coffee at my favourite French supermarket."*

Key deductions:
- **Tram** → he boarded near the Irigații.ro location, which is at the terminus of **tramway line 4**
- **French supermarket** → **Auchan** (a French retail chain) has a store on Calea Buziașului nr. 11, Timișoara — directly in this area
- **Last run before the big day** → combined with the Threads post dated 4/28 (a meme about running every weekend), the "big day" referred to a marathon — consistent with the out-of-office message about preparing for a marathon

The tram station at this terminus, serving both the Irigații.ro location and the nearby Auchan, is **Piața Gheorghe Domășnean**.

**Answer: Piața Gheorghe Domășnean**

---

## Summary

| Task | Answer |
|------|--------|
| Full name | Jim Lee |
| Email | jimleepro1@gmail.com |
| Phone | +40 743 321 239 |
| City | Timișoara |
| Tram station (7 May 2026) | Piața Gheorghe Domășnean |

---

## Key Takeaways

- **Git commit metadata is public.** Even if you delete a repo or switch accounts, `.patch` files on GitHub expose the author's email. Always configure `git config user.email` carefully, or use GitHub's no-reply address.
- **Out-of-office replies are an OSINT goldmine.** Sending an email to a leaked address costs nothing and can return full contact details, company names, and operational patterns.
- **Geolocation from images is highly precise.** A single billboard in a photo was enough to identify not just the city, but the exact street and transit stop.
- **Cross-referencing platforms multiplies findings.** No single source gave the full picture — Komoot, GitHub, Threads, and email each contributed a piece.
- **Operational security failures compound.** Each individual mistake (leaked email, public bio, OOO reply) seems minor in isolation. Together, they gave away a complete identity.