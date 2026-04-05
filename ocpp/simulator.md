# VoltWise OCPP Simulator

## Overview

The **VoltWise Simulator** is a CLI-based OCPP 1.6 Charge Point (charger) simulator.  It connects to a Central System (OCPP server) via WebSocket and behaves exactly as a real EV charger would — sending boot notifications, heartbeats, status changes, transaction records, and meter readings according to the OCPP 1.6 JSON specification.

The simulator is designed for:

- **Local development** — start chargers instantly without physical hardware.
- **Integration testing** — drive the OCPP server through complete charging session lifecycles.
- **Load simulation** — spin up hundreds of concurrent chargers to stress-test the server.
- **Demo & QA** — generate realistic traffic and inspect server-side behaviour.

---

## Use Cases

### Development

When building or modifying `voltwise-ocpp` (the OCPP Central System), the simulator provides a controllable substitute for real hardware.  Any OCPP message flow can be replicated deterministically without waiting for physical chargers.

### Testing

Each scenario maps to a well-defined test case:

| Scenario | What it validates |
|---|---|
| `basic` | Server correctly handles registration and keepalive |
| `full_charge` | Full session lifecycle: auth, energy metering, billing |

New scenarios can be added in `simulator/scenarios/` and registered in `simulator/scenarios/__init__.py` to cover additional OCPP edge cases (e.g. remote-start, firmware update, error recovery).

### Load Simulation

The `--count` flag spawns N concurrent chargers as independent asyncio tasks.  Each charger is staggered by `connect_delay × index` seconds to avoid a thundering-herd effect at the WebSocket server.

```bash
# 50 chargers, full charge scenario, 0.2 s stagger
voltwise-simulator run --count 50 --scenario full_charge --connect-delay 0.2
```

---

## Scenarios

### `basic`

The minimal registered charger scenario.

**Flow:**
1. **BootNotification** — charger identifies itself (vendor, model, serial, firmware).
2. **Heartbeat × 3** — periodic liveness pings; uses the `--delay` interval.

**When to use:** Verify connectivity, BootNotification acceptance logic, and heartbeat interval negotiation.

### `full_charge`

A complete end-to-end EV charging session.

**Flow:**
1. **BootNotification** — register charger.
2. **StatusNotification** `Available` — connector is idle and ready.
3. *(random driver arrival delay)*
4. **StatusNotification** `Preparing` — driver tapped RFID card.
5. **StartTransaction** — charging session begins; server returns a `transactionId`.
6. **StatusNotification** `Charging` — current is flowing.
7. **MeterValues** × N — periodic energy (Wh) and power (W) readings.  N and the interval are configurable via `--meter-samples` and `--meter-interval`.
8. **StatusNotification** `Finishing` — connector unlocking.
9. **StopTransaction** — session ended; final meter value sent.
10. **StatusNotification** `Available` — connector is free again.

**When to use:** End-to-end session tests, billing/metering accuracy, concurrent session load.

---

## Usage

### Installation

```bash
cd voltwise-simulator
pip install -e .
```

### Commands

```bash
# Show help
voltwise-simulator --help

# List all available scenarios
voltwise-simulator list-scenarios

# Run the basic scenario against a local OCPP server
voltwise-simulator run

# Custom server URL
voltwise-simulator run --url ws://localhost:8080/ocpp

# Full charge scenario
voltwise-simulator run --scenario full_charge

# 5 concurrent chargers
voltwise-simulator run --count 5

# 10 chargers, full charge, custom ID prefix
voltwise-simulator run --count 10 --scenario full_charge --prefix EV

# Full options example
voltwise-simulator run \
  --url ws://localhost:8080/ocpp \
  --count 5 \
  --scenario full_charge \
  --delay 0.5 \
  --meter-samples 10 \
  --meter-interval 3.0 \
  --connect-delay 0.2 \
  --prefix CHARGER
```

### All CLI Options

| Flag | Default | Description |
|---|---|---|
| `--url` | `ws://localhost:8080/ocpp` | Base WebSocket URL.  Each charger appends `/{charger-id}` |
| `--count` | `1` | Number of concurrent chargers |
| `--scenario` | `basic` | Scenario to run |
| `--delay` | `1.0` | Seconds between OCPP messages |
| `--prefix` | `SIM` | Charger ID prefix (produces `SIM-001`, `SIM-002`, …) |
| `--meter-samples` | `5` | MeterValues samples per session (full_charge only) |
| `--meter-interval` | `5.0` | Seconds between MeterValues samples |
| `--connect-delay` | `0.5` | Stagger delay between charger connections |
| `--retry-attempts` | `3` | Connection retries before giving up |

---

## Integration

### Connecting to `voltwise-ocpp`

The simulator is built specifically for use with `voltwise-ocpp`, the VoltWise OCPP Central System written in Go.

**Start the OCPP server (default port 8080):**

```bash
cd voltwise-ocpp
go run ./cmd/voltwise-ocpp
```

**Start the simulator (default `ws://localhost:8080/ocpp`):**

```bash
voltwise-simulator run --scenario full_charge
```

Each charger connects to `ws://localhost:8080/ocpp/{charger-id}`, matching the route the Go server binds on `/ocpp/{chargerID}`.

### Message flow

```
Simulator                         voltwise-ocpp
  │                                    │
  │─── WebSocket connect ─────────────►│  /ocpp/SIM-001
  │                                    │
  │─── [2,"xx","BootNotification",{}]─►│
  │◄── [3,"xx",{"status":"Accepted"}] ─│
  │                                    │
  │─── [2,"yy","StatusNotification",{}►│
  │◄── [3,"yy",{}] ───────────────────│
  │                                    │
  │─── [2,"zz","StartTransaction",{}] ►│
  │◄── [3,"zz",{"transactionId":…}] ──│
  │                                    │
  │─── [2,"aa","MeterValues",{…}] ────►│  (× N)
  │◄── [3,"aa",{}] ───────────────────│
  │                                    │
  │─── [2,"bb","StopTransaction",{…}] ►│
  │◄── [3,"bb",{}] ───────────────────│
```

### Adding new scenarios

1. Create `simulator/scenarios/my_scenario.py` with an `async def run(client, config)` function.
2. Register it in `simulator/scenarios/__init__.py`:

```python
from simulator.scenarios.my_scenario import run as _my_scenario

_REGISTRY["my_scenario"] = _my_scenario
```

3. Run it:

```bash
voltwise-simulator run --scenario my_scenario
```
