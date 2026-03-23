# RFC 0004 — Business Model

**Status:** Draft
**Created:** 2026-03-24

---

## The Open Core Model

IRON is free for the vast majority of users. The core runtime, all protocol drivers,
the web UI, the historian, the alarm engine — everything you need to run a real plant
is open source under the MIT license. No feature flags. No telemetry. No "community edition."

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
| **MIT license** | ✅ | — |
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
