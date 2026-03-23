# IRON RFCs

This directory contains Request for Comments documents for the IRON project.
RFCs are the primary mechanism for proposing and discussing significant decisions
about architecture, philosophy, APIs, and product direction.

An RFC is not a formal specification. It is a structured argument. Write one when
you have a proposal that is significant enough to deserve discussion before implementation.

---

## RFC Index

| RFC | Title | Status |
|---|---|---|
| [0001](0001-vision.md) | Vision | Draft |
| [0002](0002-architecture.md) | Architecture | Draft |
| [0003](0003-dx-philosophy.md) | DX Philosophy | Draft |
| [0004](0004-business-model.md) | Business Model | Draft |
| [0005](0005-security.md) | Security Architecture | Draft |

---

## RFC Statuses

| Status | Meaning |
|---|---|
| **Draft** | Work in progress, not ready for broad review |
| **Review** | Ready for community discussion — open an Issue to comment |
| **Accepted** | Decision made, implementation can begin |
| **Implemented** | Code exists that fulfills this RFC |
| **Superseded** | Replaced by a later RFC (link provided) |
| **Withdrawn** | Abandoned, with explanation |

---

## How to Propose an RFC

1. **Open an Issue first.** Describe the problem you want to solve in 3–5 sentences.
   This surfaces early objections before you write 1,000 words.

2. **Fork the repository** and copy the template below into a new file:
   `rfcs/XXXX-short-title.md` (use the next available number).

3. **Write the RFC.** Follow the template. Be concrete. Show trade-offs.
   The goal is not to win an argument — it is to make the best decision.

4. **Open a Pull Request.** Link it to your Issue. Set status to `Review`.

5. **Discussion happens on the PR.** Expect pushback. Incorporate feedback.

6. **Acceptance** requires at least one maintainer approval and no unresolved
   blocking objections. There is no vote — it is a judgment call by maintainers,
   informed by the discussion.

---

## RFC Template

```markdown
# RFC XXXX — Title

**Status:** Draft
**Created:** YYYY-MM-DD
**Supersedes:** (RFC number, if applicable)

---

## Summary

One paragraph. What are you proposing and why?

---

## Motivation

What problem does this solve? Be specific. If possible, show a concrete example
of the current situation and why it is unsatisfactory.

---

## Proposal

The actual proposal. Be concrete. Include:
- What changes
- What stays the same
- Examples, diagrams, code snippets as needed

---

## Trade-offs

What does this RFC give up? Every design decision has costs.
If you cannot identify any trade-offs, you haven't thought about it hard enough.

---

## Alternatives Considered

What other approaches were evaluated? Why were they rejected?

---

## Open Questions

What is still unresolved? What would change your mind?
```

---

## What Makes a Good RFC

**Good:**
- Solves a real problem with a concrete proposal
- Shows trade-offs honestly
- References prior art (how does Ignition solve this? How does Rails solve this?)
- Written so that an automation engineer and a software developer can both understand it

**Bad:**
- A wish list without a design
- A design without motivation
- "We should use X technology" without explaining what problem X solves that
  the current approach does not

---

## What Does Not Need an RFC

- Bug fixes
- Documentation improvements
- Minor API additions that are obviously consistent with existing design
- Configuration changes

If you're unsure, open an Issue and ask.
