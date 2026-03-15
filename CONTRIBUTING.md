# Contributing to Titan Core

Titan Core is in the **concept and architecture phase**.
There is no code yet. This is intentional.

Code written without architectural conviction becomes technical debt on day one.
The first goal is conviction — shared, tested, argued over.

---

## What we need most right now

### Domain expertise

If you have worked in industrial automation — PLC programming, process engineering,
system integration, OT security — your feedback is worth more than any pull request.

Tell us what we got wrong. Tell us what constraints we haven't considered.
Tell us what a real plant deployment looks like from your experience.

Open an Issue with label `domain-expertise`.

### Architectural challenges

Every decision in this architecture has trade-offs.
We would rather discover them in a GitHub discussion than after 50,000 lines of code.

Open an Issue with label `architecture` if you believe:
- A technology choice is wrong
- A principle breaks under a specific condition
- There is a simpler solution we missed

### Proof of concept components

If you want to validate a specific component:

| Component | Description |
|---|---|
| Rust Edge Agent | Modbus TCP reader → deadband filter → NATS publisher |
| Tag Server | Phoenix GenServer holding tag state + WebSocket broadcast |
| Visual Editor | Drag-and-drop widget placement on a canvas |
| Simulator | Mock PLC generating realistic industrial data |
| TimescaleDB Schema | Hypertables + continuous aggregates + retention policies |

Open an Issue first to discuss scope. A working PoC for any one component is extremely valuable.

### Evangelism

If this resonates with you — share it.
Industrial software people are not always on GitHub.
LinkedIn, industry forums, local automation communities.

The idea needs to reach the people who understand the problem firsthand.

---

## Principles for contribution

**Honest over optimistic.** If something is hard or unsolved, say so.
This architecture has real challenges. Document them, don't hide them.

**Simple over complete.** A deployed system beats a theoretically perfect one.
Resist adding components.

**Both audiences.** Every decision should consider both the automation engineer
who does not know React and the developer who does not know Modbus.

---

## Language

Issues, PRs, and discussions in English or Russian are both welcome.
When Titan Core has code, it will use English identifiers and comments.
Documentation is maintained in both languages.

---

## Code of conduct

Be respectful. Domain experts and software generalists often talk past each other.
The goal of Titan Core is to bridge that gap.

If you are a developer: the automation engineer has seen failure modes
that would never occur to you. Listen first.

If you are an automation engineer: the developer has practices
that will make your work more reliable and maintainable. Stay curious.
