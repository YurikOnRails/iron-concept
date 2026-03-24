# RFC 0005 — Security Architecture

**Status:** Draft
**Created:** 2026-03-24

---

## Why Security Is Architectural, Not a Feature

In classical SCADA systems, security is bolt-on. Wonderware, WinCC, and MasterSCADA
were designed in the 1990s when OT networks were air-gapped. They were never built
to be secure — they were built to be isolated. That isolation is gone.
Modern plants have IT/OT convergence, remote access, cloud historians, and vendor VPNs.
The attack surface is enormous and the software was never designed for it.

IRON treats security as a first-class architectural constraint, not an afterthought.
Every design decision is evaluated against two questions:
1. Does this make it easier to read data without authorization?
2. Does this make it easier to send commands without authorization?

If the answer to either is yes, the design is wrong.

---

## The READ/WRITE Separation as Security Primitive

The most important security property in IRON is structural, not cryptographic:

```
READ path  — data flows in one direction only:
  Sensors → Edge Agent → NATS → TimescaleDB → iron-web → Browser

WRITE path — every command is explicit, authenticated, and logged:
  Operator → iron-web (Auth + RBAC) → Audit Log → Command Service
           → NATS (write topic) → Edge Agent → PLC
```

These paths share no code and share no process. A bug in the visualization layer —
a memory corruption, a logic error, an XSS — cannot reach the write path.
This is not a configuration option. It is the architecture.

**What this eliminates:**
- A compromised dashboard sending spurious commands to machines
- A developer accidentally wiring a read widget to a write endpoint
- Privilege escalation from the UI layer to the command layer

Traditional SCADA monoliths (WinCC, Wonderware) entangle these paths in a single
process. IRON makes their separation a physical property of the system.

---

## Network Segmentation

The network topology enforces the same separation:

```
┌──────────────────────────────────────────┐
│  OT Zone (VLAN 10)                       │
│  PLCs · Sensors · I/O modules            │
│  No direct internet access               │
│  No management from IT zone by default   │
└──────────────────┬───────────────────────┘
                   │ Edge Agent initiates connection to NATS (OT → IT)
                   │ Firewall rule: OT → IT allowed (agent connects out)
                   │              IT → OT blocked (no unsolicited access)
                   │
                   │ Over the established connection:
                   │   READ:  agent publishes sensor data to NATS
                   │   WRITE: agent subscribes to command topics on NATS
┌──────────────────▼───────────────────────┐
│  IT Zone (VLAN 20)                       │
│  Edge server · NATS · TimescaleDB        │
│  iron-web · Audit log                    │
└──────────────────┬───────────────────────┘
                   │ HTTPS / WebSocket (TLS)
                   │ Authenticated, authorized
┌──────────────────▼───────────────────────┐
│  Operators · Developers · Management     │
└──────────────────────────────────────────┘
```

The WRITE path crosses the firewall only through the Command Service,
which requires authentication, RBAC validation, and audit logging
before a message reaches the NATS write topic.

---

## Authentication and Authorization

### Authentication

- All human access to iron-web requires authentication (username + password minimum,
  TOTP or hardware key recommended for write-capable roles)
- API access uses short-lived JWT tokens
- Enterprise tier: SAML 2.0 / LDAP / Active Directory integration
- Session tokens are never stored in browser localStorage — HttpOnly cookies only

### Role-Based Access Control

IRON has four built-in role levels. Roles are assigned per-object, not globally:

| Role | Read data | Acknowledge alarms | Send commands | Configure tags | Manage users |
|---|---|---|---|---|---|
| **Viewer** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Operator** | ✅ | ✅ | ✅ (authorized objects only) | ❌ | ❌ |
| **Engineer** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Admin** | ✅ | ✅ | ✅ | ✅ | ✅ |

Object-level RBAC means: an operator can start pumps on Line 1 but not Line 2,
without any custom code.

### Command Confirmation

High-consequence commands require explicit confirmation:

```yaml
# commands/reactor_01.yaml
emergency_shutdown:
  requires_role: operator
  requires_confirm: true     # UI shows confirmation dialog
  requires_2fa: true         # TOTP code required at runtime
  audit: true
  cooldown: 60s              # Cannot be re-sent within 60 seconds
```

---

## Audit Log

Every WRITE action is immutable and append-only:

```
2026-03-24T14:23:11Z  arman@plant.kz  COMMAND  reactor_01.pump_start  → true
  role: operator  ip: 192.168.20.15  confirmed: true  result: ACK
  previous_value: false  duration_ms: 234

2026-03-24T14:23:45Z  arman@plant.kz  COMMAND  reactor_01.pump_start  → true
  role: operator  ip: 192.168.20.15  confirmed: true  result: NACK
  reason: "PLC returned exception code 02 (illegal data address)"
```

The audit log is written to TimescaleDB before the command is sent to the PLC.
If the write to the audit log fails, the command is not sent. No exceptions.

For enterprise tier: audit export in formats compatible with SOC 2, ISO 27001,
and IEC 62443 reporting requirements.

---

## IEC 62443 Alignment

IEC 62443 is the international standard for industrial cybersecurity.
IRON's architecture maps directly to its requirements:

| IEC 62443 Requirement | IRON Implementation |
|---|---|
| SL1 — Authentication | Username + password on all access |
| SL1 — Audit log | Immutable append-only command log |
| SL2 — Role-based access control | Object-level RBAC |
| SL2 — Use control (write separation) | READ/WRITE path separation |
| SL2 — Data integrity | TLS for all transport, PostgreSQL checksums |
| SL3 — MFA for critical operations | TOTP on write commands (configurable) |
| SL3 — Zone and conduit model | VLAN separation, firewall rules documented |
| SL3 — Software update management | Git-based config, signed releases |

IRON does not claim IEC 62443 certification out of the box — certification requires
a formal assessment process. But the architecture is designed to make that assessment
as straightforward as possible.

---

## NIS2 Compliance

The EU NIS2 Directive (effective October 2024) mandates cybersecurity measures
for operators of essential services, including industrial operators.

Key NIS2 requirements and how IRON addresses them:

**Risk management measures:**
- Network segmentation (VLAN architecture documented)
- Access control (RBAC + MFA)
- Asset management (all tags, objects, and configurations in Git)

**Incident handling:**
- Audit log provides complete incident timeline
- Alarm engine detects anomalies in real time
- Git history shows configuration state at any point in time

**Supply chain security:**
- Apache 2.0 licensed, full source available for audit
- No proprietary black-box components
- Dependency management via Cargo (Rust) and Hex (Elixir) with lock files

**Reporting:**
- Enterprise tier provides compliance report export
- Audit log export for regulatory submissions

A proprietary SCADA system that hasn't received a security update since 2019
and runs on Windows Server 2012 is not NIS2-compliant. IRON, with its
Git-based configuration management, documented security architecture, and
standard update mechanism, is demonstrably closer to compliance.

---

## Rust and Memory Safety

The edge agent — the component with direct access to PLC hardware — is written in Rust.

Google's Android team reports that memory safety vulnerabilities (buffer overflows,
use-after-free, null pointer dereferences) account for ~70% of critical CVEs
in C/C++ codebases. Rust eliminates this class of vulnerability at compile time.

In an OT context, a memory corruption in a process that talks directly to PLCs
is not just a security issue — it is a safety issue. A corrupted write to a PLC
register can command physical actuators to move.

The choice of Rust for iron-core is not a technology preference.
It is a safety requirement.

---

## What IRON Does Not Claim

**IRON is not a safety system.** It is a monitoring and control system.
Safety-critical functions (emergency stops, SIL-rated interlocks) must remain
in dedicated safety PLCs (Pilz, Siemens F-series) that operate independently
of IRON. IRON can monitor safety system state but must never be the
primary safety mechanism.

**IRON does not eliminate all attack vectors.** A compromised operator workstation,
a malicious insider with valid credentials, or a vulnerability in a dependency
can still cause harm. Defense in depth is required. IRON is one layer.

**IEC 62443 certification requires a formal assessment.** The architecture is
designed for compliance, but certification is a process, not a property of the code.
