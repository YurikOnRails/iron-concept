# RFC 0006 — Development and Growth Roadmap

**Status:** Draft
**Created:** 2026-03-24

---

## 1. Summary

This document describes how IRON gets from a documented concept to a mature open source
platform used in production by factories, farms, and integrators globally.
It covers technical milestones, community building, funding, and team formation.

This is a hard project. The intersection of Rust, Phoenix/LiveView, and industrial
automation expertise is genuinely rare. Solo execution to a mature product is
nearly impossible. This roadmap does not pretend otherwise — it describes how to
build the team, find the funding, and attract the community that makes it possible.

The project has one structural advantage that most open source projects lack:
IndustrialPROFI — a live, revenue-generating product serving the same market —
provides the initial funding, the first customer relationships, and the domain
credibility to make IRON's first steps tractable.

---

## 2. The Honest Challenge

### The intersection problem

IRON requires three things simultaneously that almost nobody has all of:

```
Rust developer      — knows memory safety, async, embedded
                      does NOT know Modbus, PLC scan cycles, OT networks

Phoenix developer   — knows LiveView, PubSub, GenServer, real-time
                      has never been on a factory floor

Automation engineer — knows Siemens S7, deadband filtering, IEC 61131-3
                      does NOT know Git, Rust, or why NATS beats Redis for this
```

The people who sit at the intersection of all three are rare. Looking for them
directly is the wrong approach.

### Why this is an opportunity, not just a problem

Every successful open source project started with a founder who had one of these
skills, built in public, and attracted the others through the work itself.

DHH knew web development, not databases — he found contributors who knew both.
José Valim knew Ruby and wanted something better — he attracted the Erlang/OTP
experts after Elixir launched. Torvalds knew OS kernels but not device drivers —
Linux attracted the driver authors after the kernel was public.

The sequence that actually works:

```
publish RFC → someone comments → opens issue →
writes first PR → becomes contributor → becomes maintainer → becomes co-founder
```

You do not find a team. You create conditions where the right people find you.

### The IndustrialPROFI advantage

Most open source infrastructure projects start with zero revenue and compete
for developer time against paid work. IRON starts differently:

[IndustrialPROFI](https://industrialprofi.com) is an active product managing
workforce competency at industrial enterprises. It generates revenue, serves
the same customer base IRON targets, and provides:

- **Funding:** IndustrialPROFI revenue funds IRON development time directly.
  IRON does not need external funding to start — it needs it to accelerate.
- **First customers:** IndustrialPROFI clients already trust the team and
  understand the value of modern industrial software. They are the warmest
  possible leads for IRON pilots.
- **Domain credibility:** years of selling to plant managers, engineers, and
  integrators in CIS markets is not easily replicated. This is a real moat.
- **Complementary product:** IndustrialPROFI manages people, IRON manages
  equipment. The upsell conversation is natural: "You already use us for
  competency management — here is the monitoring layer."

---

## 3. What We Are Building Toward

### IRON at maturity (year 3–5)

A farmer in India receives a Telegram alert at midnight: her greenhouse temperature
dropped below threshold. She taps a button in the IRON mobile view to check the
trend — the heating system has been cycling erratically for two hours. She calls
her technician. The harvest is saved.

An integrator in Brazil opens a new client project. She runs `iron new cement_plant`,
connects to their Modbus PLC in forty minutes, shows the plant manager a live
dashboard before lunch. The project that used to take three months starts generating
value on day one.

A factory in Poland replaces their end-of-life Wonderware installation. They self-host
IRON on two Hetzner servers for €76/month. The migration takes six weeks instead of
eighteen months. The automation engineer uses Git for the first time in his career
and cannot imagine going back.

### Concrete success metrics

| Milestone | Target | Phase |
|---|---|---|
| First GitHub star from a stranger | — | Phase 0 |
| 50 GitHub stars | — | Phase 0 end |
| First external contributor (merged PR) | 1 person | Phase 1 |
| Working `iron new` demo on Raspberry Pi | — | Phase 1 |
| First production installation | 1 plant | Phase 2 |
| Regular external contributors | 3–5 people | Phase 2 |
| GitHub stars | 2,000 | Phase 2 end |
| First enterprise paying customer | 1 contract | Phase 3 |
| Active community members | 50+ | Phase 3 |
| Countries with active users | 5+ | Phase 3 |
| GitHub stars | 10,000 | Phase 4 |
| Production installations | 50+ | Phase 4 |
| Contributors (lifetime) | 100+ | Phase 4 |
| Annual Recurring Revenue | $100,000+ | Phase 4 |
| Core team | 3–5 people | Phase 4 |

---

## 4. Phase 0 — Foundation (Months 0–2)

**Goal:** make IRON findable by the right people before writing a line of code.

The most common mistake in open source: start coding, then try to find users.
The RFC-first approach is the correct one. People who comment on an RFC before
code exists are the highest-quality future contributors — they care about the
problem, not just the implementation.

### Deliverables

- [x] iron-rfcs repository published with complete RFC documentation (0001–0006)
- [ ] README that stops a senior Rust or Elixir developer mid-scroll
- [ ] GitHub Discussions enabled — one thread per RFC
- [ ] Reserve: `getiron.dev` domain, `@getiron` on X/Twitter and LinkedIn
- [ ] CONTRIBUTING.md with a "first contribution in 30 minutes" path

### Promotion actions

**Elixir Forum** (highest priority — most responsive technical community):
Post in the "Projects" category. Frame it as a technical problem, not a project
announcement. Title example: *"Building a real-time SCADA in Phoenix LiveView —
RFC feedback wanted."* Link the architecture RFC. Ask a specific question.

**Reddit — r/elixir and r/rust:**
Same framing. Specific technical questions outperform announcements.
r/SCADA and r/PLC for domain credibility — different audience,
validates real-world relevance.

**Direct GitHub outreach** to 5 specific contributors:
- `tokio-modbus` maintainers — already solved Modbus in Rust, natural fit
- `opcua` crate contributors — OPC-UA is IRON's Phase 2 protocol
- `nerves` contributors — Elixir embedded, exact overlap with edge deployment
- `phoenix_live_view` frequent contributors — see what LiveView can do in a new domain

Message framing: "I'm building X, read your work on Y, would love your thoughts
on this architectural decision." Not: "please contribute to my project."

**One technical post on dev.to or Elixir Forum:**
Not "I'm building an open source SCADA." Nobody cares yet.
Instead: a specific problem with a specific solution.
Example: *"Backpressure between a Rust Modbus poller and a Phoenix PubSub:
what we learned."* Attract engineers who have the same problem.

### Success criteria

- 50+ GitHub stars
- 3+ meaningful comments on RFCs (not just "looks cool")
- 1 person expressing genuine interest in contributing

---

## 5. Phase 1 — Working Prototype (Months 1–6)

**Goal:** prove the idea works end-to-end, attract first contributors.

Nothing recruits like a working demo. The RFC documentation gets attention.
A Raspberry Pi showing live sensor data on a LiveView dashboard gets commitment.

### Deliverables

- `iron new myplant` command produces a working project scaffold
- Modbus TCP connection to a real PLC or software simulator (ModRSsim2 / diagslave)
- LiveView dashboard showing live tag values with deadband filtering
- One threshold alarm → Telegram notification
- Runs on Raspberry Pi 4 (4GB) without modification
- Docker Compose for local development (no PLC required to start)
- Deployment via Kamal 2 to a Hetzner VPS — documented in 30 minutes

### Finding first contributors

**Good first issues** on GitHub — labeled, scoped to 2–4 hours, with
explicit context. Examples:
- "Implement deadband filter for f32 tag values" (pure Rust, no OT knowledge needed)
- "Add LiveView component for single tag sparkline" (pure Phoenix, no Rust needed)
- "Write integration test for Modbus TCP connection" (clear scope, clear pass/fail)

A contribution guide that gets a new contributor to their first merged PR
in under 30 minutes. If it takes longer, the guide is wrong — fix the guide.

Respond to every PR within 48 hours. Quality does not matter at this phase.
The reviewer relationship matters.

### Promotion actions

**Show HN:** *"Show HN: IRON — open source SCADA in Rust + Phoenix"*
Timing: when the Raspberry Pi demo is working and filmed. Not before.
First comment (written by you): a specific technical decision with trade-offs.
HN rewards honesty about hard problems more than marketing claims.

**ElixirConf EU lightning talk submission:**
15 minutes: "Building a real-time industrial dashboard in Phoenix LiveView."
Focus on the LiveView architecture, not the SCADA domain.
The industrial angle is the hook; the LiveView content is why Elixir people watch.

**Weekly devlog** in GitHub Discussions:
What was built. What failed. What decision was made and why.
Write for the contributor you want to attract, not for the user you want to impress.
Failures and wrong turns are more valuable than polished announcements.

### Grant applications

**NLnet Foundation** — apply at end of Phase 0 / beginning of Phase 1.
IRON fits their "open source infrastructure" and "open standards" focus exactly.
Application is straightforward — describe the project, the milestones, the budget.
Amount: €5,000–€50,000. Timeline: 2–4 months to decision.
This is the highest priority grant application. Apply early.

### Success criteria

- `iron new myplant` → working dashboard in under 10 minutes (documented, filmed)
- 3–5 external contributors with at least 1 merged PR from outside
- 1 pilot partner — IndustrialPROFI client or community contact willing to test
- 500+ GitHub stars
- NLnet application submitted

---

## 6. Phase 2 — First Real Deployment (Months 6–12)

**Goal:** running in production at a real plant, first revenue signal, core team forming.

A single real production installation is worth more than 10,000 GitHub stars.
It surfaces problems that no simulation reveals. It creates a reference case
that makes every subsequent sales conversation shorter.

### Deliverables

- OPC-UA client (required for Siemens S7-1200/1500 and Beckhoff)
- Historical trend charts — TimescaleDB hypertables, continuous aggregates
- Alarm management with acknowledgment and escalation
- Basic RBAC: viewer / operator / engineer / admin
- Self-hosted installation guide (Kamal 2 on Hetzner AX41) — under 30 minutes
- At least one real production installation with a documented case study
- Public status page for hosted demo instance

### First production installation: where to find it

**Priority 1 — IndustrialPROFI clients:**
Existing clients already have a relationship, already trust the team,
already understand the value of modern industrial software.
The conversation: "We built the workforce layer. Here is the equipment layer.
We want one plant to run it for free for six months and give us feedback."
This is the warmest possible lead. Start here.

**Priority 2 — CIS market via existing network:**
Kazakhstan and Uzbekistan: Ignition is nearly absent, Siemens is expensive,
MasterSCADA and Trace Mode are technically obsolete. An affordable self-hosted
alternative with a local team is genuinely attractive.

**Priority 3 — Brazil:**
The ScadaBR community is orphaned. Brazilian automation integrators are experienced,
English-speaking (at the technical level), and actively looking for alternatives.
Engage the ScadaBR GitHub community directly.

**Priority 4 — Poland and Czech Republic:**
Mid-size manufacturers, English-speaking technical teams, active search for
non-German solutions. Close to CIS markets culturally and technically.

### Co-founder search (active phase)

Stop looking passively. Start looking actively.

**Technical co-founder profile:**
Senior Rust or Elixir developer who has worked on distributed systems.
Does not need industrial experience — that transfers. Must have shipped production
software. Must care about the problem, not just the technology.

What they get: domain knowledge that takes years to acquire, product vision,
existing customer relationships, CIS market access.

Where to look:
- Elixir Forum "Jobs" section and direct messages to frequent contributors
- Rust Users Forum — people building embedded/systems projects
- GitHub contributors to tokio, nerves, phoenix_live_view
- ElixirConf and RustConf attendees (in-person is faster than online)

**Business co-founder profile:**
B2B sales experience in industrial or manufacturing sector. Understands plant
manager psychology. Can walk a factory floor and ask the right questions.

What they get: a technical product that works, a technical vision they can sell,
an existing IndustrialPROFI customer base to start from.

Where to look: LinkedIn — automation integrators and SCADA salespeople
who post about frustration with existing tools. They exist.

**Evaluation criteria:**
Values alignment first. Skills second. The question is not "can they do the job"
but "will we still be working together in three years when things are hard."

### Grant applications

**Sovereign Tech Fund (Germany):**
Focus: sustaining open source infrastructure the world depends on.
IRON is too early at Phase 1 for this, but Phase 2 production users
is the threshold they look for. Amount: €100,000–€1,000,000.
This is a major application — worth significant effort.

**Rust Foundation grant:**
When iron-core publishes reusable Rust crates (Modbus, OPC-UA, tag engine),
a Rust Foundation grant is natural. They fund growth of the Rust ecosystem.
Amount: $10,000–$20,000. Lower effort than STF, apply earlier.

### Community building by region

**Brazil:**
Engage the ScadaBR community on GitHub and their forum.
Post in Brazilian Elixir groups (they are large and active).
One Portuguese-language technical post goes further than ten English ones.

**Poland:**
Reach automation integrator LinkedIn groups.
AGH University of Science and Technology (Kraków) has a strong automation department.
Student projects on real open source tooling are mutually beneficial.

**India:**
Smart agriculture boom — greenhouse, irrigation, cold storage monitoring.
Priya's persona (IoT developer building agriculture platform) is a real archetype.
Engage the Indian Elixir and Rust communities, which are growing fast.

### Success criteria

- 1 production installation with documented case study
- 2–3 regular contributors who review PRs and close issues
- 2,000+ GitHub stars
- First revenue: pilot contract or enterprise inquiry
- At least 1 grant application under review
- Co-founder conversation in progress

---

## 7. Phase 3 — Community and Enterprise (Months 12–24)

**Goal:** self-sustaining community, enterprise tier launched, first ARR.

### Deliverables

- Siemens S7 protocol driver (huge installed base in CIS and Eastern Europe)
- SVG mnemonic editor (React island via phx-hook) — visual drag-and-drop
- Enterprise SSO: SAML 2.0 / OIDC / Active Directory
- Multi-site management dashboard
- Plugin system for community-contributed protocol drivers and widgets
- Official Docker images on Docker Hub and GitHub Container Registry
- Comprehensive documentation site at getiron.dev
- Helm charts for Kubernetes deployment (enterprise requirement)
- **PLC Runtime — Phase 3 scope:**
  - Contribute to [`plc-lang/rusty`](https://github.com/PLC-lang/rusty) upstream:
    add WASM compilation target
  - `iron test --sim`: compile Structured Text programs to WASM, run with
    simulated I/O, verify alarm logic and control sequences without hardware
  - iron-core bridge: IEC 61131-3 variables published directly as NATS tags
    (no OPC-UA intermediary when running the IRON runtime)
  - Publish `iron-plc-runtime` crate on crates.io (reusable by the wider community)

### Enterprise tier launch

Define the boundary between free and paid clearly before announcing.
The boundary must feel fair — not arbitrary.

Free forever:
- Core runtime, all protocol drivers, unlimited tags, web UI, historian,
  alarm engine, REST API, single-site deployment

Enterprise (paid):
- Multi-site management, SSO, SLA support, compliance reports (IEC 62443),
  cloud historian, priority security patches, dedicated onboarding

First enterprise customer: find them through IndustrialPROFI relationships.
The plant that already trusts IndustrialPROFI for workforce management
is the natural first enterprise IRON customer. Price the first contract
to cover 3 months of a part-time developer. Revenue is not the goal yet —
feedback and reference case is.

### Community milestones

- First protocol driver contributed by the community (not the core team)
- First integrator company publicly building on IRON
- Regional community call (online, monthly) — start with 5 people
- Active Discord or GitHub Discussions where questions are answered
  by community members, not just maintainers

### Corporate sponsorship

**Hetzner open source program:**
Apply for free or discounted servers for CI/CD and public demo instances.
Hetzner has an open source program and IRON is exactly the kind of project
they want to support — runs on their hardware, generates future customers.

**Hardware vendors:**
Advantech, Moxa, and Pepperl+Fuchs sell edge hardware that IRON would run on.
IRON makes their hardware more attractive to modern developers.
Partnership conversation: "We'll document official support for your hardware
in our deployment guide. You list us in your ecosystem."
Not a financial ask — a mutual visibility arrangement first.

**Elixir ecosystem:**
Dashbit (José Valim's company), Smartrent (Elixir in embedded/IoT).
Not financial sponsors — potential technical contributors and validators.
An endorsement from Dashbit carries significant weight in the Elixir community.

### Success criteria

- 5,000+ GitHub stars
- 10+ external contributors
- 3+ countries with active users
- First enterprise contract signed
- Monthly ARR > $5,000
- getiron.dev live with full documentation

---

## 8. Phase 4 — Scale (Year 2–3)

**Goal:** become the default choice for SME industrial automation globally.

### Deliverables

- Complete protocol suite: DNP3, BACnet, PROFINET, EtherNet/IP
- Cloud historian (managed TimescaleDB for enterprises who don't want self-hosting)
- Mobile operator app (React Native or Progressive Web App)
- IEC 62443 compliance documentation package (enterprise sales requirement)
- Formal integrator partner program with certification
- i18n: Portuguese, Spanish, German, Kazakh, Uzbek, Hindi
- **PLC Runtime — Phase 4 scope:**
  - `iron-plc`: standalone component for soft PLC deployments
  - Linux PREEMPT_RT scan cycle with <1ms jitter on commodity x86/ARM64 hardware
  - Hardware-in-the-loop testing (CI pipeline with physical PLCs)
  - Hardware vendor conversations: Advantech, Kunbus, CLICK PLUS —
    "Official IRON runtime support" as co-marketing arrangement
  - Sponsorship opportunity: hardware vendors gain ecosystem value from an open
    runtime that makes their hardware attractive to modern developers

### Market expansion

50+ integrator partners means IRON deploys without the core team's involvement.
That is the inflection point. Before that, every deployment requires founder attention.
After that, the community deploys IRON faster than the team can track.

Case studies needed — one per industry vertical:
- Greenhouse / agriculture monitoring
- Food and beverage processing
- Water treatment
- Small-scale manufacturing
- Building automation (HVAC)

Each case study is a sales tool for that vertical globally.

### Success criteria

- 20,000+ GitHub stars
- 100+ contributors (lifetime)
- 50+ production installations
- $100,000+ ARR
- Core team of 3–5 people
- Active integrator partner program

---

## 9. How the Team Grows

The founder-to-team sequence that actually works — documented in every successful
open source project:

**Phase 0: Solo founder**
RFC documentation, first prototype, build-in-public begins.
The work is visible. The problems are public. The right people start watching.

**Phase 1: First contributor appears**
Not recruited. They comment on an RFC or open an issue.
They write a small PR. You review it carefully, merge it promptly, thank them publicly.
They come back. This is how it starts.

**Phase 2: Contributor becomes co-maintainer**
After 5–10 merged PRs, a contributor who clearly understands the vision
is invited to co-maintain. They get commit access. They start reviewing
other people's PRs. The bus factor drops from 1.

**Phase 3: Co-maintainer becomes co-founder or first hire**
If they are deeply invested — technically and emotionally — the conversation
about equity or salary happens naturally. Not as recruitment. As recognition.

**Phase 4: Community becomes the team**
Protocol drivers, translations, documentation, support — done by the community,
not the core team. The core team focuses on architecture, security, and enterprise.

### What to offer early contributors (not salary)

- Interesting problems that don't exist elsewhere
- Public credit and visibility in a growing community
- Domain knowledge that transfers to a rare and valuable career niche
- Genuine influence over architecture decisions
- Equity conversation for contributors who go deeper than code

The developer who contributes a Modbus driver to IRON becomes one of the few
Rust developers globally with production OT experience. That is worth something
independent of any IRON outcome.

---

## 10. Funding Strategy Timeline

### Primary funding source: IndustrialPROFI

Before any grant or investor: IndustrialPROFI revenue funds IRON development.

This is the correct model for a bootstrapped open source project:
one revenue-generating product sustains another while it finds its footing.
This is not unusual — it is how most durable open source companies started.

The commitment is explicit: a defined portion of IndustrialPROFI revenue
is allocated to IRON development time each month. This creates a sustainable
development pace that does not depend on grant cycles or investor appetite.

When IRON generates its own revenue, reinvestment decisions are made independently.
The two products are complementary but financially separate.

### Grant timeline

| Grant | Amount | When to apply | Requirements | Notes |
|---|---|---|---|---|
| NLnet Foundation | €5k–€50k | Phase 0 end | Published RFCs, clear milestones | Easiest application, apply first |
| Rust Foundation | $10k–$20k | Phase 1 end | Published Rust crates on crates.io | When iron-core crates are public |
| Rust Foundation (PLC) | $20k–$50k | Phase 3 | `iron-plc-runtime` crate published, WASM target merged to rusty | PLC runtime crates are a compelling Rust ecosystem story |
| Sovereign Tech Fund | €100k–€1M | Phase 2 | Production users, active community | Major application, prepare carefully |
| Prototype Fund | €47,500 | Phase 2 | EU partner organization required | Find German university or NGO partner |
| Horizon Europe / NGI Zero | €50k–€2M | Phase 3 | EU consortium, 6–12 month process | Long lead time, start conversations early |

### Corporate sponsors (non-cash)

| Sponsor | What to ask for | When | Value |
|---|---|---|---|
| Hetzner | Free servers for CI/CD and demo | Phase 1 | €50–200/month infrastructure |
| Advantech / Moxa | Hardware for testing | Phase 2 | Reference hardware, co-marketing |
| Dashbit / Smartrent | Technical endorsement | Phase 2–3 | Community credibility |

### What IRON does not need

Venture capital — at least not before $1M ARR. VC funding for open source
infrastructure at concept phase is expensive equity for problems you can solve
with grants and bootstrapping. If enterprise revenue grows past $1M ARR and
the opportunity is larger than self-funding can capture, the VC conversation
becomes rational. Before that: grants, IndustrialPROFI revenue, and earned
enterprise contracts.

---

## 11. Build in Public — The Playbook

Building in public is not marketing. It is the mechanism by which contributors
find the project and the project finds contributors.

### What to publish and where

**Weekly devlog** (GitHub Discussions, every Friday):
- What was built this week (with code snippets or screenshots)
- What failed or took longer than expected
- One architectural decision made with the trade-offs considered
- What is planned for next week

Length: 300–500 words. Not polished. Not corporate.
The developer who reads that you spent three days debugging a Modbus framing
edge case and found it — that developer trusts you. Polish hides competence.

**Monthly technical deep-dive** (dev.to and Elixir Forum):
One specific technical problem with a specific solution.
Examples that work:

- *"Why we use NATS instead of Phoenix PubSub for the Rust-to-Elixir bridge"*
- *"Deadband filtering in Rust: keeping 100,000 sensor tags from flooding your message broker"*
- *"LiveView at 50,000 tag updates per second: what breaks and how we fixed it"*
- *"TimescaleDB vs InfluxDB for industrial historian: a real comparison"*

Examples that do not work:
- "We are building an open source SCADA system" (nobody cares yet)
- "IRON v0.1 released" (only relevant after you have an audience)

### The Show HN strategy

**Timing:** when you have a working demo on a Raspberry Pi with a screencast.
Not before. HN rewards concrete things. An RFC document without running code
gets 3 upvotes. A working Modbus-to-LiveView demo gets 200.

**Title:** *"Show HN: IRON – open source SCADA built with Rust and Phoenix LiveView"*
Specific technologies in the title. HN readers self-select by technology.

**First comment (written by you, immediately after posting):**
A specific technical challenge with the decision made and why.
Example: "The hardest part so far has been the backpressure model between
the Rust poller and the Phoenix PubSub. Here is what we tried and what we landed on: [...]"
This invites engineers to engage with a real problem, not applaud an announcement.

### How to frame problems to attract contributors

Wrong: "We need help with protocol drivers."
Right: "We are implementing OPC-UA client in Rust using the `opcua` crate.
We have a specific problem with session reconnect after network interruption.
Here is what we tried: [code]. Here is what we observe: [behavior].
Does anyone have experience with this?"

Engineers cannot resist a well-framed problem. It is not a weakness — it is
the mechanism by which open source communities form.

### Engage every comment

In Phase 0 and Phase 1: respond to every GitHub issue, every RFC comment,
every forum reply within 24 hours. Not with a templated response —
with a substantive reply that shows you read what they wrote.

The contributor who receives a thoughtful response within 24 hours
comes back. The one who waits three weeks does not.

---

## 12. Risk Register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Solo burnout before co-founder is found | High | Critical | IndustrialPROFI provides income stability; set sustainable pace; find co-founder by Phase 2 at latest |
| No external contributors by Phase 1 end | Medium | High | Aggressive build-in-public; lower barrier to first PR; direct GitHub outreach to 10+ specific developers |
| First production installation fails publicly | Medium | High | Pilot with IndustrialPROFI client first — controlled environment, trusted relationship |
| Better-funded competitor copies the idea | Medium | Medium | Community moat; first-mover in open source SCADA on Rust+Phoenix is durable |
| Protocol edge cases consume all development time | High | Medium | Scope v0.1 to Modbus TCP only; add protocols after first deployment is stable |
| Grant applications rejected | Medium | Medium | Multiple simultaneous applications; IndustrialPROFI revenue as fallback |
| NLnet application rejected | Low | Low | Strong fit for their criteria; straightforward application |
| Sovereign Tech Fund rejected | Medium | Medium | Apply early, iterate on application; community traction improves chances |
| Key contributor stops contributing | Medium | High | More than one person knows each critical subsystem by Phase 2 |
| IndustrialPROFI revenue declines | Low | High | IRON revenue becomes independent by Phase 3; not permanently dependent |
| Apache 2.0: large company forks and competes | Low | Medium | Niche market, requires OT expertise; community moat more protective than license |

---

## 13. What Success Looks Like — The Real Vision

It is 2029. Bekzod is a plant manager in Tashkent. He has not called his automation
engineer at 3am in fourteen months, because IRON sends him a Telegram message
before problems become failures. When something does break, the historian shows
him exactly what happened and when — not a guess, not a missing logbook entry,
not a "the system was restarted and the data is gone." He does not know what
Rust is. He does not care. His plant runs, and he can see it from his phone.

Aigerim is an automation integrator in Almaty. She runs `iron new` on a client's
laptop during the first meeting, connects to their demo PLC, and shows a live
dashboard before the coffee is finished. Her proposals win not because she is
cheaper than the Siemens integrator — she is not — but because she can show
something working before signing a contract. She has deployed IRON at eleven
plants. She knows the codebase well enough to fix problems herself. There is no
vendor to call. There is no license to renew. The client owns their system.

Carlos is in São Paulo. He found IRON on Hacker News in 2026, opened a GitHub
issue about a Modbus edge case he hit at his food processing plant, got a response
the same day, fixed it with a two-line PR, and has been a core contributor since.
He gave a talk at ElixirConf Brazil in 2027. Three Brazilian integrators are now
using IRON because they heard him speak. He did not plan to become an open source
maintainer. He just had a problem and someone responded.

Arman is 27 years old. He submitted his 50th pull request to iron-core last week —
a new Embassy-based firmware for bare-metal sensor nodes. He is one of perhaps
two hundred people globally who understands both async Rust on microcontrollers
and industrial automation protocols. He did not have that expertise in 2026.
He built it by working on IRON, in public, with people who knew what they were doing.

---

## 14. Open Questions

These are real questions without settled answers. They belong in the document
because pretending they are settled would be dishonest.

**Legal entity:**
When to incorporate, and where. Necessary before accepting grant money in most
jurisdictions. Kazakhstan, Netherlands (Stichting for open source), or Estonia
(e-Residency, common for EU-adjacent founders) are the realistic options.
Decision needed before the first NLnet payment.

**Equity if a co-founder is found:**
No formula works for every situation. The honest answer is that it depends on
timing, contribution, and what they give up. Ranges that are defensible:
40–50% for a co-founder who joins before revenue, 10–25% after first enterprise
contract. Vesting with a 1-year cliff is non-negotiable. Document before any
co-founder conversation.

**Venture capital:**
The current position: do not raise VC before $1M ARR or before the opportunity
is demonstrably larger than bootstrapping can capture. VC money solves
a specific problem — scaling faster than revenue allows. Before that problem
exists, VC creates more problems than it solves (board oversight, exit pressure,
growth-at-all-costs dynamics that conflict with open source health).
Revisit if: a clear path to $10M ARR exists and requires capital to capture.

**Long-term governance:**
BDFL (Benevolent Dictator For Life — the Linux, Python model) is the right
structure for Phase 0–2. A foundation or governance committee becomes relevant
when the community is large enough that BDFL becomes a bottleneck and the
project is valuable enough that governance disputes would be costly.
Target: revisit governance model at 20+ regular contributors.

**Open source sustainability:**
Apache 2.0 is the current license. Dual-licensing (Apache + commercial) is
available as a future path if a large competitor exploits the codebase.
See RFC 0004 for the full license rationale and trigger conditions.
