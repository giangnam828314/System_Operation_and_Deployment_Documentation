# 10 — ĐẶC TẢ API VÀ TÍCH HỢP HỆ THỐNG

## 1. Quy ước API

- REST/JSON over HTTPS unless an approved alternative is documented.
- Versioned prefix `/api/v1`; OpenAPI is source of truth.
- OAuth2/OIDC or approved service authentication.
- Idempotency for create, financial and event commands.
- UTC timestamps; branch timezone for user-local scheduling.
- Pagination/filter/sort limits; standard error envelope and correlation ID.
- State and permission validation on server.

## 2. Endpoint catalogue

| Method | Endpoint | Purpose |
|---|---|---|
| POST | /api/v1/bookings | Create booking |
| GET/PATCH | /api/v1/bookings/{id} | Read/update guarded booking |
| POST | /api/v1/dispatch/candidates | Find eligible vehicles |
| POST | /api/v1/assignments | Create offer/assignment |
| POST | /api/v1/assignments/{id}/accept | Accept offer |
| POST | /api/v1/assignments/{id}/reject | Reject offer |
| POST | /api/v1/driver/trips/{id}/events | Submit driver event |
| POST | /api/v1/trips/{id}/complete | Complete trip |
| GET | /api/v1/trips/{id}/timeline | Read timeline |
| POST | /api/v1/payments | Initiate payment |
| POST | /api/v1/webhooks/payments/{provider} | Payment callback |
| POST | /api/v1/settlements | Create batch |
| POST | /api/v1/settlements/{id}/approve | Approve batch |
| POST | /api/v1/settlements/{id}/lock | Lock batch |

## 3. Example booking payload

```json
{
  "channel": "HOTLINE",
  "customer": {
    "phone": "+84XXXXXXXXX",
    "name": "Customer Example"
  },
  "pickup": {
    "formatted_address": "Pickup address",
    "lat": 10.0001,
    "lon": 106.0001,
    "place_ref": "provider-place-reference"
  },
  "destination": {
    "formatted_address": "Destination address",
    "lat": 10.0101,
    "lon": 106.0101
  },
  "service_type": "STANDARD",
  "scheduled_at": null,
  "passenger_count": 2,
  "notes": "Pickup instruction"
}
```

Example only; required fields and catalog must be confirmed.

## 4. Error response

```json
{
  "error": {
    "code": "BOOKING_VALIDATION_FAILED",
    "message": "Booking information is invalid.",
    "details": [{"field": "pickup", "reason": "PICKUP_LOCATION_REQUIRED"}],
    "correlation_id": "request-id"
  }
}
```

## 5. Event envelope

event_id, event_type, aggregate_type, aggregate_id, occurred_at_device, received_at_server, actor_type, actor_id, source_app, device_id, schema_version, correlation_id, payload.

## 6. Integration catalogue

| Integration | Direction | Control | Decision |
|---|---|---|---|
| Maps/geocoding | Query | API key scope, quota, provider place ref | Provider TBD |
| GPS ingestion | Driver/vehicle → platform | Authentication, freshness, rate limit | Protocol TBD |
| Push | Platform → Driver App | Delivery, retry, token lifecycle | Provider TBD |
| SMS/Voice | Platform ↔ provider | Consent, callback, cost | Provider TBD |
| Payment | Platform ↔ provider | Signature, idempotency, reconciliation | Provider TBD |
| CRM/ERP | Platform ↔ enterprise | Mapping, ownership, retry | Scope TBD |
| Identity | User ↔ IAM | OIDC/SAML, MFA, lifecycle | Provider TBD |

## 7. Reliability

Retry transient failures with backoff and dead-letter; validate webhook signatures/timestamps; retain idempotency keys through provider retry horizon; circuit-break degraded integrations; reconcile missing callbacks; monitor queue lag and delivery failures.
