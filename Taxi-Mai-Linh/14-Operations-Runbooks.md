# 14 — VẬN HÀNH VÀ SỔ TAY XỬ LÝ SỰ CỐ

## 1. Mô hình hỗ trợ

| Tier | Responsibility |
|---|---|
| L1 Service Desk | Login/device/app/network triage and ticket intake |
| L2 Application | Booking/dispatch/trip/API diagnosis |
| L2 Infrastructure | Network, compute, DB, queue, storage, backup |
| L3 Development/Vendor | Code defects, integration faults, hotfix |
| Business Operations | Dispatch fallback and customer operational decisions |

## 2. Danh sách kiểm tra giám sát

- API health, p95 latency and error ratio.
- DB connections, replication, storage and slow queries.
- Queue lag, retry count and dead-letter events.
- GPS ingestion and location freshness.
- Push delivery and Driver App sync queue.
- Booking backlog, no-candidate and offer expiry.
- Payment callback failures and unmatched transactions.
- Settlement calculation failures.
- Certificate/domain expiry and secret rotation.
- Backup jobs and restore verification.

## 3. Sổ tay xử lý — GPS suy giảm

1. Check ingestion service, provider and network.
2. Compare device_time with server received_at.
3. Identify device/carrier/zone/global scope.
4. Mark stale freshness; do not infer vehicle is offline.
5. Contact driver through approved channel if required.
6. Apply dispatch restrictions only under approved safety policy.
7. Record incident, affected trips, timeline and recovery.

## 4. Sổ tay xử lý — Điều phối suy giảm

1. Check API, candidate service, DB and queue.
2. Distinguish no eligible cars from candidate service failure.
3. Preserve booking queue; prevent silent cancellation.
4. Use manual dispatch only under authorized contingency.
5. Record override and notify supervisor.
6. Reconcile assignment/events after recovery.

## 5. Sổ tay xử lý — Callback thanh toán thất bại

1. Check provider and webhook endpoint.
2. Verify signature, timestamp and delivery logs.
3. Query provider transaction status securely.
4. Reconcile by provider reference/idempotency key.
5. Do not mark paid based only on screenshot/customer statement.
6. Create exception and Finance review.
7. Confirm ledger only after authoritative status.

## 6. Sao lưu và khôi phục thảm họa

Define RPO/RTO, backup type/frequency, encryption, offsite/immutable copy, retention, legal hold, restore test frequency, DR failover/failback and evidence. Values are TBD. A successful backup job is not proof of recoverability.

## 7. Hồ sơ sự cố

incident_id, severity, start/end, affected service/region, detection source, owner/commander, timeline, customer/driver impact, workaround, recovery, root cause, corrective actions and linked booking/trip/payment IDs.

SEV-1 safety/security/financial integrity event must follow approved on-call escalation.
