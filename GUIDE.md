# RelayPulse — User Guide

How the app works, what every field means, and what it does on your behalf.

- [What it is](#what-it-is)
- [How it works](#how-it-works)
- [Requirements](#requirements)
- [Getting started](#getting-started)
- [Reading a relay card](#reading-a-relay-card)
- [Status: online, stale, offline](#status-online-stale-offline)
- [Auto-fix](#auto-fix)
- [Tools](#tools)
- [Wallets and rewards](#wallets-and-rewards)
- [Settings](#settings)
- [Troubleshooting](#troubleshooting)
- [Licence](#licence)

---

## What it is

RelayPulse is a desktop dashboard for people who run more than one Anyone Protocol
relay. It polls every relay you own on a schedule, shows them side by side, tells
you when one is genuinely unhealthy, and can repair common failures without you
opening a terminal.

It is a **client**, not a service. Nothing runs in the cloud, no account is
created, and your relay list never leaves your machine.

## How it works

```
   your machine                          your relays
┌────────────────┐   SSH (key-based)   ┌──────────────┐
│   RelayPulse   │ ──────────────────▶ │  relay #1    │
│                │        or           │  relay #2    │
│  polling loop  │   HTTPS agent       │  relay #3 …  │
└────────┬───────┘                     └──────────────┘
         │
         │  read-only lookups
         ▼
  Anyone network APIs   (consensus weight, rewards, network totals)
```

Each poll opens a short connection to the relay, reads what is already there
(`systemctl` state, `/proc`, listening ports, the `anon` fingerprint), and closes.
**Nothing is installed on your relays** unless you explicitly ask for it — the one
exception is the optional HTTPS agent, described under [Tools](#tools).

Separately, RelayPulse queries public Anyone APIs to learn what the *network*
thinks of your relays — consensus weight, observed bandwidth, and rewards. Those
are read-only lookups keyed by your relays' fingerprints.

## Requirements

- **Key-based SSH access** to your relays (a passphrase-less key, or one loaded in
  your agent), **or** the RelayPulse HTTPS agent installed on them.
- Nothing else. No database, no daemon, no port to open on your own machine.

Relays are expected to run `anon` as a systemd service — either `anon@default`,
a named instance such as `anon@myrelay`, or multiple instances on one host.
All three layouts are recognised.

## Getting started

1. **Settings → add your relays.** Each entry needs a name, host, SSH user, port,
   and the path to the private key that opens it. Different keys for different
   relays is normal and supported — set the key per relay.
2. **Pick a connection mode.** On first launch RelayPulse asks how it should reach
   your relays:
   - **HTTPS Agent** — fastest, needs the agent installed on each relay.
   - **SSH** — no agent required; works out of the box on any relay you can already
     `ssh` into. Recommended for a first run.
3. **Let it poll once.** Cards fill in as each relay answers. Fingerprints are read
   over SSH and can take a few minutes on a large fleet — several fields stay blank
   until that finishes.

> **Tip for large fleets:** raise the poll interval (Settings → Polling) before
> adding a hundred relays. Every poll opens one connection per relay; at a short
> interval that is a lot of concurrent SSH.

## Reading a relay card

Each card is one relay. Top to bottom:

| Field | Meaning |
| --- | --- |
| **Status pill** | `ONLINE` / `STALE` / `OFFLINE` — see the next section |
| **Host** | Address RelayPulse connects to |
| **Last seen** | Time since the last successful poll |
| **anon:** | State of the `anon` service itself (`ok`, `down`, `?`) |
| **FAMILY** | Whether this relay's `MyFamily` line matches the rest of your fleet |
| **CONNECTION** | Active connections the relay is currently carrying |
| **FINGERPRINT** | The relay's fingerprint, with a copy button |
| **EST. / h · / day** | Estimated earnings. Shows `Busy` when the reward network is rate-limiting — see [Troubleshooting](#troubleshooting) |
| **CPU · RAM · DISK** | Host resources |
| **NET (rx/tx)** | Live throughput |
| **SSH · HTTPS · ANON · PORT 9001** | Four independent health probes |
| **UPTIME · PUBLIC IP · NIC · LOAD** | Host facts |
| **NET WEIGHT** | The network's consensus weight for this relay — see below |

### Net Weight

Consensus weight is the Anyone network's own rating of a relay's capacity.
Clients choose relays roughly in proportion to it, so it is the closest thing to a
direct predictor of traffic — and therefore of rewards.

RelayPulse reads it from the Anyone network by fingerprint, **not** over SSH, so it
reflects how the network sees your relay rather than what the box reports about
itself. Hover the tile for observed bandwidth, whether the relay has been measured,
and whether the network currently lists it as running.

A relay can be perfectly healthy on every SSH probe and still carry a low weight —
that gap is exactly what this tile is for.

## Status: online, stale, offline

RelayPulse deliberately does not call a relay dead on one failed poll:

| State | Meaning |
| --- | --- |
| **Online** | Last poll succeeded |
| **Stale** | One consecutive failure. Usually a blip — a dropped connection, a slow host |
| **Offline** | Two consecutive failures. Treated as a real incident |

This is why a brief network hiccup turns a few cards yellow instead of setting off
alarms across the fleet.

**Health is more than reachability.** A relay answering SSH while its `anon`
service is dead is *not* healthy, and RelayPulse says so with a separate warning
chip rather than a green pill. The four probes — SSH, HTTPS agent, `anon` service,
ORPort reachability — are reported independently so you can see *which* layer
broke.

## Auto-fix

When enabled, RelayPulse can repair a relay on its own. It is deliberately narrow.

**It triggers on exactly three transitions:**

1. A relay goes offline (two consecutive SSH failures)
2. The `anon` service transitions to inactive while the host is still reachable
3. The relay reports `running = false` to the Anyone dashboard

**It deliberately does nothing when:**

- The failure is an SSH credential or host-key error. If RelayPulse cannot
  authenticate, it cannot fix anything either — retrying would only hammer the host
  and risk a fail2ban ban. It logs the relay as skipped and moves on.
- The relay is up with zero active connections. That is idleness, not a fault.

**How a fix is chosen.** With an AI provider configured (your own OpenAI or Claude
API key, entered in Settings), RelayPulse sends the error and recent log lines and
applies the suggested command. With no key — or if the API is unavailable — it
falls back to pattern matching against your configured command list. Both paths are
capped to the commands you have allowed.

Turn it on in **Settings → AI**. It is off until you enable it, and it will not run
without at least one command in the allow-list.

## Tools

| Tool | What it does |
| --- | --- |
| **`anonrc` editor** | Read and write each relay's `anonrc`, apply a preset, or set exit ports across the fleet |
| **MyFamily** | Collect fingerprints from every relay, build the correct `MyFamily` line, and write it to all of them in one pass |
| **HTTPS agent** | Install, remove, or test the optional agent that makes polling faster than SSH. Can be rolled out to the whole fleet at once |
| **Security scan / harden** | Check relays for SSH exposure and fail2ban state, then harden the risky ones in bulk |
| **fail2ban whitelist** | Add an address to the fail2ban whitelist across relays — useful after locking yourself out |
| **Nyx** | Open Nyx on any relay in one click (it finds the control port or socket for you) |
| **Htop / Logs** | Live `htop` and the last `anon` log lines for a relay, in a terminal window |
| **Setup check** | Per-relay audit that verifies the relay is configured the way the network expects |
| **Port check** | Tests whether the relay's ORPort is actually reachable from outside |

## Wallets and rewards

RelayPulse reads reward data straight from the Anyone network:

- **Fingerprints** — collected from your relays and matched against the network's
  registered device list, so you can see which of your relays are actually claimed
  and earning.
- **Wallet binding** — bind a relay to a wallet address from inside the app.
- **Earnings** — per-relay and fleet-wide, with a detector that flags any relay
  earning below the fleet median while its bandwidth and uptime look normal. That
  combination usually means a registration or consensus problem rather than a
  hardware one.

Rewards are read-only. RelayPulse never moves funds and never asks for a private
key or seed phrase.

## Settings

| Section | What it controls |
| --- | --- |
| **Polling** | How often relays are polled. Raise it for large fleets |
| **Defaults** | Default SSH user, port, and key for newly added relays |
| **AI** | Auto-fix on/off, provider, your API key, and the allowed command list |
| **Alarm** | Sound and repeat interval for offline alerts |
| **Tiles** | Which tiles appear on a relay card |
| **Zoom** | Interface scale — useful with many relays on one screen |
| **Test** | Fire a test alarm or a test auto-fix without waiting for a real incident |

Language can be switched between English and Turkish.

## Troubleshooting

**Cards show `—` for a long time after launch.**
Fingerprints and several derived fields are read over SSH. On a large fleet the
first sweep takes minutes. Anything that depends on fingerprints — Net Weight,
per-relay rewards — stays blank until it completes.

**Earnings show `Busy` instead of a number.**
The Anyone reward data lives on-chain and the public endpoint rate-limits under
load. RelayPulse detects this, shows `Busy` rather than a bare dash, and retries by
itself every few minutes. Nothing is wrong with your relays or the app.

**Many relays go offline at once, right after a restart.**
Check whether something else on the same machine is competing for CPU or memory.
Polling a large fleet opens many connections at once; if the host is starved, polls
time out and healthy relays are reported offline. Raising the poll interval fixes
this permanently.

**A relay is `SSH STALE` but `HTTPS OK`.**
The relay is fine; RelayPulse cannot reach it over SSH. Usual causes: the wrong key
for that relay, a changed SSH port, or a fail2ban ban on your own address. Note that
a fail2ban ban usually presents as a *timeout*, not a refused connection.

**Auto-fix logs "skipped — SSH credential or host key problem".**
Working as designed. See [Auto-fix](#auto-fix): RelayPulse will not try to run
remote commands it cannot authenticate for.

## Licence

Free 14-day trial, no account required. After that, a licence key unlocks the app.
The check is offline — the app never reports who you are or what you run.

Questions, bugs, or licence issues: **baris.adiy1974@gmail.com**

---

RelayPulse is an independent tool built by a relay operator, for relay operators.
It is **not affiliated with, endorsed by, or supported by the Anyone Protocol
project**.
