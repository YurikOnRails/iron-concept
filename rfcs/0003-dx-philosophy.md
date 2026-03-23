# RFC 0003 — Developer Experience Philosophy

**Status:** Draft
**Created:** 2026-03-24

---

## The Problem with Industrial Software DX

Every SCADA system has a getting-started guide that looks like this:

1. Install Windows Server 2019
2. Install SQL Server 2019 (Standard or Enterprise, not Express)
3. Install the SCADA platform (45-minute wizard, 12 license screens)
4. Configure the license server
5. Open the "Tag Browser" and create your first tag
6. Open the "Display Builder" — it looks like Microsoft Paint from 2003
7. Read the 800-page manual to understand why your tag isn't updating

Forty-five minutes in, a developer who knows Rust and Elixir has learned nothing
about industrial automation. They've learned how to click through a Windows installer.

**IRON rejects this entirely.**

---

## The `iron new` Vision

```bash
iron new myplant
cd myplant
iron simulate --demo greenhouse
iron dev
```

Open `http://localhost:4000`. You have a live dashboard with simulated sensor data,
a working alarm, and a trend chart. You haven't touched a PLC. You haven't read
a manual. You've spent 5 minutes.

This is not a simplified toy. This is the actual system with simulated data sources.
The simulator generates realistic industrial data — sinusoidal temperature curves,
random walk pressure, step changes that trigger alarms. When you connect real hardware,
you change one line in `config/tags.yaml`.

---

## The `iron` CLI

Every interaction with IRON goes through a single CLI. No GUI configuration tools
that only run on Windows. No vendor-specific IDEs.

```bash
# New project
iron new myplant

# Add a tag from a Modbus source
iron tag add reactor_01.temperature \
  --source modbus://192.168.1.10/holding/0x1000 \
  --type float32 --unit "°C" --deadband 0.5

# Simulate without hardware
iron simulate reactor_01 --temperature "sine:20:180:60s"
iron simulate pump_01 --status "cycle:on:30s:off:5s"

# Validate the entire configuration
iron validate
# ✅ 47 tags valid
# ⚠️  reactor_01.flow — no alarm limits defined
# ❌ pump_02.status — source unreachable (connection refused)

# Generate object scaffolding (tags + dashboard + alarms)
iron generate object reactor_01 --template chemical_reactor

# Deploy to edge device
iron deploy --target edge-01.local
# Compiling Rust agent...  done (23s)
# Uploading configuration... done
# ✅ 47 tags active, real data flowing
# Dashboard: http://myplant.local:4000

# Upgrade
iron upgrade
# Backing up current config...
# Downloading iron 0.8.0...
# Running migrations...
# ✅ Upgraded from 0.7.2 to 0.8.0
```

---

## Three Levels of Entry, One System

The automation engineer who has spent 20 years configuring Siemens is not less valuable
than the developer who knows Rust. IRON is usable by both without either having to
learn the other's domain.

```
Level 1 — Mouse only          (process engineer, operator)
  Visual drag-and-drop dashboard builder
  No configuration files, no code
  Deploy a working screen in 30 minutes

Level 2 — YAML configuration  (instrumentation engineer)
  Tags, alarms, and objects defined in plain text
  Git-tracked, code-reviewed, diff-able
  The simulation command works against YAML definitions

Level 3 — LiveView + Elixir   (developer)
  Custom widgets, custom logic, custom protocol drivers
  Elixir for extension, Rust for performance-critical code
  Standard web development tooling: mix, hex, git, CI/CD
```

These are not separate modes. They are layers of the same system.
A Level 1 user sees the dashboard a Level 3 developer built.
A Level 2 user's YAML is what the Level 1 user's dashboard shows.

---

## The Simulator Is Not Optional

Existing SCADA systems treat hardware simulation as an enterprise add-on or
an afterthought. In IRON, the simulator is a first-class citizen.

A developer should never need physical access to a PLC to:
- Build and test a screen
- Test alarm logic
- Develop a new protocol driver
- Train a new operator

The simulator generates stateful industrial data. A `sine` wave with realistic noise.
A pump that starts, runs for 30 seconds, and stops. A temperature that exceeds a limit
and triggers an alarm. A PLC that goes offline and comes back.

```bash
iron simulate plant --scenario morning_startup
# Simulates: boiler heating sequence, pumps starting in order,
# temperature rising to setpoint, one sensor going UNCERTAIN,
# operator acknowledging alarm
```

This cuts development time in half. It also makes IRON accessible to developers
who have never been on a factory floor — they can learn industrial concepts
through realistic simulation before touching real equipment.

---

## Configuration as Code

Every IRON configuration is text. Tags, alarms, dashboards, user roles — all YAML,
all in a Git repository.

What this means in practice:

```bash
# A configuration change is a pull request
git diff config/tags/reactor_01.yaml

-    limit: 180
+    limit: 175

# Review it. Understand why. Approve it.
# The alarm limit changed. Who changed it? When? Why?
# git blame knows.
```

This is not a feature. In traditional SCADA, configuration lives in a proprietary
binary file on one machine. When that machine dies, so does your configuration history.
When someone changes an alarm limit at 3am, there is no record of who or why.

IRON makes the dangerous thing hard and the auditable thing easy.

---

## DX Comparison

| Capability | IRON | ScadaBR | RapidSCADA | Ignition |
|---|---|---|---|---|
| `new project` command | `iron new` ✅ | Manual setup | Manual setup | 45-min wizard |
| Works on Linux | ✅ | Partial | Partial | ✅ (Java) |
| Works on Raspberry Pi | ✅ | ❌ | ❌ | ❌ |
| Config in Git | ✅ YAML | ❌ Binary DB | ❌ Binary DB | ❌ Binary DB |
| Built-in simulator | ✅ | ❌ | ❌ | Limited |
| No per-tag licensing | ✅ | ✅ | ✅ | ✅ |
| Modern web UI | LiveView | JSP (2003) | WinForms | ✅ |
| REST API | ✅ | Limited | Limited | ✅ (paid) |
| Rust edge agent | ✅ | ❌ | ❌ | ❌ |
| Developer docs quality | RFC-level | Poor | Poor | Good |
| Community | Growing | Inactive | Small | Large |
| Open source license | Apache 2.0 | GPL | MIT | Proprietary |

---

## What "5 Minutes to First Dashboard" Means Concretely

The claim is falsifiable. Here is the exact sequence:

```
00:00  iron new greenhouse && cd greenhouse
00:15  iron simulate --demo greenhouse
00:20  iron dev
00:25  Open http://localhost:4000
       → 12 tags: temperature ×4, humidity ×4, CO₂ ×2, soil moisture ×2
       → Live trend chart, last 1 hour
       → One alarm: "Zone A temperature high" (simulated)
       → Alarm panel with acknowledge button
05:00  You have a working SCADA system for a greenhouse.
       No PLC. No Windows. No license key.
```

At 05:01, you connect real Modbus sensors by editing `config/tags.yaml`.
The dashboard does not change. The only change is the data source.

This is the Rails philosophy applied to industrial automation:
**make the right thing the easy thing.**
