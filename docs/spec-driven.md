# Spec-Driven Development in IRON

> *"Convention over configuration means AI has less to hallucinate."*

In traditional software, a specification is a document.
Someone writes it, someone else reads it, someone else implements it.
Three people, three interpretations, three chances for divergence.

In IRON, the specification is the configuration.
The YAML you write to define a tag is simultaneously:

- The source of truth for what the system should do
- The input to code generation
- The contract that `iron validate` enforces
- The basis for automatically generated tests and simulation scenarios

This is Spec-Driven Development: **write the spec, derive everything else from it.**

---

## The Specification is YAML

```yaml
# config/tags/reactor_01.yaml
# This file is the specification. Not a comment. Not a document.
# The system reads this and knows exactly what to build.

reactor_01:
  temperature:
    source: modbus://plc-01/holding/0x1000
    type: float32
    unit: "°C"
    range: [0, 200]
    scan_rate: 1s
    deadband: 0.5
    alarms:
      high:
        limit: 180
        priority: 2
        message: "High reactor temperature"
      high_high:
        limit: 195
        priority: 1
        message: "Critical — initiate shutdown"
        action: emergency_shutdown
```

From this single file, IRON derives:

```
iron generate tests reactor_01
  → test/unit/reactor_01_test.exs          (ExUnit stubs)
  → test/sim/scenarios/reactor_01.yaml     (simulation scenarios)

iron validate
  → checks source reachability
  → checks alarm limit ordering (high < high_high)
  → checks range consistency (limits within [0, 200])
  → checks action exists (emergency_shutdown defined?)

iron dev
  → generates dashboard widget for reactor_01.temperature
  → generates trend chart
  → generates alarm panel entry

iron field --object reactor_01
  → knows it's an AI signal (float32, 4-20mA implied by range)
  → generates the correct verification steps automatically
```

One spec. Many derived artifacts. No duplication.

---

## iron generate — Deriving from the Spec

### Generate test stubs

```bash
iron generate tests reactor_01
```

Produces ready-to-run test stubs derived directly from the YAML:

```elixir
# test/unit/reactor_01_test.exs
# Generated from config/tags/reactor_01.yaml
# Fill in assertions — structure is derived from spec.

defmodule IRON.Tags.Reactor01Test do
  use ExUnit.Case

  describe "reactor_01.temperature" do
    # Spec: type float32, range [0, 200], unit °C

    test "converts 4mA raw input to range minimum (0.0°C)" do
      # TODO: assert convert_ai(4.0, ...) == 0.0
    end

    test "converts 20mA raw input to range maximum (200.0°C)" do
      # TODO: assert convert_ai(20.0, ...) == 200.0
    end

    test "deadband 0.5 suppresses small changes" do
      # TODO: assert !should_publish(87.3, 87.4, 0.5)
    end

    # Spec: alarm high at 180°C
    test "HIGH alarm triggers at 180.0°C" do
      # TODO: assert alarm triggered at exactly 180.0
    end

    test "HIGH alarm does not trigger at 179.9°C" do
      # TODO: assert no alarm at 179.9
    end

    # Spec: alarm high_high at 195°C, action: emergency_shutdown
    test "HIGH HIGH alarm triggers emergency_shutdown action" do
      # TODO: assert emergency_shutdown called when value >= 195
    end
  end
end
```

### Generate simulation scenarios

```bash
iron generate scenarios reactor_01
```

```yaml
# test/sim/scenarios/reactor_01_normal.yaml
# Generated from config/tags/reactor_01.yaml

name: reactor_01_normal_operation
description: Temperature within normal range — no alarms

timeline:
  - t: 0s
    set: { reactor_01.temperature: 87.5 }

assertions:
  - at: 1s
    expect: { reactor_01.temperature: 87.5, quality: GOOD }
    expect: { reactor_01.alarm_high: false }
    expect: { reactor_01.alarm_high_high: false }

---
# test/sim/scenarios/reactor_01_high_alarm.yaml
# Generated from alarm spec: high.limit = 180

name: reactor_01_high_alarm
description: Temperature crosses HIGH limit

timeline:
  - t: 0s
    set: { reactor_01.temperature: 179.9 }
  - t: 1s
    set: { reactor_01.temperature: 180.0 }

assertions:
  - at: 0s
    expect: { reactor_01.alarm_high: false }
  - at: 1s
    expect: { reactor_01.alarm_high: true }
    expect: { reactor_01.alarm_high_high: false }

---
# test/sim/scenarios/reactor_01_emergency.yaml
# Generated from alarm spec: high_high.action = emergency_shutdown

name: reactor_01_emergency_shutdown
description: Temperature crosses HIGH HIGH limit — emergency action

timeline:
  - t: 0s
    set: { reactor_01.temperature: 195.0 }

assertions:
  - at: 1s
    expect: { reactor_01.alarm_high_high: true }
    expect: { reactor_01.emergency_shutdown: true }
```

### Generate objects with full scaffolding

```bash
iron generate object reactor_01 --template chemical_reactor
```

Creates the full vertical slice from a single command:

```
config/tags/reactor_01.yaml          ← the spec
config/commands/reactor_01.yaml      ← write path (separate, always)
test/unit/reactor_01_test.exs        ← generated test stubs
test/sim/scenarios/reactor_01_*.yaml ← generated scenarios
lib/iron_web/live/reactor_01_live.ex ← LiveView scaffold
```

---

## iron validate — The Spec Checker

Before anything reaches the plant floor, `iron validate` reads every YAML
specification and checks it against known constraints.

```bash
iron validate

✅ 47 tags valid
⚠️  reactor_01.flow — no alarm limits defined
    hint: a flow tag without alarms will not detect pump failures
❌ pump_02.status — source unreachable (connection refused at 192.168.1.11:502)
❌ reactor_01.pressure — alarm high_high (220 bar) exceeds range maximum (200 bar)
    the alarm will never trigger — range and alarm limits are inconsistent
```

`iron validate` knows the rules because the spec defines them:

```
range: [0, 200]  +  alarm high_high.limit: 220
→ ERROR: limit exceeds range — alarm will never trigger

alarm high.limit: 195  +  alarm high_high.limit: 180
→ ERROR: high_high must be greater than high

action: emergency_shutdown  (no emergency_shutdown defined in commands/)
→ ERROR: action references undefined command
```

These are not runtime errors. They are specification errors caught before deployment.

---

## Spec-Driven Development in the AI Era

This is where the architecture becomes genuinely different from what existed before.

### Why YAML specs are AI-native

Large language models are excellent at generating structured text from natural language.
They are unreliable at generating correct imperative code for domains they cannot verify.

The difference for IRON:

```
Traditional approach:
  Engineer: "Add a temperature sensor for reactor 01"
  AI: generates Elixir code, Rust structs, database migrations, LiveView components
  Problem: AI cannot verify correctness — hallucinations reach production

IRON approach:
  Engineer: "Add a temperature sensor for reactor 01, 4-20mA, range 0-200°C,
             alarm at 180, emergency shutdown at 195"
  AI: generates config/tags/reactor_01.yaml  (20 lines of YAML)
  iron validate: catches errors before deployment
  iron generate: derives all implementation from the verified spec
  Problem surface: one small YAML file, checked by a deterministic validator
```

The spec is the AI's output. The implementation is derived deterministically.
AI generates what it is good at (structured data from natural language).
IRON generates what AI is unreliable at (correct Rust, Elixir, SQL).

### The AI workflow

```bash
# 1. Engineer describes the sensor in natural language
# AI generates the YAML spec

# 2. Engineer reviews the spec — 20 lines, easy to read
cat config/tags/reactor_01.yaml

# 3. Validator catches what AI got wrong
iron validate
# ❌ alarm high_high (220) exceeds range maximum (200)
# AI made a mistake — caught before deployment

# 4. Engineer fixes the spec (or asks AI to fix it)
# iron validate passes

# 5. Generate everything from the verified spec
iron generate tests reactor_01
iron generate scenarios reactor_01

# 6. Tests run immediately — the spec is already verified
cargo test && mix test
iron test --sim
```

### Convention as AI safety net

Convention over configuration reduces what AI must invent.
When AI must make fewer decisions, it makes fewer mistakes.

```yaml
# Without convention — AI must invent everything:
reactor_01.temperature:
  modbus_host: 192.168.1.10
  modbus_port: 502
  modbus_function_code: 3
  modbus_register_address: 4096
  modbus_data_type: IEEE754_float_big_endian_word_swap
  engineering_unit_low: 0.0
  engineering_unit_high: 200.0
  ...

# With convention — AI fills in the meaningful parts:
reactor_01:
  temperature:
    source: modbus://192.168.1.10/holding/0x1000
    type: float32
    unit: "°C"
    range: [0, 200]
```

The second form has fewer fields. Fewer fields = fewer decisions for AI.
Fewer decisions = fewer hallucinations. `iron validate` catches the rest.

### iron validate as a contract

When AI generates a spec that passes `iron validate`, the engineer knows:

- All sources are syntactically valid
- All alarm limits are within range
- All referenced commands exist
- All required fields are present
- All type constraints are satisfied

This is a machine-verified contract between the AI's output and the system.
Not a code review. Not a manual check. A deterministic verifier.

```bash
# The rule: nothing generated by AI reaches the plant floor
# without passing iron validate first

iron validate && iron generate tests && iron test --sim
# If all three pass: the spec is correct, tests exist, behavior is verified
# Deploy with confidence
```

---

## Spec-Driven at Every Level

The spec-driven approach applies to every layer of the system:

| Layer | Spec | Derived artifacts |
|---|---|---|
| Tag definition | `config/tags/*.yaml` | Tests, simulation scenarios, dashboard widgets, field verification steps |
| Command definition | `config/commands/*.yaml` | RBAC rules, audit log schema, operator UI buttons |
| Alarm definition | `config/alarms/*.yaml` | Alarm panel entries, escalation logic, Telegram templates |
| Dashboard layout | `config/dashboards/*.yaml` | LiveView screens, mobile views |
| User roles | `config/roles/*.yaml` | RBAC enforcement, UI element visibility |

Everything that can be derived from a spec is derived from a spec.
Everything that cannot be derived is written by a developer.

---

## The Payoff

A system integrator onboards a new plant:

```bash
# Day 1: Engineer describes the plant to AI
# AI generates config/tags/*.yaml for 200 tags

iron validate
# 12 errors found — alarm limits, missing sources, type mismatches
# All caught before touching the PLC

# Engineer fixes the 12 errors — mostly copy-paste mistakes by AI
iron validate
# ✅ 200 tags valid

iron generate tests
iron generate scenarios
# 200 test stubs, 600 simulation scenarios — generated in seconds

iron test --sim
# All generated scenarios pass — behavior matches spec

iron deploy --target edge-01
# 200 tags active on the plant floor

# Day 2: Field verification
iron field
# 200 tags — systematic verification, results in Git
```

200 tags configured, tested, and deployed with verified correctness.
No Excel. No paper checklists. No runtime surprises.

This is what convention over configuration means in the age of AI:
not just a better developer experience — a fundamentally more reliable
path from specification to running system.
