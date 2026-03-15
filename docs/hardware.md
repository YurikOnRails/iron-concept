# Hardware Reference

## Philosophy

Traditional SCADA buys expensive proprietary hardware because "it's more reliable."
In reality, reliability comes from **redundancy architecture**, not from a price tag.

Cheap reliable hardware + correct architecture > expensive hardware without redundancy.

---

## PLC Options

| Option | Price | Best for |
|---|---|---|
| CLICK PLUS (AutomationDirect) | $200 | New projects, Linux inside, Rust agent on same hardware |
| CODESYS on Wago PFC200 | $400 | Standard CODESYS environment, proven reliability |
| Beckhoff CX5130 (TwinCAT 3) | $1,500 | Large objects, VS Code + Git workflow, C++/Rust modules |
| Soft PLC (CODESYS on x86) | $0 runtime | Maximum flexibility, commodity hardware replacement |

### CLICK PLUS — the non-obvious choice

Most integrators have never heard of it. At $200, it has Linux inside (Debian-based),
Modbus TCP/RTU native, and you can SSH into it and run a Rust Edge Agent directly.

```bash
# SSH into the PLC like a normal Linux server
ssh admin@192.168.1.10

# Run your Rust agent on the same hardware as the PLC
./edge-agent --modbus-host localhost --nats-url nats://server:4222
```

Two processes on one device: PLC logic in real-time and data collection agent.
Isolated at OS level. One fewer device in the cabinet.

---

## Edge Computing Hardware

### Beelink EQ12 — recommended for most cases

- Intel N100 (4 cores, 3.4GHz)
- 16GB DDR5 RAM
- 2× 2.5GbE ports (one for IT network, one for OT network)
- **Passive cooling — no fan, no bearings, nothing to wear out**
- 10W power consumption
- Price: ~$150

The passive cooling is not a nice-to-have. In an industrial environment,
fan bearing failure is a leading cause of edge device downtime.
At 10W, the N100 runs cool without active cooling.

Put it in a DIN-rail enclosure ($30) and you have an industrial computer for $180.

### Raspberry Pi CM4 in industrial enclosure

- ARM Cortex-A53, 4GB RAM, eMMC storage
- -40°C to +85°C operating range (industrial)
- Total cost with enclosure: ~$150
- Available from: Waveshare, Seeed Studio industrial carriers

### For demanding workloads

Toradex Verdin i.MX8 — industrial SoM with TSN (Time-Sensitive Networking)
Ethernet. TSN provides deterministic latency for time-critical applications.
~$200–400 depending on configuration.

---

## Server Infrastructure

### The key principle: two cheap > one expensive

One $15,000 server with one power supply = single point of failure.

Two $500 servers + Patroni (automatic PostgreSQL failover) + NATS cluster
= real HA at 1/30th the cost. Automatic failover in under 10 seconds.

### Hetzner dedicated server (cloud)

- AX41: AMD Ryzen 3600, 64GB RAM, 2× 512GB NVMe — €38/month
- Run two for HA: €76/month total
- German data center, excellent uptime, GDPR-compliant

### On-premise

Any server with ECC RAM and IPMI.
ECC RAM prevents silent data corruption — important for historian data.
IPMI allows remote management when the OS is unresponsive.

Supermicro X12 or Dell PowerEdge R350 are solid choices.

---

## Control Cabinet Layout

```
┌─────────────────────────────────────┐
│  ZONE 1 — POWER                     │
│  ABB breakers · 24VDC PSU · UPS     │
├─────────────────────────────────────┤
│  ZONE 2 — CONTROLLER                │
│  PLC (CLICK PLUS or equivalent)     │
│  Managed switch (VLAN IT/OT)        │
├─────────────────────────────────────┤
│  ZONE 3 — I/O MODULES               │
│  DI×8 · DO×8 · AI×4                │
├─────────────────────────────────────┤
│  ZONE 4 — FIELD TERMINALS           │
│  Phoenix Contact · labeled · grouped│
├─────────────────────────────────────┤
│  ZONE 5 — SAFETY                    │
│  Pilz E-Stop · SIL2 safety relay   │
├─────────────────────────────────────┤
│  ZONE 6 — DOOR PANEL                │
│  Start/Stop · Optional 7" tablet   │
└─────────────────────────────────────┘
         Cable entries (bottom)
```

### Rules

**Cable routing:** power cables (230VAC, motors) on the left side.
Signal cables (sensors, digital inputs) on the right. Never in the same bundle.
Mixed routing causes interference on analog signals.

**Fill ratio:** maximum 60–70% at installation. Reserve for expansion and heat dissipation.
A cramped, overheated cabinet has a 3–5 year lifespan instead of 15.

**VLAN isolation:** managed switch with VLAN 10 (OT: PLC, sensors) and
VLAN 20 (IT: edge server, uplink). Traffic between VLANs only through
explicitly defined rules. This is the physical implementation of
READ/WRITE path separation.

**UPS:** Even 5 minutes is enough for graceful shutdown. Without UPS,
a voltage spike reboots the PLC into unknown state. In industrial processes,
that can mean restarting an already-running motor.

---

## Estimated Cabinet Cost

| Component | Cost |
|---|---|
| Enclosure (Rittal or equivalent) | $300–500 |
| ABB breakers + 24VDC PSU | $150 |
| Mini UPS | $200 |
| CLICK PLUS PLC | $200 |
| Managed DIN-rail switch | $150 |
| I/O modules (DI/DO/AI) | $200 |
| Phoenix Contact terminals | $100 |
| Pilz E-Stop relay | $150 |
| Beelink EQ12 (edge server) | $150 |
| Cables, labels, hardware | $200 |
| **Total** | **~$1,800** |

Equivalent functionality with Siemens S7-1200 + Weintek HMI + industrial PC: $6,000–10,000.

---

## Sensors

### IO-Link — the non-obvious standard

IO-Link (IEC 61131-9) turns any sensor into a smart device with bidirectional communication.
Instead of a 4-20mA analog signal (just a number), you get:

```json
{
  "value": 23.4,
  "unit": "°C",
  "status": "GOOD",
  "serial_number": "IFM-12345",
  "calibration_valid_until": "2026-06",
  "diagnostics": "OK"
}
```

**Recommended vendors:**
- IFM Electronic (German, excellent quality, good documentation)
- Turck (German, strong RFID line for pallet/tooling identification)

IO-Link masters from IFM (AL1350) output Modbus TCP or OPC-UA directly.
No custom protocol parsing in the Edge Agent.

### Vibration monitoring on a budget

Industrial vibration sensors (SKF, Brüel & Kjær): $800–1,500 each.

Alternative: MEMS accelerometer (ADXL355, $15) on a custom board with ESP32-C3
and a Rust firmware. Provides full vibration spectrum in real-time.

50–100x cheaper, same data. The ESP32-C3 is RISC-V — Rust compiles to it natively
via `esp-hal`.

This is a validated approach for predictive maintenance on motors and pumps in
non-critical monitoring applications.
