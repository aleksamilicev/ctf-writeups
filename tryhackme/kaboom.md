# [THM] Kaboom — Writeup
---

https://tryhackme.com/room/kaboom

**Difficulty:** Medium
**Tags:** OT, ICS, Modbus, Siemens S7, PLC, SCADA, pymodbus, python-snap7
**Solved:** July 2026

---

## Overview

Kaboom drops you into an OT/ICS environment as the operator. The room's premise: a single crafted Modbus write over-pressurises the main pump, causing a blow-out and flooding the plant with alarms — and somewhere in that chaos, a flag.

The challenge is less about exploitation and more about **understanding industrial protocols and the architecture connecting them**: a PLC simulation (Siemens S7-300 via Snap7), a Modbus TCP interface, and a web-based CCTV simulator that displays plant status — but, critically, doesn't read from the protocol you'd expect.

---

## Enumeration

```bash
nmap -p- 10.113.138.255 -sV
```

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh       OpenSSH 9.6p1 Ubuntu
80/tcp   open  http      Werkzeug httpd 3.1.3 (Python 3.12.3) — "PLC CCTV Simulator"
102/tcp  open  iso-tsap  Siemens S7 PLC (CPU 315-2 PN/DP, Snap7-SERVER emulation)
502/tcp  open  modbus    Modbus TCP
1880/tcp open   —        Node-RED (Express/Node.js, fingerprinted via OpenJS Foundation banner)
8080/tcp open  http      Werkzeug httpd 2.3.7 (Python 3.12.3), /login required
```

Five open services point to a fairly typical ICS lab layout: a web HMI/CCTV view (80), an authenticated web app (8080), Node-RED as a likely middleware/automation layer (1880), and two industrial protocols feeding the simulation — Modbus TCP (502) and Siemens S7 (102).

---

## First (wrong) assumption: Modbus drives the CCTV

The CCTV page polls a simple JSON API every 5 seconds:

```js
fetch('/api/state')   // returns { status, video }
// video field is injected directly into:
fetch(`/video?mode=${video}`)
```

Initial testing read/wrote holding registers 0–15 with small values (0, 1, 1000) using `pymodbus`. Writes were confirmed via a separate read script, but `/api/state` never changed — CCTV stayed on "Cooling Off, Low Temperature".

**Lesson:** confirming a write succeeded at the protocol level says nothing about whether the *application logic* consuming that data actually reacts to it.

---

## Finding the live register

Rather than continuing to brute-force a handful of fixed values, I wrote a small monitor script that reads a wide register range, remembers the previous poll, and only prints what changed:

```python
rr = client.read_holding_registers(address=start, count=chunk_size)
# diff against previous snapshot, print only changed indices
```

Scanning registers 0–999 (in 125-register chunks, due to the Modbus protocol cap per request) showed that **only register 0 was "live"** — fluctuating in a narrow band (~50–67) that matched normal sensor noise for "Cooling Off, Low Temperature".

---

## Mapping the state machine

Writing 65535 (max 16-bit unsigned) to register 0 flipped the CCTV status to **"High Temperature, Cooling ON"** — confirming register 0 was the right target, but this clearly wasn't the final "blow-out" state described in the room.

To map the full threshold structure instead of guessing, I scripted a sweep: write a candidate value, wait for the backend poll cycle, query `/api/state`, log the result. Sweeping log-spaced values from 0 to 65535 produced exactly **two** states across the whole register range:

| Value range | Status |
|---|---|
| Low values (transient, settling) | "Unknown State" / "default" video |
| ~90 and above | "High Temperature, Cooling ON" / "cooling" video |

No value in the entire 16-bit range produced anything beyond "Cooling ON". This ruled out "just push the number higher" as the answer — register 0 alone caps out at a *warning* state, not an explosion. Something else was actively keeping the system stable.

Side investigations that turned out to be dead ends:
- **Path traversal on `/video?mode=`** (`../app.py`, etc.) — sanitised, always fell back to `default.mp4`.
- **Negative/overflow values** — a 16-bit register can't represent anything pymodbus hadn't already sent as 65535 (its two's-complement equivalent of -1).
- **`/static/` directory brute-force** — nothing found.

---

## The actual trigger: Modbus coils, not registers

The CCTV status implied an active "Cooling ON" response — i.e., a safety/regulation mechanism was compensating for the high register value. That pointed away from holding registers (analogue data) and toward **coils** (binary on/off outputs), which is where a cooling/safety-enable flag would typically live in a PLC simulation.

A coil monitor script (same diff-based approach as the register monitor, but using `read_coils`) revealed **coil 15 set to `True`** — the cooling/safety override keeping the pump from going critical even with register 0 pegged at max.

```python
client.write_coil(address=15, value=False)
```

Flipping coil 15 to `False` while register 0 was held at 65535 removed the compensating safety logic. The CCTV state progressed past "Cooling ON" into the final blow-out state, and the flag was retrievable from there.

---

## Note on the S7 (port 102) red herring

A teammate suggested pivoting to the Siemens S7 protocol (port 102, Snap7 emulation) instead of Modbus, since S7-300 PLCs typically expose process data via Data Blocks (DB) rather than Modbus registers. This is a reasonable instinct in real ICS engagements — S7 and Modbus often run in parallel on the same PLC, exposing the *same* underlying process data through two different protocols. In this room specifically, the coil approach via Modbus turned out to be the intended path, but enumerating S7 DBs (`python-snap7`, `list_blocks()` / `db_read()`) is a useful technique to have ready for ICS boxes where the Modbus side is a dead end.

---

## Key Takeaways

- **Verifying a write ≠ verifying an effect.** A successful Modbus write doesn't mean the consuming application reacts to it — always check the actual observable output (here, the CCTV API), not just protocol-level confirmation.
- **Differential monitoring beats blind brute-force.** Logging *changes* across a wide register/coil range instantly separates "live" process values from static noise, instead of guessing a handful of fixed addresses.
- **Map thresholds systematically.** Sweeping a value range and logging the resulting application state turns "does this work?" guessing into a clear picture of the underlying state machine — and tells you when you've hit a ceiling that *can't* be pushed further with the variable you're testing.
- **Holding registers and coils represent different things.** Analogue process values (pressure, temperature) live in registers; binary safety/control flags (cooling enable, valve open/closed, alarm reset) live in coils. A system that "resists" an attack at the register level may need the corresponding coil disabled first.
- **ICS targets often expose the same process through multiple protocols** (Modbus + S7 here). When one path stalls, pivoting to a sibling protocol is a reasonable move — but don't abandon a working lead (the coil) just because a teammate suggests a different protocol.

---

*Tags: OT · ICS · TryHackMe · Modbus · Siemens S7 · PLC · SCADA · pymodbus · python-snap7 · CTF*