# RFC 0004 — Business Model

**Status:** Draft
**Created:** 2026-03-24

---

## The Open Core Model

IRON is free for the vast majority of users. The core runtime, all protocol drivers,
the web UI, the historian, the alarm engine — everything you need to run a real plant
is open source under the Apache 2.0 license. No feature flags. No telemetry. No "community edition."

The paid tier exists for enterprises that need features that only make sense at scale:
multi-site management, SSO, SLA-backed support, compliance reporting. These features
are not artificially withheld from the free tier — they genuinely require infrastructure
and ongoing support that the open source model cannot sustain.

This is the Grafana model. This is the GitLab model. It works.

---

## Tier Comparison

| Feature | IRON (Free) | IRON Enterprise |
|---|---|---|
| **Core runtime** | ✅ | ✅ |
| **All protocol drivers** | ✅ | ✅ |
| **Web UI + dashboards** | ✅ | ✅ |
| **Historian (TimescaleDB)** | ✅ | ✅ |
| **Alarm engine** | ✅ | ✅ |
| **REST API** | ✅ | ✅ |
| **Git-based configuration** | ✅ | ✅ |
| **Edge agent (Rust)** | ✅ | ✅ |
| **WASM custom modules** | ✅ | ✅ |
| **Unlimited tags** | ✅ | ✅ |
| **Self-hosted** | ✅ | ✅ |
| **Apache 2.0 license** | ✅ | — |
| **Multi-site management** | — | ✅ |
| **Enterprise SSO (SAML, LDAP)** | — | ✅ |
| **Cloud historian (managed)** | — | ✅ |
| **SLA-backed support** | — | ✅ 99.9% uptime |
| **IEC 62443 compliance reports** | — | ✅ |
| **Audit export (SOC2, ISO27001)** | — | ✅ |
| **Dedicated onboarding** | — | ✅ |
| **Priority security patches** | — | ✅ |

---

## Pricing Philosophy

### No per-tag licensing. Ever.

Wonderware charges per tag. WinCC charges per tag. Per-tag licensing is a tax on
operational visibility — the more your factory grows, the more you pay just to see it.

IRON charges per server (for enterprise) or nothing (for self-hosted free tier).
A greenhouse with 40 tags pays the same as a refinery with 100,000 tags.
Growth is never punished.

### Free for small operators. Genuinely.

A farmer with a greenhouse runs IRON free on a $50/month VPS or a Raspberry Pi.
No registration. No "contact sales." No credit card. Just:

```bash
iron new mygreenhouse && iron deploy --target pi.local
```

The free tier is not a limited demo. It is the full product. The goal is to make
modern industrial automation accessible to operators who have been priced out of
the market by $15,000–$80,000 license fees.

### Enterprise pricing is value-based

Enterprise features are priced relative to the value delivered, not the cost to build.
A mid-size plant with $15M annual revenue and 10% OEE improvement from real-time
visibility generates $1.5M/year in additional output. Against that, an enterprise
support contract at $20,000–50,000/year is an obvious decision.

Pricing is not public. Sales conversations start from business impact, not feature lists.

---

## Revenue Streams

### 1. Enterprise SaaS (primary)

Multi-site management and cloud historian hosted by IRON.
Target: industrial holding companies managing 5–50 plants.
Model: per-site subscription + support SLA.

### 2. Support contracts

For enterprises running self-hosted IRON.
Model: tiered response time SLA — 4h, 8h, next-business-day.
Target: plant managers who need a throat to choke when something breaks at 3am.

### 3. System integrator program

Integrators who build on IRON get:
- Certified Ironworker status (visible in the ecosystem directory)
- Early access to new features
- Technical support channel

IRON gets:
- Real-world deployments that drive product quality
- References that enterprise sales can use
- A distribution network without hiring a sales team

### 4. Training and certification

Online courses and in-person workshops for automation engineers making the transition
to modern tooling. Git, YAML, LiveView — through the lens of industrial use cases.

---

## Why This Serves Farmers AND Enterprises

The free tier and paid tier are not in tension. They serve different markets
that reinforce each other:

**Farmers and SMEs** (free tier):
- Cannot afford $15,000–80,000 licenses
- Don't need multi-site management
- Generate community, pull requests, and real-world edge cases
- Demonstrate the technology's accessibility — a powerful marketing story

**Enterprises** (paid tier):
- Already spend $150,000–$500,000 on SCADA deployments
- Need SLA, SSO, compliance reports
- Pay for the development of features that benefit everyone
- Value the open source foundation because it eliminates vendor lock-in risk

A software company with 100,000 free users and 100 enterprise customers
is a sustainable business. The free users are not a charity — they are the proof
that the product works, and the pipeline that enterprise customers trust.

---

## IndustrialPROFI: The Complementary Platform

[IndustrialPROFI](https://industrialprofi.com) is a workforce competency management
platform for industrial enterprises. Built on Rails 8 + Inertia.js + React.

**IRON manages equipment. IndustrialPROFI manages people.**

Together, they form a complete industrial operations platform:

```
IndustrialPROFI:
  Who is qualified to operate reactor_01?
  When is the next certification due for the pump maintenance team?
  Which operators have completed the safety training for this line?

IRON:
  What is reactor_01 doing right now?
  When did the pump last exceed vibration limits?
  What happened on this line at 14:23 last Tuesday?
```

### Integration points

- **Permission sync:** IRON can query IndustrialPROFI to verify that an operator
  holds a current certification before allowing a WRITE command on critical equipment.
  "Operator wants to start reactor_01. Is their certification current?" — one API call.

- **Incident correlation:** an alarm in IRON (equipment event) cross-referenced with
  IndustrialPROFI (who was on shift, what their qualification level was) gives
  a complete picture for root cause analysis and regulatory reporting.

- **Onboarding:** new operators are provisioned in IndustrialPROFI first.
  IRON receives their role and equipment permissions automatically.

### Business model alignment

Both platforms are open core. Both target the CIS industrial market as primary entry point.
Both are built by the same team and share the same philosophy: industrial software
should be as good as the best software in any other domain.

An integrator who deploys IRON for a plant can offer IndustrialPROFI as the natural
next step. One conversation, two products, one integrated system.
The platforms are independently valuable and more powerful together.

---

## The Integrator Business Case

A system integrator building on IRON:

```
Traditional SCADA project (Ignition):
  License cost to client:     $40,000
  IRON vendor margin:         $0 (Inductive Automation takes it)
  Integrator revenue:         Implementation fees only

IRON project:
  License cost to client:     $0 (open source)
  Support contract:           $15,000/year (integrator provides)
  Implementation:             Same as before
  Ongoing:                    Annual support renewals
  Integrator margin:          Higher (no license cut to vendor)
  Client lock-in:             Zero (they own the deployment)
```

The integrator who builds on IRON owns the client relationship completely.
There is no vendor between them and the client.
That is a fundamentally different business.

---

## License Decision: Apache 2.0

IRON is licensed under Apache 2.0. This was a deliberate choice with explicit trade-offs.

### Why not MIT

MIT is simpler but provides no patent grant. Apache 2.0 includes an explicit patent
license from every contributor — important when enterprise legal departments review
dependencies. For a project targeting industrial deployments where patent litigation
is a real risk, the patent clause matters.

### Why not AGPL v3

AGPL is the obvious choice if your primary fear is a large company forking your project,
closing the source, and competing with you. That fear is legitimate — but for IRON
specifically, it is not the primary risk:

- The industrial automation market is niche and requires deep OT expertise.
  AWS is not going to offer managed SCADA.
- IRON's enterprise value is in support, SLA, and enterprise features —
  not in the core runtime. A fork of the runtime is not a threat to the business.
- Enterprise customers in IRON's target markets (CIS, Southeast Asia, Latin America)
  are more likely to be blocked by AGPL legal policies than Western enterprises.
  AGPL creates friction exactly where IRON needs adoption.
- Every successful open-core company at scale uses a permissive license for the core:
  Grafana (Apache 2.0), GitLab (MIT), Metabase (AGPL — but they have VC backing
  to sustain the friction). IRON is bootstrapped. Friction is expensive.

### Why not dual-license (Apache + commercial)

Dual-licensing (the Qt/MariaDB model) is a strong option and remains available
as a future path. It requires:

1. A Contributor License Agreement (CLA) from every contributor, giving IRON
   the right to relicense their code. Without this, dual-licensing is legally impossible.
2. A legal entity to hold the commercial license.
3. A sales process for the commercial license.

These are not impossible, but they are overhead that doesn't make sense at concept phase.
If a large company forks IRON and competes directly — that is when dual-licensing
becomes worth the overhead.

### When this decision should be revisited

Revisit the license if:
- A well-funded competitor forks IRON and offers a hosted service without contributing back
- The enterprise tier gains significant traction and a commercial license would
  generate meaningful revenue (>$500k ARR)
- The contributor base grows large enough that a CLA process is manageable

Until then: Apache 2.0 maximizes adoption, minimizes legal friction, and keeps
the focus on building the product rather than managing licensing complexity.

### What Apache 2.0 means in practice

- **Farmers and SMEs:** use IRON forever, free, with no conditions beyond attribution
- **System integrators:** build commercial services on top of IRON without restriction
- **Enterprises:** deploy self-hosted IRON without legal review issues
- **Contributors:** their contributions are protected by the explicit patent grant
- **Competitors:** can fork, but cannot remove attribution; patent grant is irrevocable
