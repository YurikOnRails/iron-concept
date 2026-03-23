# RFC 0002 — Architecture

**Status:** Draft
**Created:** 2026-03-24

---

## Overview

IRON has a two-layer architecture with a clear boundary between runtime and interface:

```
┌─────────────────────────────────────────────────────────────────┐
│  FIELD LEVEL                                                     │
│  PLCs: Siemens S7 · Allen-Bradley · CLICK PLUS · Soft PLC       │
│  Sensors: IO-Link · 4-20mA · HART · Vibration                   │
│  Protocols: Modbus TCP/RTU · OPC-UA · S7 · MQTT                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│  iron-core  (Rust)                                               │
│                                                                  │
│  Edge Agent                                                      │
│  • Protocol drivers: Modbus, OPC-UA, S7, MQTT                   │
│  • Deadband filtering (−80–90% traffic before leaving the floor) │
│  • Data quality: GOOD / UNCERTAIN / BAD                         │
│  • Local SQLite buffer — survives network loss                   │
│  • WASM modules — custom logic without recompiling the agent     │
│                                                                  │
│  Tag Engine · Alarm Engine · Historian (TimescaleDB)             │
└───────────────────────────┬─────────────────────────────────────┘
                            │ NATS JetStream
                            │ topic: plant.line1.reactor1.temperature
┌───────────────────────────▼─────────────────────────────────────┐
│  iron-web  (Phoenix / LiveView)                                  │
│                                                                  │
│  • Real-time dashboards — LiveView, no JavaScript required       │
│  • SVG mimic editor — React island (escape hatch for canvas UI)  │
│  • REST API · GraphQL · WebSocket                                │
│  • User management · RBAC · Audit log                            │
│  • Integrator tools · Configuration interface                    │
└───────────────────────────┬─────────────────────────────────────┘
                            │ WebSocket
                        Browser / Mobile
```

---

## iron-core (Rust)

The runtime layer. Runs on the factory floor. Responsible for everything between
the wire and the message broker.

### Edge Agent

The agent runs physically close to the PLC — on the same DIN-rail hardware, on a
Raspberry Pi in the cabinet, or on an x86 mini-PC. It is not a relay. It is intelligent:

**Deadband filtering:** a temperature sitting at 87.3°C ± 0.05°C does not generate
a single NATS message at deadband = 0.5°C. 80–90% of tag polls produce no network traffic.

```rust
fn should_publish(last: f32, current: f32, deadband: f32) -> bool {
    (current - last).abs() > deadband
}
```

**Data quality marking:** every tag value carries `GOOD / UNCERTAIN / BAD`.
Downstream consumers don't have to guess whether the sensor is alive.

**Local buffer:** when the network goes down, the agent writes to SQLite.
When the network comes back, it replays in order. No data loss during network incidents.

**Autonomous operation:** the factory floor keeps running when the server is unreachable.
The agent keeps polling, keeps buffering, keeps enforcing limits locally.

**WASM modules:** custom logic (unit conversions, derived tags, custom protocols) can be
deployed as WASM modules without recompiling the agent binary.

### Tag Engine

Tags are the core abstraction. A tag maps a physical signal to a named value in the system:

```yaml
reactor_01:
  temperature:
    source: modbus://plc-01/holding/0x1000
    type: float32
    unit: "°C"
    scan_rate: 1s
    deadband: 0.5
    range: [0, 200]
    alarms:
      high:
        limit: 180
        priority: 2
        message: "High reactor temperature"
      high_high:
        limit: 195
        priority: 1
        message: "Critical — emergency shutdown"
        action: emergency_shutdown
```

Commands are a separate file, a separate route, separate permissions:

```yaml
# commands/reactor_01.yaml — physically separate from tags/
reactor_01:
  pump_start:
    target: modbus://plc-01/coil/0x0020
    type: bool
    requires_confirm: true
    requires_role: operator
    audit: true
```

This is the READ/WRITE separation made concrete in the filesystem.

### Historian

TimescaleDB (PostgreSQL extension). One database for configuration, time-series,
alarms, users, and permissions.

```
Traditional approach:          IRON:
  PostgreSQL  (config)    →      TimescaleDB
  InfluxDB    (time-series) →    (one database)
  Redis       (current)   →      + ETS cache in Elixir

3 failure points               1 failure point
3 synchronizations             0 synchronizations
```

A plant with 10,000 tags updating every second generates 315 billion rows/year.
TimescaleDB handles this with 8–12x compression, continuous aggregates, and
chunk-based retention. A 30-day trend query returns in 3–8ms.

---

## iron-web (Phoenix / LiveView)

The interface layer. Runs on the server (cloud or on-premise). Responsible for
everything between the message broker and the human.

### Why LiveView for 90% of the UI

Every SCADA UI problem is a real-time state synchronization problem:
- A tag updates → the operator's screen must update
- The operator sends a command → every connected client must reflect the new state
- A user disconnects and reconnects → they must get current state immediately

LiveView solves all of these at the framework level. WebSocket lifecycle, reconnection,
diff-based DOM updates, state management — handled. The application developer writes
Elixir, not JavaScript event loops.

```
100,000 tag polls/sec
  → deadband filter (Rust)      → ~10,000 publishes/sec  (−90%)
  → GenServer routing (Elixir)  → subscribers only
  → LiveView diff               → ~50–200 updates/sec per browser screen
```

An operator viewing a pump station sees updates for the 50–200 tags on that screen.
Not 100,000. This is the difference between correct and naive architecture.

### React Island: SVG Mimic Editor

The one place where LiveView is not the right tool is an interactive SVG canvas editor —
drag-and-drop placement of equipment symbols, wiring, animation binding.
This is a DOM-heavy, stateful, single-user editing experience.

IRON uses a React component for this, mounted as a LiveView hook. LiveView manages
the component's lifecycle (mount, update, destroy). The component communicates back
to LiveView via `pushEvent`. The operator's display of the mimic at runtime is
pure LiveView — React is only present in the editor.

This is not a compromise. This is the correct separation of concerns.

---

## Communication Layer

NATS JetStream is the internal message bus between iron-core and iron-web.

**Why NATS over Kafka, RabbitMQ, Redis Streams:**
- Single binary, zero dependencies, 30MB RAM
- 10M+ messages/sec on commodity hardware
- At-least-once delivery with JetStream
- Subject hierarchy mirrors physical topology: `plant.line_1.reactor_01.temperature`
- Wildcard subscriptions: `plant.line_1.>` gives you all of line 1
- Replay on reconnect — a new iron-web instance can catch up immediately

NATS subjects enforce the Unified Namespace concept: every tag has exactly one
canonical address in the system.

---

## Protocol Support Roadmap

| Protocol | Status | Notes |
|---|---|---|
| Modbus TCP | v0.1 | Most common protocol in the field |
| Modbus RTU | v0.1 | Serial, RS-485, very common in older equipment |
| OPC-UA | v0.2 | Modern standard, required for Siemens/Beckhoff |
| S7 (Siemens) | v0.3 | Huge installed base in CIS, no OPC-UA on older S7-300/400 |
| MQTT | v0.3 | IoT sensors, edge-to-cloud bridging |
| BACnet | v0.5 | Building automation, HVAC |
| DNP3 | v0.6 | Utilities, water treatment |
| EtherNet/IP | v0.6 | Allen-Bradley, large North American installed base |
| IO-Link | v0.4 | Smart sensors, bidirectional diagnostics |
| PROFINET | v1.0 | Required for large Siemens installations |

---

## Edge Deployment Model

iron-core is designed to run on cheap, reliable hardware:

| Hardware | Cost | Use case |
|---|---|---|
| Raspberry Pi CM4 (industrial carrier) | ~$150 | Small plants, greenhouses, up to ~2,000 tags |
| Beelink EQ12 (Intel N100, passive cooling) | ~$150 | Mid-size plants, DIN-rail mounting |
| CLICK PLUS PLC | ~$200 | PLC + edge agent on same hardware (Linux inside) |
| x86 industrial PC | $300–800 | Large plants, demanding workloads |

The passive cooling on the Beelink EQ12 is not a nice-to-have. Fan bearing failure
is a leading cause of edge device downtime in industrial environments.

### Network Topology

```
VLAN 10 (OT): PLCs, sensors, I/O modules — isolated
VLAN 20 (IT): Edge server, uplink to NATS
Firewall rule: OT → IT allowed (edge agent publishes)
               IT → OT blocked by default (no unsolicited access to PLCs)
```

This is the READ path made concrete in network topology.

---

## Scaling

A mid-size refinery has 100,000 tags updating every second. Three filtering layers
make this tractable:

```
Layer 1 — Deadband (Rust edge agent)
  100,000 polls/sec → ~10,000 publishes/sec  (−90%)

Layer 2 — GenServer routing (Elixir)
  Each tag = one lightweight Elixir process (~2KB RAM)
  100,000 tags = ~200MB RAM
  Updates go only to subscribers of that tag, not broadcast

Layer 3 — Client subscription (LiveView)
  Operator opens pump station screen
  → subscribes to 50–200 visible tags only
  → browser receives 50–200 updates/sec, not 10,000
```

Benchmarks on commodity hardware:

```
NATS JetStream:         10–15M messages/sec
Phoenix WebSocket:      2M concurrent connections
Tag → browser latency:  5–15ms
TimescaleDB insert:     ~1M rows/sec (batched)
30-day trend query:     3–8ms (continuous aggregates)
```
