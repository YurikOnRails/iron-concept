# Testing Guide

IRON is built around a simple belief: an automation engineer should never need
physical access to a PLC to develop, test, or validate a system.
A developer should never need a factory floor to write a protocol driver.

This is the Rails philosophy applied to industrial testing:
**make the right thing the easy thing.**

```bash
iron test          # development — no hardware required
iron field         # field verification — hardware connected
```

---

## Testing Philosophy

Traditional SCADA testing looks like this:

1. Configure tags in the vendor tool
2. Connect to a real PLC
3. Check manually that values appear on screen
4. Write results in an Excel spreadsheet
5. Sign a paper checklist

There is no repeatability. There is no version control. There is no way to run
the same test twice and get the same result. When something breaks at 3am,
there is no test history to consult.

IRON rejects this entirely. Every test is:

- **Repeatable** — same command, same result, every time
- **Version-controlled** — test results stored in Git alongside configuration
- **Automated where possible** — CI runs on every commit
- **Honest about what requires hardware** — simulation is not a substitute
  for field verification, but it eliminates the need for hardware during development

---

## Level 1 — Unit Tests

Fast. No external dependencies. Run on every commit.

### iron-core (Rust)

```bash
cargo test
```

What is tested:

```rust
// Deadband filter — pure logic, no hardware needed
#[test]
fn deadband_suppresses_small_changes() {
    assert!(!should_publish(87.3, 87.4, 0.5));
    assert!(should_publish(87.3, 87.9, 0.5));
}

// Data quality transitions
#[test]
fn quality_degrades_on_timeout() {
    let tag = Tag::new(source, scan_rate: 1s, timeout: 3s);
    advance_time(4s);
    assert_eq!(tag.quality(), Quality::BAD);
}

// Engineering unit conversion
#[test]
fn raw_to_engineering_units() {
    // 4mA → 0°C, 20mA → 200°C, midpoint 12mA → 100°C
    assert_eq!(convert_ai(12.0, 4.0, 20.0, 0.0, 200.0), 100.0);
}

// Alarm evaluation
#[test]
fn high_high_alarm_triggers_at_limit() {
    let alarm = Alarm::new(limit: 195.0, priority: 1);
    assert_eq!(alarm.evaluate(194.9), AlarmState::Normal);
    assert_eq!(alarm.evaluate(195.0), AlarmState::Active);
}
```

### iron-web (Elixir / ExUnit)

```bash
mix test
```

What is tested:

```elixir
# Tag GenServer — state management and PubSub broadcast
test "tag process updates value and notifies subscribers" do
  {:ok, pid} = TagServer.start_link(tag: "reactor_01.temperature")
  TagServer.update(pid, value: 87.5, quality: :good)
  assert_receive {:tag_update, %{value: 87.5, quality: :good}}
end

# Alarm acknowledgment — state transition
test "acknowledged alarm clears active state" do
  alarm = create_alarm(tag: "reactor_01.temperature", state: :active)
  Alarms.acknowledge(alarm, user: operator_user())
  assert Alarms.get(alarm.id).state == :acknowledged
end

# RBAC — operator cannot access admin routes
test "operator role cannot access user management" do
  conn = build_conn() |> log_in_user(operator_user())
  conn = get(conn, ~p"/admin/users")
  assert redirected_to(conn) == ~p"/dashboard"
end
```

### Phoenix LiveView layer

Phoenix provides `Phoenix.LiveViewTest` — a full LiveView test helper that mounts
a LiveView, drives interactions, and asserts on the rendered DOM without a browser.
This covers the last mile: tag value → WebSocket diff → browser DOM.

```elixir
# Tag value update reaches the browser DOM
test "dashboard shows updated tag value" do
  {:ok, view, _html} = live(conn, ~p"/dashboard/reactor_01")

  TagServer.update("reactor_01.temperature", value: 95.0, quality: :good)

  assert render(view) =~ "95.0"
  assert render(view) =~ "°C"
  assert render(view) =~ "GOOD"
end

# Quality BAD shows visual warning in DOM
test "BAD quality tag renders warning state" do
  {:ok, view, _html} = live(conn, ~p"/dashboard/reactor_01")

  TagServer.update("reactor_01.temperature", value: 0.0, quality: :bad)

  assert render(view) =~ "BAD"
  assert view |> element("[data-tag='reactor_01.temperature']") |> render() =~
           "quality-bad"
end

# Alarm appears in alarm panel
test "active alarm appears in alarm panel" do
  {:ok, view, _html} = live(conn, ~p"/dashboard/reactor_01")

  Alarms.trigger("reactor_01.temperature", :high, value: 183.0)

  assert render(view) =~ "High reactor temperature"
  assert render(view) =~ "183.0"
end

# Operator acknowledges alarm — panel updates immediately
test "alarm panel updates after acknowledgment" do
  alarm = create_alarm(tag: "reactor_01.temperature", state: :active)
  {:ok, view, _html} = live(conn, ~p"/dashboard/reactor_01")

  view |> element("[data-alarm-id='#{alarm.id}'] [data-action='ack']") |> render_click()

  assert render(view) =~ "acknowledged"
  refute render(view) =~ "unacknowledged"
end

# Field table — /field route shows all tags with quality
test "field table renders all tags with current quality" do
  {:ok, view, _html} = live(conn, ~p"/field")

  assert render(view) =~ "reactor_01.temperature"
  assert render(view) =~ "pump_01.running"
end

# Field table updates in real time — same as dashboard
test "field table reflects tag update without reload" do
  {:ok, view, _html} = live(conn, ~p"/field")

  TagServer.update("pump_01.running", value: true, quality: :good)

  assert render(view) =~ "TRUE"
end
```

### Convention

Tests live next to the code they test:

```
iron-core/
  src/
    deadband.rs
  tests/
    deadband_test.rs       # cargo test

iron-web/
  lib/iron_web/
    tag_server.ex
  test/iron_web/
    tag_server_test.exs    # mix test
```

---

## Level 2 — Integration Tests

Tests that require running services: NATS, TimescaleDB, iron-core, iron-web.
Run in CI via Docker Compose. Not required locally for every change.

```bash
mix test --include integration
```

What is tested:

```
iron-core publishes tag value → NATS receives it → iron-web GenServer updates
→ LiveView pushes diff → browser receives update

iron-core detects network loss → writes to SQLite buffer
→ network recovers → buffer replays in order → no data gap in TimescaleDB

Alarm triggers → audit log entry created → Telegram notification sent
→ operator acknowledges → state updates across all connected clients
```

Docker Compose for integration tests:

```yaml
# docker-compose.test.yml
services:
  nats:
    image: nats:latest
    command: "-js"

  db:
    image: timescale/timescaledb:latest-pg16
    environment:
      POSTGRES_PASSWORD: test

  iron_core:
    image: ghcr.io/getiron/iron-core:latest
    environment:
      SIMULATE: "true"
      NATS_URL: nats://nats:4222
```

### Chain Test — full signal path

The chain test traces a signal through every layer of the system:
from a simulated sensor value all the way to the browser DOM.

This is the most important integration test. If this passes,
the entire vertical slice of the system works correctly.

```
Simulated sensor value
    ↓ iron-core: deadband filter
    ↓ iron-core: engineering unit conversion
    ↓ NATS JetStream publish
    ↓ iron-web: TagServer GenServer receives update
    ↓ iron-web: Phoenix PubSub broadcasts to subscribers
    ↓ LiveView: computes DOM diff
    ↓ WebSocket: pushes diff to client
    ↓ Browser DOM: value visible
```

```elixir
@tag :integration
test "signal flows from edge to browser — full chain" do
  # Mount both views simultaneously — as a developer would in two browser windows
  {:ok, dashboard, _} = live(conn, ~p"/dashboard/reactor_01")
  {:ok, field_view, _} = live(conn, ~p"/field")

  # Simulate iron-core publishing a tag update via NATS
  # (iron-core runs in SIMULATE mode in Docker Compose)
  IronCore.Simulator.set_tag("reactor_01.temperature",
    raw: 14.4,        # 14.4mA on 4–20mA input
    quality: :good
  )

  # After deadband + conversion: 14.4mA → 130.0°C
  # Both views update simultaneously via the same PubSub broadcast

  # Window 1: SCADA dashboard
  assert render(dashboard) =~ "130.0"
  assert render(dashboard) =~ "°C"
  assert render(dashboard) =~ "GOOD"

  # Window 2: field table
  assert render(field_view) =~ "130.0"
  assert render(field_view) =~ "GOOD"
end

@tag :integration
test "chain test — quality degrades when iron-core stops publishing" do
  {:ok, dashboard, _} = live(conn, ~p"/dashboard/reactor_01")

  # Simulate sensor timeout (iron-core stops receiving data from PLC)
  IronCore.Simulator.stop_tag("reactor_01.temperature")

  # After scan_rate timeout: quality transitions GOOD → UNCERTAIN → BAD
  Process.sleep(tag_timeout_ms())

  assert render(dashboard) =~ "BAD"
  assert view |> element("[data-tag='reactor_01.temperature']") |> render() =~
           "quality-bad"
end

@tag :integration
test "chain test — alarm triggers and appears in browser" do
  {:ok, dashboard, _} = live(conn, ~p"/dashboard/reactor_01")

  # Push value above HIGH alarm limit (180°C)
  IronCore.Simulator.set_tag("reactor_01.temperature",
    raw: 19.5,        # 19.5mA → 187.5°C
    quality: :good
  )

  assert render(dashboard) =~ "High reactor temperature"
  assert render(dashboard) =~ "187.5"
end
```

The chain test runs against a real Docker Compose stack — real NATS, real TimescaleDB,
real iron-core in simulate mode. It is the closest thing to production without hardware.

---

## Level 3 — Simulation

The most important level for developer experience.
A developer should never need a PLC to build and test automation logic.

```bash
iron test --sim programs/reactor_control.st
```

### How it works

IEC 61131-3 Structured Text programs compile to WASM and run in a sandbox
with simulated I/O. The simulation is deterministic — the same scenario
produces the same result every time.

```
reactor_control.st
      │
      ▼ plc-lang/rusty (IEC 61131-3 → LLVM IR)
      │
      ▼ WASM target
      │
      ▼ iron sandbox
        ├── simulated AI/AO/DI/DO
        ├── time acceleration (60s scenario in 2s real time)
        └── scenario file (YAML)
```

### Simulation scenarios

```yaml
# test/sim/scenarios/reactor_startup.yaml
name: reactor_startup
description: Normal startup sequence — temperature rises to setpoint

timeline:
  - t: 0s
    set: { reactor_01.temp_raw: 4.0 }      # 4mA = 20°C (cold)

  - t: 10s
    set: { reactor_01.heater_do: true }     # operator starts heater

  - t: 30s
    set: { reactor_01.temp_raw: 14.4 }     # 14.4mA = 130°C (rising)

  - t: 60s
    set: { reactor_01.temp_raw: 18.0 }     # 18mA = 175°C (near setpoint)

assertions:
  - at: 60s
    expect: { reactor_01.temperature: 175.0, quality: GOOD }
    expect: { reactor_01.alarm_high: false }   # below 180°C limit
```

```yaml
# test/sim/scenarios/high_temp_alarm.yaml
name: high_temp_alarm
description: Temperature exceeds limit — alarm triggers and latches

timeline:
  - t: 0s
    set: { reactor_01.temp_raw: 19.2 }     # 19.2mA = 190°C

assertions:
  - at: 1s
    expect: { reactor_01.alarm_high: true }
    expect: { reactor_01.alarm_high_high: false }

  - at: 2s
    set: { reactor_01.temp_raw: 20.0 }     # 20mA = 200°C → HIGH HIGH

  - at: 3s
    expect: { reactor_01.alarm_high_high: true }
    expect: { reactor_01.emergency_shutdown: true }
```

```yaml
# test/sim/scenarios/network_loss.yaml
name: network_loss
description: NATS connection drops — edge agent buffers, no data loss

timeline:
  - t: 0s
    set: { reactor_01.temp_raw: 16.0 }     # normal operation

  - t: 5s
    fault: nats_disconnect

  - t: 5s–25s
    set: { reactor_01.temp_raw: 17.0 }     # data generated during outage

  - t: 25s
    fault: nats_reconnect

assertions:
  - after: nats_reconnect
    expect_historian_gap: false            # no missing rows in TimescaleDB
    expect_buffer_cleared: true
```

### Running scenarios

```bash
# Run all simulation scenarios
iron test --sim

# Run a specific scenario
iron test --sim scenarios/high_temp_alarm.yaml

# Run all scenarios for a specific object
iron test --sim --object reactor_01

# Output
# ✅ reactor_startup         (1.8s)
# ✅ high_temp_alarm         (0.4s)
# ✅ network_loss            (2.1s)
# ❌ sensor_failure          — reactor_01.quality expected BAD, got UNCERTAIN
#    at t=8s: quality transition did not complete within timeout
```

### Convention

```
test/
  sim/
    scenarios/
      reactor_startup.yaml
      high_temp_alarm.yaml
      network_loss.yaml
      sensor_failure.yaml
    programs/
      reactor_control.st     # the ST program under test
```

---

## Level 4 — Field Verification (`iron field`)

The bridge between simulation and production.

When hardware is connected, every tag must be verified: the physical signal
reaches the system correctly, engineering unit conversion is correct,
wiring matches configuration.

Traditionally done with Excel spreadsheets and paper checklists.
In IRON, it is a first-class browser workflow with results stored in Git.

### The two-window workflow

A developer connects a laptop to the local server (or directly to the edge device
running iron-core in field mode). The laptop runs `iron-web` locally or connects
to the plant's iron-web instance on the IT network.

```
Developer opens two browser windows on the same laptop:

  Window 1: http://192.168.1.100/dashboard/reactor_01
  ┌─────────────────────────────────────────────────┐
  │  SCADA — Reactor 01                             │
  │                                                 │
  │  [Mnemonic diagram with live tag values]        │
  │  Temperature: 87.5°C  ● GOOD                   │
  │  Pump 01:     RUNNING ● GOOD                   │
  │  Valve 01:    42%     ● GOOD                   │
  │                                                 │
  │  "Does the signal make sense in context?"       │
  └─────────────────────────────────────────────────┘

  Window 2: http://192.168.1.100/field
  ┌─────────────────────────────────────────────────┐
  │  Field Verification        [AI][DI][AO][DO][All]│
  ├───────────────────┬──────┬────────┬──────┬──────┤
  │ Tag               │ Type │ Value  │ Qual │ Done │
  ├───────────────────┼──────┼────────┼──────┼──────┤
  │ reactor_01.temp   │  AI  │ 87.5°C │  ✅  │  ✅  │
  │ pump_01.running   │  DI  │ FALSE  │  ✅  │  ⬜  │ ← changes live
  │ pump_02.running   │  DI  │ FALSE  │  ⚠️  │  ❌  │
  │ valve_01.position │  AI  │ 4.1mA  │  ✅  │  ⬜  │
  └───────────────────┴──────┴────────┴──────┴──────┘
  47 tags total  ·  12 verified  ·  2 failed  ·  33 pending

  "Have I verified every single tag?"
  └─────────────────────────────────────────────────┘
```

Both windows show the same live data via LiveView — the same PubSub broadcast
updates both simultaneously. The developer does not switch between applications.
They watch both views at once.

### The field session

```bash
# Edge device serves the field UI automatically when iron-core is running
# Developer connects laptop, opens browser — no command needed
http://192.168.10.5/field

# Or start explicitly on a different port
iron field --serve --port 4001
```

The field technician (наладчик) is at the sensor with a radio.
The developer is at the cabinet with a laptop.

```
Technician (radio):  "Closing DI-003 now."
Developer (browser): Watches pump_01.running: FALSE → TRUE  ✅
                     Clicks ✅ in /field table
                     Glances at /dashboard — pump shows RUNNING on mimic  ✅
                     Radio: "Good, next."
```

The SCADA view confirms the signal makes sense in process context.
The field table confirms every tag has been verified systematically.
Neither check alone is sufficient. Together they are complete.

```bash
iron field --target edge-01
```

### Signal types and verification protocol

**AI — Analog Input** (e.g. 4-20mA temperature sensor)

```
iron field prompts the engineer:

  TAG: reactor_01.temperature
  Source: modbus://plc-01/holding/0x1000
  Range: 4mA = 0°C, 20mA = 200°C

  Step 1: Apply 4mA to AI-001.  Press Enter when ready.
  → Reading: 0.1°C   [limit ±1°C]   ✅

  Step 2: Apply 20mA to AI-001. Press Enter when ready.
  → Reading: 199.8°C [limit ±1°C]   ✅

  Step 3: Apply 12mA (midpoint). Press Enter when ready.
  → Reading: 100.2°C [limit ±1°C]   ✅
```

**DI — Digital Input** (e.g. pump running feedback, limit switch)

```
  TAG: pump_01.running
  Source: modbus://plc-01/coil/0x0010

  Step 1: Ensure DI-003 is OPEN.  Press Enter when ready.
  → Reading: FALSE   ✅

  Step 2: Close DI-003.          Press Enter when ready.
  → Reading: TRUE    ✅
```

**AO — Analog Output** (e.g. control valve 4-20mA)

```
  TAG: reactor_01.valve_position
  Target: modbus://plc-01/holding/0x2000

  Step 1: IRON will set output to 0% (4mA). Confirm at field device.
  → Engineer confirms: ✅

  Step 2: IRON will set output to 50% (12mA). Confirm at field device.
  → Engineer confirms: ✅

  Step 3: IRON will set output to 100% (20mA). Confirm at field device.
  → Engineer confirms: ✅
```

**DO — Digital Output** (e.g. motor start command)

```
  TAG: pump_01.start_cmd
  ⚠️  WRITE OPERATION — requires role: operator

  Step 1: IRON will energize DO-005 for 2 seconds.
          Confirm pump_01 starts at field.
          Authorized by: aigerim@plant.kz
  → Engineer confirms: ✅
```

### Selective verification

```bash
# Verify all tags
iron field --target edge-01

# Verify a specific tag
iron field --tag reactor_01.temperature

# Verify all analog inputs
iron field --type AI

# Verify all tags for one object
iron field --object reactor_01

# Dry run — shows what would be tested, no writes
iron field --dry-run
```

### Commissioning report

Every `iron field` run generates a report:

```markdown
# Field Verification Report
Date: 2026-03-24 14:32
Target: edge-01
Engineer: Arman Seitkali
Authorized by: Aigerim Bekova (operator)

## Results

| Tag | Type | Step | Expected | Actual | Result |
|-----|------|------|----------|--------|--------|
| reactor_01.temperature | AI | 4mA  | 0.0°C   | 0.1°C  | ✅ |
| reactor_01.temperature | AI | 20mA | 200.0°C | 199.8°C| ✅ |
| pump_01.running        | DI | OPEN | FALSE   | FALSE  | ✅ |
| pump_01.running        | DI | CLOSE| TRUE    | TRUE   | ✅ |
| pump_01.start_cmd      | DO | ON   | —       | confirmed | ✅ |

## Summary
47 tags verified. 0 failures. 2 warnings.

⚠️  reactor_01.flow — not verified (source unreachable)
⚠️  pump_02.status  — not verified (skipped by engineer)
```

The report is saved as `reports/field/2026-03-24-edge-01.md` and committed to Git.
Author, timestamp, and results are permanently in version history.

### READ/WRITE enforcement

`iron field` enforces the same READ/WRITE separation as the rest of IRON:

- AI and DI verification: no writes to the PLC — safe to run at any time
- AO and DO verification: requires `operator` role minimum, full audit log entry
- Emergency stop tags: require `engineer` role and explicit `--allow-safety` flag

A bug in the verification workflow is physically incapable of sending an
unauthorized command to a machine.

---

## CI/CD Pipeline

```yaml
# .github/workflows/test.yml
on: [push, pull_request]

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - run: cargo test                    # iron-core
      - run: mix test                      # iron-web

  integration:
    runs-on: ubuntu-latest
    services:
      nats: { image: nats:latest }
      db:   { image: timescale/timescaledb:latest-pg16 }
    steps:
      - run: mix test --include integration

  simulation:
    runs-on: ubuntu-latest
    steps:
      - run: iron test --sim               # all scenarios
```

`iron field` is not in CI — it requires physical hardware.
It runs on-site during commissioning, not in a pipeline.

---

## What Comes Next (Phase 3+)

**Hardware-in-the-loop (HIL):**
A physical PLC connected to the CI environment. Every commit that touches
a protocol driver is tested against real hardware before merging.
This is expensive to set up and maintain — it belongs in Phase 3,
after the first production deployment proves the software layer is stable.

```
CI runner → iron-core → Modbus TCP → physical PLC (test bench)
                                    ← real register values
```

---

## Signal Path Coverage Map

Every layer of the signal chain is covered by at least one test level:

```
Layer                    Unit    Integration   Simulation   Field
──────────────────────────────────────────────────────────────────
Browser DOM              LV*         ✅            —           —
LiveView diff            LV*         ✅            —           —
Phoenix PubSub           ✅          ✅            —           —
TagServer GenServer      ✅          ✅            —           —
NATS broker              —           ✅            —           —
iron-core Rust logic     ✅          ✅            ✅          —
Protocol driver          ✅          ✅            —           —
Physical signal (AI/DI)  —           —             —           ✅
Physical output (AO/DO)  —           —             —           ✅

LV* = Phoenix.LiveViewTest (no real browser needed)
```

The chain test in Level 2 covers the full vertical path from iron-core to browser DOM
in a single test. `iron field` covers the physical layer that automation cannot replace.

---

## Summary

| Command | Hardware | When |
|---|---|---|
| `cargo test` / `mix test` | No | Every commit |
| `mix test` (LiveView) | No | Every commit |
| `mix test --include integration` | No (Docker) | Every commit, CI |
| Chain Test | No (Docker) | Every commit, CI |
| `iron test --sim` | No | Development, CI |
| `iron field` (browser) | Yes | On-site commissioning |
| HIL tests | Yes (test bench) | Phase 3+, protocol driver changes |

The goal: a developer with no factory access can build, test, and validate
an entire automation system. When they arrive on-site with a laptop and a patch cable,
`iron field` is the only step that requires physical hardware.
Everything before that is simulation.
