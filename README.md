# IRON

**Industrial automation for the rest of us.**

Make professional-grade industrial automation accessible to any engineer, factory,
or farmer on the planet — without six-figure licenses, Windows-only runtimes,
or six-month integration projects.

> Inspired by Ruby on Rails. Developer experience is not a feature — it is the foundation.
> `iron new myplant` produces a working real-time dashboard in under five minutes
> on any Linux machine, including a Raspberry Pi.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Status: Concept](https://img.shields.io/badge/status-concept-blue.svg)]()
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](CONTRIBUTING.md)

---

## The Problem

Walk into any industrial plant today and you will find:

- A SCADA system running on Windows XP because the vendor's driver only supports it
- An HMI built with tools from 1998, with a UX that has not changed since
- No version control — configuration lives in binary files on a single machine
- A historian database that takes 3 minutes to return a 7-day trend
- A $200,000 invoice for software that does less than a modern web application
- Zero separation between reading sensor data and sending commands to machines

This is not a niche problem. This is the global standard.

The world's factories run on software that would be unacceptable in any other domain.
We accept it because "that's how industrial software works."

**IRON rejects this premise.**

---

![IRON Architecture](docs/assets/architecture.svg)

---

## Optimized for Happiness

Rails has a concept: software should be optimized for programmer happiness.
Industrial automation has never had this. It has been optimized for certification,
for vendor margins, for backwards compatibility with decisions made in 1995.

IRON is built around a different set of beliefs. They are heretical in the OT world.
We think they are obviously correct.

**Your configuration belongs in Git.**
Not in a binary file on one machine. Not in a database only the vendor's tool can read.
In a text file, in a repository, with history, authors, and diffs.
When something breaks at 3am, you want to know what changed last Tuesday and who changed it.

**Windows is not required.**
It never was. It was a choice the industry made in 1995 and never revisited.
IRON runs on a Raspberry Pi, a $150 mini-PC, or any Linux server.
The factory floor does not need a Windows license.

**Per-tag licensing is exploitation.**
You pay more as your factory grows. The software becomes a tax on operational visibility.
IRON has unlimited tags. Growth is never punished.

**A $150 computer is enough.**
A Beelink EQ12 with passive cooling handles 50,000 tags.
A Raspberry Pi handles a greenhouse with 40 sensors.
The idea that industrial automation requires expensive certified hardware
is a story vendors tell to protect margins, not reliability.

**You should be able to demo before the client signs.**
`iron new myplant` + `iron dev` + a laptop.
A working dashboard with simulated data in five minutes.
If you cannot show something working before the contract, you are selling faith, not software.

**The automation engineer and the developer deserve equal respect.**
The engineer who has configured Siemens S7 for 20 years is not less valuable
than the developer who knows Rust. IRON serves both.
Neither should have to learn the other's tools to do their job.

**Open source is more trustworthy than a black box.**
A proprietary SCADA system you cannot inspect is not more secure because it is certified.
It is less secure because nobody outside the vendor has ever read the code.
IRON is Apache 2.0. Read it. Audit it. Fork it if you need to.

These are not features. They are the beliefs the architecture is built on.
If they resonate — you are in the right place.

---

## The Vision

A factory should be as observable, reliable, and maintainable as the best software systems in the world. Not "good for industrial software." Good by absolute standards.

This means:
- Every configuration change tracked in Git with author, timestamp, and reason
- A developer productive on day one — without learning a proprietary tool
- An automation engineer productive on day one — without learning Elixir
- Sensor data visible in a browser anywhere in the world in under 100ms
- A hardware failure that triggers automatic recovery, not a 3am phone call
- A system that costs $20,000 to deploy, not $200,000

IRON is the architectural blueprint for this factory.

---

## Five Principles

### I. The READ and WRITE paths are sacred

A visualization bug must be physically incapable of sending a command to a machine. This is not a feature. It is a non-negotiable constraint that every component is designed around.

```
READ:   Sensors → Edge → Broker → Storage → UI
        (one direction, always, no exceptions)

WRITE:  Operator → Auth → Audit Log → Command Service → Edge → Machine
        (explicit, authorized, fully logged, confirmed)
```

### II. Intelligence belongs at the edge

The agent next to the PLC is not a relay. It validates, filters, buffers, and protects. When the network goes down — it keeps working. When it comes back — no data was lost. When a sensor fails — the system knows, it does not guess.

### III. One database, not a zoo

Configuration, time-series, alarms, users, permissions — one engine, one query language, one backup. TimescaleDB is PostgreSQL. Everything you already know applies. Cross-domain JOINs in a single query. No synchronization between systems.

### IV. The developer and the engineer are equal

The automation engineer with 20 years of PLC experience who knows nothing about React is not less valuable than the developer who knows Rust and nothing about Modbus. IRON serves both. Three levels of entry, one underlying system.

### V. Open is not optional

Industrial software that is closed-source, licensed per tag, and requires a certified partner for installation is not just expensive — it is fragile. When the vendor raises prices or goes bankrupt, the factory is hostage. IRON is Apache 2.0 licensed. Fork it. Own it. Run it forever.

---

## Architecture

```mermaid
graph TD
    subgraph iron-core ["iron-core (Rust · Edge)"]
        EDGE["Edge Agent\nModbus · OPC-UA · S7 · MQTT\nDeadband · Buffer · WASM modules"]
    end

    subgraph iron-web ["iron-web (Phoenix / LiveView · Server)"]
        PHOENIX["LiveView\nGenServer per tag · 2M WebSocket\nReal-time diffs · Auto-recovery"]
        TSDB["Historian\nTimescaleDB · 8–12x compression\nContinuous aggregates · Full SQL"]
        ALARM["Alarm Engine\nRules · Dedup · Escalation"]
        HMI["Web UI\nDashboards · Trends · Alarms\nSVG editor (React island)"]
    end

    PLC["PLCs & Sensors\nModbus TCP/RTU · OPC-UA · S7 · IO-Link"]
    NATS["NATS JetStream\n10M+ msg/sec · At-least-once · Replay"]

    PLC -->|protocols| EDGE
    EDGE -->|publish| NATS
    NATS --> PHOENIX
    PHOENIX --> TSDB
    PHOENIX --> ALARM
    PHOENIX -->|WebSocket| HMI
```

---

## Technology Stack

| Layer | Technology | Why |
|---|---|---|
| **Edge Agent** | Rust | No GC pauses, memory safety, single binary, native ARM/x86 |
| **Message Broker** | NATS JetStream | Single binary, 30MB RAM, at-least-once, hierarchical topics |
| **Time-Series DB** | TimescaleDB | PostgreSQL-compatible + 8–12x compression + continuous aggregates |
| **Backend** | Elixir / Phoenix | Erlang VM: GenServer per tag, 2M WebSocket connections, supervision trees |
| **Frontend** | Phoenix LiveView + React island | LiveView handles 90% (real-time UI, state sync, WebSocket); React for SVG mimic editor only |
| **PLC Programming** | CODESYS / TwinCAT 3 | IEC 61131-3 ST, Git-friendly, VS Code integration |

---

## Scaling to 100,000 Tags

A mid-size refinery has 100,000 tags updating every second. Three layers make this tractable:

```
100,000 polls/sec
  → deadband filter (Rust)     → ~10,000 publishes/sec  (−90%)
  → GenServer routing (Elixir) → subscribers only
  → client subscription        → 50–200 updates/sec per browser
```

An operator viewing a pump station screen receives updates for the 50–200 tags on that screen. Not 100,000. This is the difference between correct and naive architecture.

---

## Convention Over Configuration

IRON knows what a tag looks like. IRON knows what a Modbus connection needs.
IRON knows what an alarm threshold configuration should contain.

Convention means you write less. Less means less to get wrong — by humans and by AI alike.

```yaml
# This is all you need to monitor a reactor temperature
reactor_01:
  temperature:
    source: modbus://plc-01/holding/0x1000
    type: float32
    unit: "°C"
    deadband: 0.5
    alarms:
      high: { limit: 180, message: "High reactor temperature" }
      high_high: { limit: 195, action: emergency_shutdown }
```

IRON generates the dashboard, the trend chart, and the alarm panel from this.
You write the domain knowledge. IRON handles the infrastructure.

YAML configurations that AI can generate. A Rust runtime that validates them.
`iron validate` catches what the model missed before anything reaches the plant floor.

---

## Quick Start

```bash
# 1. New project — scaffolds everything
iron new myplant && cd myplant

# 2. Run locally with Docker — no PLC, no server, no configuration
iron dev
# → http://localhost:4000 in under 2 minutes

# 3. Deploy to your local server (LAN, data stays on-premise)
iron deploy --target local
# → Kamal 2 deploys via SSH, zero downtime, SSL optional
# → Running in under 5 minutes on any Linux machine

# 4. Connect real hardware when ready — one line change
iron tag add reactor_01.temperature \
  --source modbus://192.168.10.100/holding/0x1000 \
  --type float32 --unit "°C"
```

Local deployment is the default. Your data stays in your network.
Cloud is one config change away — same command, different IP.

---

## Developer Experience

```bash
# Generate a process object with scaffolding
iron generate object reactor_01 --template chemical_reactor

# Simulate without hardware
iron simulate reactor_01 --temperature "sine:20:180:60s"

# Validate before deploying
iron validate
# ✅ 47 tags valid
# ⚠️  reactor_01.flow — no alarm limits defined
# ❌ pump_02.status — source unreachable

# Deploy to edge agent (Raspberry Pi in OT zone)
iron deploy --target edge-01
# → ARM64 image pushed, agent restarted in 4s
# ✅ 47 tags active, real data flowing
```

The simulator is not a convenience — it is a philosophy. A developer should never need physical access to a PLC to build and test a screen. This alone cuts development time by half.

---

## Why This Has Not Been Built Yet

The pieces exist. NATS, TimescaleDB, Elixir, Rust — all mature and production-proven. The Unified Namespace movement evangelizes the broker-centric approach. Factry Historian uses TimescaleDB. Ignition proved SCADA can be web-based.

Nobody has assembled it into one cohesive, open, deployable system. Why?

**The cultural gap.** People who understand OT — PLC scan cycles, Modbus edge cases, why you cannot run Nmap in a production plant — rarely know modern software architecture. People who know Rust and Elixir rarely have factory floor experience. IRON requires both. That intersection is small.

**Conservative buyers are rational.** A plant that runs is not touched. The entry point is greenfield projects and forward-thinking integrators.

**Incumbents have no incentive.** Siemens, Rockwell, and Schneider profit from vendor lock-in. They will not build this.

**The market is ready.** NIS2 in Europe mandates OT cybersecurity. CIS markets want data sovereignty. A generation of automation engineers is retiring and being replaced by people who expect Git and a CLI.

**The window is open.**

---

## Roadmap

```
Phase 0 — Foundation    (months 0–2)   RFC documentation · GitHub · Community building
Phase 1 — Prototype     (months 1–6)   iron new · Modbus TCP · LiveView dashboard · Raspberry Pi
Phase 2 — First Deploy  (months 6–12)  OPC-UA · TimescaleDB trends · RBAC · First production plant
Phase 3 — Community     (months 12–24) S7 driver · SVG editor · Enterprise SSO · Plugin system
Phase 4 — Scale         (year 2–3)     Full protocol suite · 50+ installations · Partner program
```

---

## Economic Impact

Each percentage point of OEE (Overall Equipment Effectiveness) at a $15M/year plant represents ~$150k in additional profit. Modern SCADA with real-time visibility and predictive analytics contributes +5–15% OEE. Payback period: 3–6 months.

Traditional SCADA deployment: $150–300k, 18–24 months payback.
IRON deployment: $20–50k, 3–6 months payback.

The ROI argument is not subtle. See [Economic Impact](docs/economics.md).

---

## Target Markets

**CIS industrial SMEs** — Kazakhstan, Uzbekistan, and broader CIS region. Ignition has minimal presence. Siemens is expensive. Data sovereignty concerns make self-hosted solutions attractive. Local domain expertise is a genuine competitive moat.

**Greenfield industrial projects** — new plants without legacy to protect.

**System integrators** — ready to offer a modern alternative with better margins and faster deployment.

---

## RFCs

Architecture decisions, philosophy, and business model are documented as RFCs:

| RFC | Title | Summary |
|---|---|---|
| [0001](rfcs/0001-vision.md) | Vision | What IRON is, why it exists, what winning looks like |
| [0002](rfcs/0002-architecture.md) | Architecture | iron-core, iron-web, edge deployment, protocol roadmap |
| [0003](rfcs/0003-dx-philosophy.md) | DX Philosophy | `iron new`, 5 minutes to first dashboard, DX manifesto |
| [0004](rfcs/0004-business-model.md) | Business Model | Open core tiers, pricing philosophy, IndustrialPROFI |
| [0005](rfcs/0005-security.md) | Security | READ/WRITE separation, IEC 62443, NIS2 compliance |
| [0006](rfcs/0006-roadmap.md) | Roadmap | Phases 0–4, funding strategy, build-in-public playbook |
| [0007](rfcs/0007-visual-system.md) | Visual System | Three-layer rendering: widgets, SVG mimics, visual editor |

**Supporting docs:**
[Deployment Guide](docs/deployment.md) · [Testing Guide](docs/testing.md) · [TDD Guide](docs/tdd.md) · [Spec-Driven Development](docs/spec-driven.md) · [Hardware Selection](docs/hardware-selection.md) · [Hardware Reference](docs/hardware.md) · [Economic Impact](docs/economics.md)

---

## Contributing

IRON is in the concept phase. The most valuable contributions right now are not pull requests — they are conversations.

- **Challenge the architecture** — open an Issue, argue a better approach
- **Share domain expertise** — what does this get wrong from your factory floor experience?
- **Build a proof of concept** — any single component implemented and tested
- **Spread the idea** — if this resonates with you, share it with someone it might resonate with too

See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## License

Apache 2.0 — own it, fork it, deploy it, build a business on it.

---

*IRON is a concept born from a simple belief: the tools to build better industrial software finally exist. Someone just needs to put them together.*

*If you share that belief — welcome.*
