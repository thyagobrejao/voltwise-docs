# VoltWise OCPP — Architecture

## Overview

**OCPP** (Open Charge Point Protocol) is the industry-standard communication protocol between EV charging stations and a central management system. VoltWise uses **OCPP 1.6 JSON** — the most widely deployed version — which runs over WebSocket with a simple JSON-array framing.

**voltwise-ocpp** is the Go service that acts as the OCPP Central System. It:

- accepts WebSocket connections from chargers
- parses and validates OCPP messages
- dispatches them to action-specific handlers
- calls into the VoltWise core domain for business logic
- returns conformant OCPP responses

---

## Supported Messages

| Action | Direction | Purpose |
|---|---|---|
| **BootNotification** | Charger → Server | Charger registers itself on boot |
| **Heartbeat** | Charger → Server | Periodic liveness signal |
| **StatusNotification** | Charger → Server | Reports connector status changes |
| **StartTransaction** | Charger → Server | Begins a charging session |
| **StopTransaction** | Charger → Server | Ends a charging session |
| **MeterValues** | Charger → Server | Reports energy/power readings |

---

## Architecture

```
voltwise-ocpp/
├── cmd/voltwise-ocpp/        # Application entry point
│   └── main.go
└── internal/
    ├── ocpp/                  # Protocol types, parser, serialiser
    │   ├── message.go
    │   └── actions.go
    ├── handlers/              # Per-action message handlers + router
    │   ├── router.go
    │   ├── boot_notification.go
    │   ├── heartbeat.go
    │   ├── status_notification.go
    │   ├── start_transaction.go
    │   ├── stop_transaction.go
    │   └── meter_values.go
    ├── session/               # Connection & session manager
    │   └── manager.go
    ├── server/                # WebSocket server & HTTP wiring
    │   └── server.go
    └── ports/                 # Interfaces to external systems
        └── core.go
```

### Components

#### WebSocket Server (`internal/server`)

Listens on `/ocpp/{chargerID}`, upgrades to WebSocket with the `ocpp1.6` subprotocol, registers the connection, and enters a read loop. Handles disconnects and concurrent chargers safely.

#### Message Parser (`internal/ocpp`)

Implements the OCPP 1.6J wire format:

- **Call** (type 2): `[2, UniqueID, Action, Payload]` — charger → server
- **CallResult** (type 3): `[3, UniqueID, Payload]` — server → charger
- **CallError** (type 4): `[4, UniqueID, ErrorCode, Description, Details]` — server → charger

#### Router (`internal/handlers`)

Maps action strings (e.g. `"BootNotification"`) to handler implementations. Unknown actions receive a `NotImplemented` error. Handler failures are caught and returned as `InternalError`.

#### Handlers (`internal/handlers`)

Each handler implements a single `Handle(ctx, conn, call) → (payload, error)` interface. Handlers are stateless; they receive the connection metadata and the parsed Call, perform any domain interaction through the `CoreService` port, and return the response payload.

#### Session Manager (`internal/session`)

Thread-safe map of `chargerID → Conn`. Tracks the WebSocket connection, last heartbeat time, and current OCPP transaction ID. Supports last-writer-wins reconnection (stale connection is closed when the same charger reconnects).

#### Ports (`internal/ports`)

Defines `CoreService` — the interface through which handlers call VoltWise core domain logic — plus optional `EventPublisher` and `Authenticator` interfaces for future use.

---

## Message Flow

```
Charger                    voltwise-ocpp                          voltwise-core
  │                            │                                       │
  │── WebSocket connect ──────►│                                       │
  │                            │ Register(chargerID, ws)               │
  │                            │                                       │
  │── [2,"id","BootNotif",{}]─►│                                       │
  │                            │ ParseMessage(data)                    │
  │                            │ Router.Dispatch(call)                 │
  │                            │   └─ BootNotificationHandler.Handle() │
  │◄─ [3,"id",{Accepted}] ────│                                       │
  │                            │                                       │
  │── [2,"id","StartTx",{}] ──►│                                       │
  │                            │   └─ StartTransactionHandler.Handle() │
  │                            │        └─ CoreService.StartSession() ─►│
  │                            │◄── ChargeSession ─────────────────────│
  │◄─ [3,"id",{txId,Accepted}]│                                       │
  │                            │                                       │
  │── [2,"id","MeterValues",{}]►│                                       │
  │                            │   └─ MeterValuesHandler.Handle()      │
  │                            │        └─ CoreService.RecordMeterValue()►│
  │◄─ [3,"id",{}] ────────────│                                       │
```

---

## Integration

### voltwise-core

The `CoreService` port maps directly to voltwise-core use cases:

| Port Method | Core Use Case |
|---|---|
| `StartSession` | `StartChargingSession` |
| `StopSession` | `StopChargingSession` |
| `UpdateChargerStatus` | `UpdateChargerStatus` |
| `RecordMeterValue` | `RecordMeterValue` |

An adapter struct (not yet implemented) will import voltwise-core and wire repository implementations to the use case structs.

### voltwise-cloud (Django)

voltwise-cloud does not connect to this service directly. Both services share the same PostgreSQL database. The cloud service reads session and charger data written by the OCPP service's core adapters.

In the future, an event bus (Kafka or MQTT) can be used for real-time notifications from OCPP to cloud.

---

## Future Improvements

### OCPP 2.0.1
The handler/router architecture makes it straightforward to add new actions behind a version negotiation layer (subprotocol `ocpp2.0.1`).

### Security
- **TLS** — run behind a TLS-terminating reverse proxy or add TLS config to the HTTP server.
- **Authentication** — implement the `Authenticator` port to validate charger identity tokens during WebSocket upgrade.
- **Authorization** — add an `Authorize` handler for OCPP `Authorize` messages.

### Scaling
- The session manager is per-instance. For horizontal scaling, introduce a shared session store (Redis) and use the `EventPublisher` port for cross-instance coordination.
- Use pod affinity or sticky sessions at the load balancer so chargers reconnect to the same instance when possible.

### Reliability
- Retry logic for domain calls (with backoff) when transient errors occur.
- Dead-letter queue for failed events.
- Structured metrics (Prometheus) for connection counts, message rates, and handler latencies.
