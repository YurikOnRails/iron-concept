# RFC 0001 — Vision

**Status:** Draft
**Created:** 2026-03-24

---

## What Is IRON?

IRON is an open source industrial automation framework. It replaces SCADA systems that cost
$15,000–$80,000 in licenses, run on Windows XP, and haven't meaningfully changed since 1998.

It is not an improvement on existing SCADA. It is a ground-up rethinking of what industrial
automation software should look like when built with the tools that exist today.

Tagline: **"Industrial automation, forged in Rust."**

---

## The Problem, Stated Plainly

Walk into any industrial plant and you will find software that would be unacceptable in every
other domain of software engineering:

- **Windows XP in production** — not because engineers want it, but because the vendor's
  driver requires it, and replacing the driver requires a $40,000 upgrade project
- **Per-tag licensing** — you pay more as your factory grows. The software is a tax on
  operational visibility.
- **No version control** — configuration lives in binary files on one machine.
  When that machine dies, so does your configuration.
- **Proprietary scripting** — VBScript in Wonderware. CimEdit in iFIX. Skills that exist
  nowhere else in the software world.
- **Historian that can't answer simple questions** — "what was the temperature of reactor 3
  last Tuesday at 14:23?" takes 3 minutes because the query optimizer doesn't understand
  time-series data.
- **$200,000 for software that does less than a modern web application**

Open source alternatives exist (ScadaBR, RapidSCADA, OpenSCADA). They failed not because
the idea was wrong — but because developer experience was an afterthought.
Nobody built the Rails of industrial automation.

**IRON is that thing.**

---

## Philosophy

### Rails for Industrial Automation

Ruby on Rails did not invent web development. It assembled the right pieces in the right way,
made the right thing easy and the wrong thing hard, and wrote documentation that made you
want to build something.

IRON does the same for industrial automation:

- `iron new myplant` — a working development environment in 5 minutes
- YAML for configuration, Elixir for extension, Rust for protocol drivers
- The simulator is not an optional extra — it is built in, so you never need physical hardware
  to develop a screen
- Sensible defaults. No 47-step installation wizard.

### The Two Sacred Paths

Every architecture decision in IRON flows from one constraint:

```
READ:  Sensors → Edge → NATS → Storage → UI
       (one direction, always, no exceptions)

WRITE: Operator → Auth → Audit Log → Command Service → Edge → Machine
       (explicit, authorized, fully logged, confirmed)
```

A visualization bug is physically incapable of sending a command to a machine.
In classical SCADA (WinCC, Wonderware, MasterSCADA), these paths are entangled
in a single monolith. That is the architectural mistake IRON refuses to make.

### Open Is Non-Negotiable

Industrial software that is closed-source, licensed per tag, and requires a certified
partner for installation is not just expensive — it is a hostage situation.
When the vendor raises prices or goes bankrupt, the factory is trapped.

IRON is MIT-licensed. Fork it. Own it. Run it forever.

---

## Target Users

### Persona 1 — Arman, Automation Engineer

**Background:** 18 years at a food processing plant in Almaty. Knows Siemens S7 cold.
Has configured Wonderware, WinCC, and MasterSCADA on three different plants. Knows exactly
where each one breaks and costs money to fix.

**Problem:** His current SCADA license renewal is $60,000. The vendor requires a certified
integrator for any configuration change. The historian loses data when the Windows service
crashes every 3–4 weeks.

**What he wants:** Something he can actually own. Git for his configurations. A historian
that works. No vendor to call at 2am when the system crashes.

**How IRON serves him:** YAML configuration he can put in Git. A CLI that validates his
tag definitions before deployment. A historian built on PostgreSQL — a technology
with 30 years of documentation and a community of millions.

---

### Persona 2 — Zarina, System Integrator

**Background:** Runs a 5-person automation consultancy in Tashkent. Builds SCADA systems
for food factories, water treatment, and small manufacturers. Currently resells Ignition
and Weintek HMIs.

**Problem:** Ignition margins are thin because everyone sells it. Her clients balk at
the license cost. Every project takes 3–4 months because she's configuring the same
things from scratch each time.

**What she wants:** A platform she can build a business around. Faster deployment.
Better margins. Something her clients can own outright.

**How IRON serves her:** A reference architecture she can deploy in days, not months.
An open core where she sells implementation and support, not licenses.
Configurations in Git means she can maintain her clients' systems remotely.

---

### Persona 3 — Bakyt, Greenhouse Owner

**Background:** Runs a 2-hectare greenhouse operation near Bishkek. Grows tomatoes
year-round. Has 40 sensors for temperature, humidity, CO₂, soil moisture.
Currently logs everything manually twice a day.

**What he needs:** Automatic data collection. Alerts when temperature drops at night.
Something he can check from his phone. Budget: $2,000 total.

**How IRON serves him:** A Raspberry Pi with the edge agent. A single-server deployment
for $50/month on a VPS. A dashboard he can view on his phone. Automated alerts via
Telegram. No per-tag licensing — 40 sensors costs the same as 4.

---

### Persona 4 — Nikita, Backend Developer

**Background:** 6 years writing Go and Elixir for fintech. Has never been inside a factory.
Sees IRON on GitHub and realizes that Elixir, NATS, and TimescaleDB are tools he already
knows — applied to a domain he doesn't.

**What he wants:** To contribute to something that matters outside of fintech. A codebase
with genuine engineering challenges. Not another CRUD API.

**How IRON serves him:** Real distributed systems problems — at-least-once delivery,
deadband filtering, WebSocket fan-out at scale. A codebase that uses familiar tools
in an unfamiliar domain. A community that needs exactly his skills.

---

## What Winning Looks Like

### Year 1

- `iron new` works end-to-end on a Raspberry Pi in under 10 minutes
- Modbus TCP/RTU and OPC-UA drivers are production-quality
- 5 real deployments at real plants (not demos)
- 50 GitHub stars from people who understand what they're looking at

### Year 2

- 3 system integrators using IRON as their primary platform
- 100 plants running IRON in production
- iron-web visual editor: you can build a working dashboard without writing code
- First enterprise customer on the paid tier

### Year 5

- IRON is the default choice for new industrial automation projects in CIS markets
- 1,000+ plants in production
- A community of "Ironworkers" — integrators, engineers, and developers
  who have built careers around the platform
- Open source contribution from integrators in 10+ countries
- Paid tier generating enough revenue to fund 3–5 full-time contributors

### What "Ignition did it" means for IRON

Ignition (Inductive Automation) broke per-tag licensing in 2003 and became a
$2–4B company serving 140+ countries. They proved the market is enormous and
the incumbents are vulnerable. IRON does what Ignition did — but open source,
built on Rust + Elixir, with a developer experience that Ignition never had.

Grafana is the other reference point: open source observability → $6B company.
Same model. Same trajectory. Different domain.

---

*IRON exists because the tools to build it finally exist,
and because someone needs to put them together.*
