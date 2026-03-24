# Test-Driven Development in IRON

TDD is not a methodology imposed on IRON from the outside.
It is the natural consequence of choosing Rust and Elixir —
two languages where testing is a first-class citizen, built into the toolchain,
not bolted on as an afterthought.

```bash
cargo test   # iron-core — Rust, built-in, zero configuration
mix test     # iron-web  — Elixir, built-in, zero configuration
```

No test framework to install. No configuration files.
The right thing is the easy thing.

---

## The TDD Cycle for Industrial Automation

The cycle is the same as everywhere else. What changes is what you are testing.

```
1. RED    Write a test that describes the desired behavior.
          It fails because the behavior does not exist yet.

2. GREEN  Write the minimum code to make the test pass.
          Not beautiful code. Working code.

3. REFACTOR  Clean up. The test protects you.
             If it still passes, you did not break anything.
```

In industrial automation, "desired behavior" means things like:

- A temperature sensor reading 14.4mA should produce 130.0°C
- A deadband of 0.5°C should suppress a change from 87.3 to 87.4
- A HIGH alarm should trigger at exactly 180°C, not 179.9
- A tag that stops publishing for 3 scan cycles should become UNCERTAIN

These are precise, testable, and consequential.
A wrong deadband wastes bandwidth. A wrong alarm limit misses a real event.
TDD forces you to define "correct" before you write the code that produces it.

---

## TDD in iron-core (Rust)

Rust's test system is built into the language. Tests live in the same file
as the code they test, in a `#[cfg(test)]` module. No separate test runner,
no test framework, no dependencies.

### The cycle in practice

**Step 1 — RED: write the test first**

```rust
// src/deadband.rs

pub fn should_publish(last: f32, current: f32, deadband: f32) -> bool {
    todo!() // does not exist yet
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn suppresses_change_within_deadband() {
        // 87.3 → 87.4 with deadband 0.5 — should NOT publish
        assert!(!should_publish(87.3, 87.4, 0.5));
    }

    #[test]
    fn publishes_change_outside_deadband() {
        // 87.3 → 87.9 with deadband 0.5 — should publish
        assert!(should_publish(87.3, 87.9, 0.5));
    }

    #[test]
    fn publishes_on_exact_deadband_boundary() {
        // 87.3 → 87.8 = exactly 0.5 — boundary: should publish
        assert!(should_publish(87.3, 87.8, 0.5));
    }

    #[test]
    fn always_publishes_quality_change() {
        // value did not change but quality degraded — always publish
        assert!(should_publish_with_quality(
            (87.3, Quality::Good),
            (87.3, Quality::Uncertain),
            deadband: 0.5
        ));
    }
}
```

```bash
cargo test
# FAILED: should_publish — not yet implemented (todo!())
```

**Step 2 — GREEN: minimum code to pass**

```rust
pub fn should_publish(last: f32, current: f32, deadband: f32) -> bool {
    (current - last).abs() >= deadband
}
```

```bash
cargo test
# ok: suppresses_change_within_deadband
# ok: publishes_change_outside_deadband
# ok: publishes_on_exact_deadband_boundary
```

**Step 3 — REFACTOR: the test protects you**

```rust
// Extract to a struct — cleaner API, same test still passes
pub struct DeadbandFilter {
    last_value: f32,
    last_quality: Quality,
    deadband: f32,
}

impl DeadbandFilter {
    pub fn should_publish(&mut self, value: f32, quality: Quality) -> bool {
        let value_changed = (value - self.last_value).abs() >= self.deadband;
        let quality_changed = quality != self.last_quality;
        value_changed || quality_changed
    }
}
```

```bash
cargo test
# All tests still pass. Refactor was safe.
```

### TDD for protocol drivers

Protocol drivers are where TDD has the highest return: testing against
a real PLC is slow, expensive, and not always possible.
TDD with a mock gives you fast, repeatable, hardware-free tests.

```rust
// Mock Modbus client — same trait as the real one
struct MockModbusClient {
    registers: HashMap<u16, u16>,
}

impl ModbusClient for MockModbusClient {
    fn read_holding_register(&self, address: u16) -> Result<u16, ModbusError> {
        self.registers.get(&address)
            .copied()
            .ok_or(ModbusError::AddressNotFound)
    }
}

#[test]
fn reads_float32_from_two_consecutive_registers() {
    let mut client = MockModbusClient::new();
    // 180.0°C as IEEE 754 float32 split across two 16-bit registers
    client.set_register(0x1000, 0x4334); // high word
    client.set_register(0x1001, 0x0000); // low word

    let value = read_float32(&client, 0x1000).unwrap();
    assert!((value - 180.0).abs() < 0.001);
}

#[test]
fn returns_error_on_connection_timeout() {
    let client = TimeoutModbusClient::new(timeout: Duration::from_ms(100));

    let result = read_holding_register(&client, 0x1000);
    assert!(matches!(result, Err(ModbusError::Timeout)));
}
```

### Engineering unit conversion — TDD pays for itself

Unit conversion errors have caused real industrial accidents.
A test that documents the expected behavior at 4mA, 20mA, and midpoint
is not bureaucracy — it is a permanent record of what "correct" means.

```rust
#[test]
fn converts_4ma_to_range_minimum() {
    // 4mA = 0% = 0.0°C on a 0–200°C sensor
    assert_eq!(convert_ai(4.0, 4.0, 20.0, 0.0, 200.0), 0.0);
}

#[test]
fn converts_20ma_to_range_maximum() {
    assert_eq!(convert_ai(20.0, 4.0, 20.0, 0.0, 200.0), 200.0);
}

#[test]
fn converts_midpoint_correctly() {
    // 12mA = 50% = 100.0°C
    assert_eq!(convert_ai(12.0, 4.0, 20.0, 0.0, 200.0), 100.0);
}

#[test]
fn clamps_below_range_minimum() {
    // 3.5mA is below 4mA — sensor fault or broken wire
    // Should clamp to minimum, not produce a negative value
    let value = convert_ai(3.5, 4.0, 20.0, 0.0, 200.0);
    assert_eq!(value, 0.0);
    // Quality should be UNCERTAIN for out-of-range raw input
}
```

---

## TDD in iron-web (Elixir)

ExUnit is Elixir's built-in test framework. It is designed for concurrent testing,
has excellent async support, and integrates directly with Phoenix.

### The cycle in Elixir

```elixir
# test/iron_web/tag_server_test.exs

defmodule IronWeb.TagServerTest do
  use ExUnit.Case, async: true

  # Step 1 — RED: describe the behavior before implementing it
  describe "value updates" do
    test "stores the latest value and quality" do
      {:ok, pid} = TagServer.start_link(tag: "reactor_01.temperature")

      TagServer.update(pid, value: 87.5, quality: :good)

      state = TagServer.get_state(pid)
      assert state.value == 87.5
      assert state.quality == :good
    end

    test "broadcasts update to subscribers via PubSub" do
      {:ok, pid} = TagServer.start_link(tag: "reactor_01.temperature")
      Phoenix.PubSub.subscribe(Iron.PubSub, "tag:reactor_01.temperature")

      TagServer.update(pid, value: 95.0, quality: :good)

      assert_receive {:tag_update, %{value: 95.0, quality: :good}}
    end

    test "does not broadcast if value is within deadband" do
      {:ok, pid} = TagServer.start_link(tag: "reactor_01.temperature",
                                         deadband: 0.5)
      TagServer.update(pid, value: 87.3, quality: :good)
      Phoenix.PubSub.subscribe(Iron.PubSub, "tag:reactor_01.temperature")

      # Change within deadband — should not broadcast
      TagServer.update(pid, value: 87.4, quality: :good)

      refute_receive {:tag_update, _}, 100
    end
  end

  describe "quality transitions" do
    test "transitions to UNCERTAIN after missed scan cycles" do
      {:ok, pid} = TagServer.start_link(
        tag: "reactor_01.temperature",
        scan_rate: 100,   # ms, fast for testing
        timeout: 300      # ms — UNCERTAIN after 3 missed cycles
      )
      TagServer.update(pid, value: 87.5, quality: :good)

      # Simulate timeout — no updates for 3 scan cycles
      Process.sleep(350)

      assert TagServer.get_state(pid).quality == :uncertain
    end

    test "always broadcasts on quality change even within deadband" do
      {:ok, pid} = TagServer.start_link(tag: "reactor_01.temperature",
                                         deadband: 0.5)
      TagServer.update(pid, value: 87.3, quality: :good)
      Phoenix.PubSub.subscribe(Iron.PubSub, "tag:reactor_01.temperature")

      # Same value, quality degrades — must broadcast
      TagServer.update(pid, value: 87.3, quality: :uncertain)

      assert_receive {:tag_update, %{quality: :uncertain}}
    end
  end
end
```

### TDD for LiveView

Phoenix.LiveViewTest allows TDD at the browser layer without a real browser.
Write the test for the rendered DOM, then implement the LiveView to pass it.

```elixir
defmodule IronWebWeb.DashboardLiveTest do
  use IronWebWeb.ConnCase

  import Phoenix.LiveViewTest

  # RED: describe what the browser should show
  test "displays tag value and quality from TagServer" do
    # TagServer not yet connected to LiveView — this test will fail
    {:ok, view, _html} = live(conn, ~p"/dashboard/reactor_01")

    TagServer.update("reactor_01.temperature", value: 87.5, quality: :good)

    assert render(view) =~ "87.5"
    assert render(view) =~ "°C"
    assert render(view) =~ "GOOD"
  end

  test "shows alarm badge when tag is in alarm state" do
    {:ok, view, _html} = live(conn, ~p"/dashboard/reactor_01")

    Alarms.trigger("reactor_01.temperature", :high, value: 183.0)

    assert view |> element("[data-alarm-badge]") |> render() =~ "HIGH"
  end
end
```

### TDD for RBAC

Security behavior must be tested explicitly.
"It probably works" is not acceptable when the question is
"can an operator send an emergency shutdown command?"

```elixir
describe "write permissions" do
  test "viewer role cannot send any command" do
    conn = log_in_user(conn, viewer_user())
    assert {:error, :unauthorized} =
      Commands.send(conn.assigns.current_user,
        tag: "pump_01.start_cmd", value: true)
  end

  test "operator can send normal commands" do
    conn = log_in_user(conn, operator_user())
    assert :ok =
      Commands.send(conn.assigns.current_user,
        tag: "pump_01.start_cmd", value: true)
  end

  test "operator cannot send emergency shutdown" do
    conn = log_in_user(conn, operator_user())
    assert {:error, :insufficient_role} =
      Commands.send(conn.assigns.current_user,
        tag: "reactor_01.emergency_shutdown", value: true)
  end

  test "engineer can send emergency shutdown" do
    conn = log_in_user(conn, engineer_user())
    assert :ok =
      Commands.send(conn.assigns.current_user,
        tag: "reactor_01.emergency_shutdown", value: true)
  end
end
```

---

## TDD for Protocol Drivers

Protocol drivers are tested against software simulators — no physical hardware needed.

| Protocol | Simulator | Notes |
|---|---|---|
| Modbus TCP | diagslave, ModRSsim2 | Free, runs on Linux/Windows |
| Modbus RTU | socat virtual serial port | Creates virtual RS-485 pair |
| OPC-UA | open62541 test server | Reference implementation |
| S7 | snap7 test server | Siemens S7 compatible |

```rust
// Integration test with a real Modbus simulator
// Requires: diagslave running on localhost:502
#[tokio::test]
#[ignore = "requires diagslave simulator"]
async fn reads_holding_register_from_real_modbus_server() {
    let client = ModbusClient::connect("127.0.0.1:502").await.unwrap();
    let value = client.read_holding_register(0x0000).await.unwrap();
    // diagslave returns 0 for unwritten registers
    assert_eq!(value, 0);
}
```

---

## What TDD Changes in Industrial Automation

Without TDD, the feedback loop is:

```
Write code → deploy to edge device → connect to PLC →
check value on screen → realize conversion is wrong →
fix code → redeploy → reconnect → check again
```

20–40 minutes per cycle. Physical hardware required.

With TDD, the feedback loop is:

```
Write test → cargo test → see failure → fix code → cargo test → pass
```

Under 10 seconds. No hardware. No deployment.

This is not a convenience. On a factory floor where PLC time is shared
with production, a developer who needs the PLC for every test change
is a developer who blocks production. TDD removes that dependency entirely.

---

## Convention

```
iron-core/
  src/
    deadband.rs          # production code
    deadband/
      mod.rs
    #[cfg(test)] inline  # unit tests in the same file

iron-web/
  lib/iron_web/
    tag_server.ex
  test/iron_web/
    tag_server_test.exs  # mirrors lib/ structure exactly

test/
  integration/           # mix test --include integration
  sim/scenarios/         # iron test --sim
```

One command to run all unit tests:

```bash
cargo test && mix test
```

If both pass, the system logic is correct. Integration and simulation
tests run in CI against the full stack.
