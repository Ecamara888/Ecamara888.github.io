---
layout: post
title: "Lab 01 — Breaking and defending Modbus/TCP"
date: 2026-10-18 18:00:00 +0100
categories: [labs, ot-security]
---

For my first OT security lab I built a small industrial network inside a VM and used it to
show, hands-on, why the **Modbus/TCP** protocol is so dangerous in legacy environments —
and how monitoring makes up for its lack of built-in security.

## The setup

Two Docker containers inside an isolated Ubuntu VM:

- **OpenPLC** — a simulated PLC (programmable logic controller) serving Modbus/TCP.
- **Conpot** — an industrial honeypot that emulates a Siemens PLC.

Plus a Python virtual environment with `pymodbus`, and `nmap`, `tcpdump`, `tshark` and
`suricata` for analysis.

## What I did

**1. Discovery.** A single `nmap` scan found the PLC's Modbus port open — no access
control at the network layer.

**2. Reading the process.** With a few lines of `pymodbus` I read the PLC's coils and
holding registers. No credentials required.

**3. The attack.** I then wrote to a coil as a rogue client:

```
Before: [False, False, False, False, False, False, False, False]
After:  [True,  False, False, False, False, False, False, False]
```

A physical output flipped from OFF to ON with **no authentication whatsoever**. In a real
plant that could be a motor, a pump or a valve.

**4. Capture.** I captured the traffic with `tcpdump` and decoded it in `tshark`. The
Modbus function codes were right there in clear text — `1` for reads, `5` for the write.
No encryption, trivially replayable.

**5. Detection.** Finally I wrote a Suricata rule to alert on Modbus writes and replayed
the capture through it:

```
[1:1000001:1] OT - Modbus write single coil (FC5) [**] {TCP} 127.0.0.1 -> 127.0.0.1:5502
```

The IDS caught the unauthorised write.

## What I learned

Modbus/TCP was designed in an era when industrial networks were physically isolated, so it
has **no authentication, no integrity and no encryption**. You can't "patch" that into the
protocol. What you *can* do is apply the ideas from **IEC 62443**:

- **Segment** OT from IT networks (zones and conduits).
- **Monitor** continuously with an IDS like Suricata or Zeek, since the protocol can't
  defend itself.
- Enforce **least privilege** so only authorised engineering stations can reach a PLC.

A couple of practical gotchas I hit: `tshark` only decodes Modbus on port 502 by default
(I had to tell it `-d tcp.port==5502,mbtcp`), and Suricata needs `sudo` to read its config.

The full write-up, scripts and detection rules live in my coursework repository (kept
private while the module is in progress).

_All testing was done against local simulators in an isolated lab VM._
