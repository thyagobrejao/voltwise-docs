# VoltWise Core — Architecture

## Purpose

**voltwise-core** is the shared domain library for the VoltWise EV charging platform. It contains the business entities, validation rules, use cases, and repository interfaces that are reused across all backend services:

| Service | Language | Role |
|---|---|---|
| **voltwise-cloud** | Django (Python) | Management dashboard, REST API, billing |
| **voltwise-ocpp** | Go | OCPP 1.6 gateway — WebSocket communication with chargers |
| **voltwise-agent** | Go | Edge agent running on-site for local control and telemetry |

By extracting the core domain into its own module, all Go services import the same entity definitions and business rules, while voltwise-cloud can reference the same concepts through its own Python models that mirror these types.

---

## Project Structure

```
voltwise-core/
├── go.mod
├── internal/
│   ├── domain/       # Business entities, value objects, validation, errors
│   ├── ports/        # Repository interfaces (driven-side adapters)
│   └── usecase/      # Application-level operations
└── pkg/              # (future) Shared utilities usable by external consumers
```

**Key design decisions:**

- **`internal/`** — prevents external packages from importing implementation details; only `pkg/` will be importable by consumers.
- **No frameworks** — the domain and use-case layers depend exclusively on the Go standard library.
- **No infrastructure** — persistence, HTTP, OCPP, and messaging are kept out of this module. Each service provides its own adapters.

---

## Domain Model

### Entity Relationship

```
Location 1──* Charger 1──* ChargingSession *──1 User
                                  │
                              Tariff (pricing)
```

### Charger

Represents a physical EV charging station.

| Field | Type | Description |
|---|---|---|
| ID | string | Unique identifier |
| Name | string | Human-readable label |
| LocationID | string | FK to the Location |
| Status | ChargerStatus | `available`, `charging`, `offline`, `fault` |
| Protocol | ChargerProtocol | Communication protocol (`ocpp1.6`) |

### ChargingSession

Represents a single charging event between a user and a charger.

| Field | Type | Description |
|---|---|---|
| ID | string | Unique identifier |
| ChargerID | string | FK to the Charger |
| UserID | string | FK to the User |
| StartTime | time.Time | When the session began |
| EndTime | *time.Time | When the session ended (nil while active) |
| EnergyKWh | float64 | Cumulative energy delivered |
| Status | SessionStatus | `active`, `completed`, `failed` |

### User

A platform user who can initiate charging sessions.

| Field | Type | Description |
|---|---|---|
| ID | string | Unique identifier |
| Name | string | Display name |
| Email | string | Contact email (validated) |

### Location

A physical site that contains one or more chargers.

| Field | Type | Description |
|---|---|---|
| ID | string | Unique identifier |
| Name | string | Site name |
| Address | string | Street address |
| Latitude | float64 | GPS latitude (-90 to 90) |
| Longitude | float64 | GPS longitude (-180 to 180) |

### Tariff

Pricing information for energy consumption.

| Field | Type | Description |
|---|---|---|
| ID | string | Unique identifier |
| PricePerKWh | float64 | Cost per kilowatt-hour |
| Currency | string | ISO 4217 currency code |

---

## Business Rules

1. **One active session per charger** — A charger cannot start a new session while another session is active.
2. **Energy tracking** — Every session must track cumulative energy (`EnergyKWh >= 0`).
3. **Time ordering** — `EndTime` must be after `StartTime` when set.
4. **Status consistency** — When a session starts the charger status transitions to `charging`; when it stops the charger returns to `available`.

---

## Use Cases

### StartChargingSession

Initiates a new session on a charger. Validates that the charger exists, the user exists, and no active session is already running on the charger. Sets the charger status to `charging`.

### StopChargingSession

Finalises the active session for a charger, records the end time, marks it `completed`, and returns the charger to `available`.

### UpdateChargerStatus

Changes a charger's operational status. Guards against invalid transitions — a charger that is `charging` with an active session cannot be set to `available` without stopping the session first.

### RecordMeterValue

Updates the cumulative energy reading on the active session. Typically invoked when an OCPP `MeterValues` message arrives from the charger.

---

## Ports (Interfaces)

Repository interfaces live in `internal/ports/` and are the only allowed dependency between use cases and infrastructure:

- **ChargerRepository** — `FindByID`, `Save`
- **SessionRepository** — `FindByID`, `FindActiveByChargerID`, `Save`
- **UserRepository** — `FindByID`

Each consuming service provides its own implementation (PostgreSQL, in-memory, etc.).

---

## Integration with Other Services

### voltwise-ocpp (Go)

Imports voltwise-core as a Go module. The OCPP gateway:
- Calls `StartChargingSession` / `StopChargingSession` when it receives `StartTransaction` / `StopTransaction` from a charger.
- Calls `RecordMeterValue` when `MeterValues` messages arrive.
- Implements the repository interfaces with a shared PostgreSQL database.

### voltwise-agent (Go)

Also imports voltwise-core. The edge agent:
- Uses the domain types for local session tracking and offline resilience.
- Syncs session state with the cloud via the same entity definitions.

### voltwise-cloud (Django)

Does not import Go code directly. Instead, the Django models mirror the domain entities defined here. The architecture document serves as the single source of truth for field names, types, and business rules to keep both sides consistent.

---

## Testing Strategy

- **Domain tests** — validate entity constraints (required fields, value ranges, time ordering).
- **Use-case tests** — use lightweight in-memory repository implementations to verify business logic without any database or network.
- No mocking frameworks — plain Go test doubles keep the test surface small and readable.
