# VoltWise Cloud — Architecture

## Overview

**voltwise-cloud** is the central SaaS backend for the VoltWise EV charging platform.
It is responsible for:

- Multi-tenant **user and organization management**
- **EV charger** registration and status tracking
- **Charging session** lifecycle (open, meter updates, close)
- Public and internal **REST APIs**
- Future: billing, analytics, and a public developer API

It is built with **Django 5** and **Django REST Framework**, backed by **PostgreSQL**, and
designed to scale horizontally behind a load balancer.

---

## Architecture

### Application structure

```
apps/
  common/         Shared utilities: BaseModel (UUID PK + timestamps),
                  StandardPagination, IsOrganizationMember and IsInternalService
                  permission classes.

  users/          Custom User model (email-based auth), JWT register/login endpoints,
                  /users/me profile endpoint.

  organizations/  Organization model.  An organization is the top-level tenant unit.
                  Creating an organization also sets the creator's user.organization FK.

  chargers/       Charger and Location models.
                  Public ViewSet (CRUD scoped to the user's organization).
                  OCPPIntegrationService — the bridge between voltwise-ocpp and this
                  service (see Integration section).
                  Internal API views consumed only by voltwise-ocpp.

  sessions/       ChargingSession model tracking OCPP transactions.
                  Read-only public ViewSet (sessions are opened/closed by voltwise-ocpp).

  billing/        Placeholder.  Will house subscription plans, invoices, and payment
                  methods once the billing integration is implemented.
```

### Request lifecycle (public API)

```
Client
  └─ HTTPS → Nginx / ALB
               └─ gunicorn / uvicorn
                    └─ Django middleware stack
                         └─ JWT authentication (simplejwt)
                              └─ IsOrganizationMember permission
                                   └─ ViewSet → Serializer → PostgreSQL
```

---

## Multi-tenancy

VoltWise uses a **shared database, shared schema** multi-tenancy model enforced at the
application layer.

### How isolation is enforced

| Layer | Mechanism |
|-------|-----------|
| Model | Every resource (`Charger`, `ChargingSession`, `Location`) has a FK to `Organization`. |
| ViewSet | `get_queryset()` always filters by `request.user.organization`. |
| Permission | `IsOrganizationMember` rejects requests from users without an organization. |
| Serializer | `perform_create()` injects `organization=request.user.organization` so users cannot assign resources to a different tenant. |

### User → Organization relationship

```
User ──FK──▶ Organization ──FK──▶ User (owner)
             │
             ├──▶ Charger ──▶ ChargingSession
             └──▶ Location
```

A `User` has an optional FK to `Organization`.  When a user registers, the FK is
`null` until they create or are invited to an organization.  When they call
`POST /api/organizations/`, the serializer sets `user.organization` automatically.

---

## API

All public endpoints are prefixed with `/api/`.  Responses are paginated (page size 20,
configurable via `?page_size=`).  Filtering and ordering are supported via query
parameters on list endpoints.

### Authentication

```
POST /api/auth/register/        Register; returns {id, name, email}
POST /api/auth/login/           Returns {access, refresh} JWT tokens
POST /api/auth/token/refresh/   Exchange a refresh token for a new access token
GET  /api/users/me/             Authenticated user profile
```

### Organizations

```
GET  /api/organizations/        List organizations accessible to the current user
POST /api/organizations/        Create a new organization (caller becomes owner)
GET  /api/organizations/{id}/   Organization detail
```

### Chargers

```
GET   /api/chargers/            List chargers (filter: ?status=, ?location=)
POST  /api/chargers/            Register a new charger
PATCH /api/chargers/{id}/       Update charger name, location, or status
GET   /api/chargers/{id}/       Charger detail
```

### Sessions

```
GET /api/sessions/              List sessions (filter: ?status=, ?charger=)
GET /api/sessions/{id}/         Session detail
```

Sessions are **read-only** via the public API.  They are created and mutated
exclusively by the OCPP integration layer.

### Internal API (voltwise-ocpp only)

Internal routes live under `/api/internal/` and require the
`X-Internal-Api-Key` header.  They are **not** authenticated with JWT.

```
POST /api/internal/chargers/{identifier}/status/   BootNotification / StatusNotification
POST /api/internal/sessions/                       StartTransaction
POST /api/internal/sessions/stop/                  StopTransaction
POST /api/internal/sessions/meter-values/          MeterValues
```

---

## Integration

### voltwise-ocpp

`voltwise-ocpp` is a standalone Go WebSocket server that speaks OCPP 1.6.
It receives messages from physical chargers and forwards state changes to
`voltwise-cloud` over HTTP using the internal API endpoints described above.

```
EV Charger ──OCPP/WS──▶ voltwise-ocpp (Go)
                              │
                POST /api/internal/...
                              │
                         voltwise-cloud (Django)
                              │
                         PostgreSQL
```

The integration contract is documented in `apps/chargers/ocpp_integration.py`
(`OCPPIntegrationService`).  If both services are ever co-located, this class
can be called directly instead of over HTTP.

**Key rules enforced by voltwise-cloud:**
- Only one `ACTIVE` session per charger at a time.
- `charger.status` is automatically updated to `charging` on session open and
  back to `available` on session close.
- Unknown charger identifiers are rejected with `404`.

### voltwise-core

`voltwise-core` is a Go library containing the domain model and business logic
shared across VoltWise services.  `voltwise-cloud` mirrors its core concepts
(charger, session, tariff) as Django models and adds the SaaS multi-tenancy
layer on top.

Future integration points:
- Shared tariff calculation logic (expose via gRPC or REST).
- Event bus (Kafka/Redis Streams) for real-time session updates.

---

## Future Improvements

### Billing

- **Subscription plans** (Free / Pro / Enterprise) with feature flags per org.
- **Usage-based billing** — calculate cost per session using configurable tariffs.
- **Stripe integration** via `djstripe` for invoicing and payment collection.
- **Prepaid credit** system for pay-as-you-go driver accounts.

### Payments & Invoicing

- Invoice generation (PDF) sent automatically at end of billing period.
- Webhook listener for Stripe events (payment succeeded, subscription cancelled).

### Analytics

- Aggregated energy-consumption reports per organization.
- Peak-hour analysis and charger utilisation dashboards.
- Export to CSV / BI tools via a dedicated read-only analytics endpoint.

### Public API

- OAuth 2.0 client credentials flow for third-party integrations.
- API key management UI.
- Rate limiting and quota enforcement per API key.
- OpenAPI / Swagger documentation (`drf-spectacular`).

### Scalability

- Celery workers for async tasks (invoice generation, notification emails).
- Redis caching for charger status (low-latency reads for the public map).
- Read replicas for analytics queries.
- Horizontal scaling behind a load balancer with sticky-session-free JWT auth.
