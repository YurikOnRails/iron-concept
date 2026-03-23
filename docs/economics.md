# Economic Impact

## OEE — The Core Industrial Metric

**OEE (Overall Equipment Effectiveness)** measures how effectively
manufacturing equipment is used:

```
OEE = Availability × Performance × Quality

World average:    55–65%
Good plant:       75–80%
World class:      85%+
```

Each percentage point of OEE at a plant with $10M annual revenue
represents approximately **$80–150k in additional profit** —
because equipment is already bought and depreciated.
OEE improvement is nearly pure margin.

---

## What Modern SCADA Contributes to OEE

| Source | OEE Improvement |
|---|---|
| Reduced unplanned downtime (predictive maintenance) | +3–5% |
| Process parameter optimization (real-time feedback) | +2–5% |
| Faster anomaly detection (trends vs. thresholds) | +1–3% |
| Quality improvement (out-of-spec early detection) | +1–3% |
| **Total potential** | **+5–15% OEE** |

---

## Concrete Example: Mid-Size Plant in Kazakhstan

```
Annual revenue:          $15M
Current OEE:             62%
After IRON:         72% (+10 percentage points)
Additional revenue:      ~$1.5M/year

Implementation cost:     $30–50k (our stack)
vs. traditional SCADA:   $150–300k (Ignition, WinCC)

Payback period:          3–4 months (vs. 18–24 months for traditional)
```

The payback argument is the conversation opener with any plant manager.

---

## Scale Effect: 1,000 Plants

Kazakhstan and Uzbekistan alone have several thousand mid-size industrial facilities.
Conservative estimate of 1,000 plants adopting modern SCADA:

```
Average revenue increase per plant:   $500k/year
1,000 plants:                         $500M/year additional production

This is real goods — more metal, more food, more chemicals,
less waste. Not financial instruments, not redistribution.
Value creation.
```

---

## Why CIS Markets Are the Entry Point

**Lower competition:** Ignition (the leading modern SCADA) has minimal presence
in Kazakhstan and Uzbekistan. The market is served primarily by old Siemens/Schneider
installations and legacy Russian SCADA (MasterSCADA, Trace Mode).

**Data sovereignty:** CIS industrial companies are sensitive about where
operational data lives. A self-hosted open-source solution addresses this directly.
Cloud-first vendors from the US face trust barriers.

**Price sensitivity:** At 1/5th the cost of Ignition and 1/10th the cost of WinCC,
the ROI conversation is easy.

**Local expertise advantage:** An integrator with European education +
CIS industrial domain knowledge + modern software stack is genuinely rare.
That combination is a competitive moat.

---

## NIS2 and IEC 62443 as Market Drivers

The EU's NIS2 directive (effective October 2024) mandates cybersecurity
for industrial operators. This creates a regulatory mandate — and budget —
for OT security improvements.

IEC 62443 is the international standard for industrial cybersecurity.
A SCADA system with documented security architecture and a clear
patch/update mechanism (Git-based, CI/CD) is demonstrably more NIS2-compliant
than a proprietary black box that hasn't been updated since 2019.

This is particularly relevant for European industrial plants and
CIS companies with European partnerships or exports.

---

## The People Effect

**Operators and process engineers** — instead of walking the plant floor
writing readings in a logbook, they see everything on a screen.
Less routine, fewer errors, fewer accidents.

**Instrumentation engineers** — instead of spending weeks investigating
why pressure dropped three weeks ago (and the data is gone),
they open the historian and see exactly what happened.
Less stress, faster decisions.

**Workers** — fewer emergency stops means stable paychecks without downtime.
Predictive maintenance means equipment stops are planned, not chaotic.

**Plant owners** — real-time visibility from a phone.
No waiting for the Monday morning report to know what happened Saturday night.
