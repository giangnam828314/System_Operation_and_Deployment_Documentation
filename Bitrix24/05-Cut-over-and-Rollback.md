# 05 — Cut-over & Rollback — Step by Step

## 1. Cut-over objective

Chuyển dịch vụ Bitrix24 từ hệ thống cũ sang hệ thống mới với downtime được kiểm soát và có thể rollback.

---

## 2. T-1 day — Readiness

- [ ] Clone/new environment đã pass Functional Test.
- [ ] Final backup plan approved.
- [ ] Backup destination verified.
- [ ] Restore test successful.
- [ ] Domain/DNS access verified.
- [ ] SSL ready.
- [ ] Rollback path verified.
- [ ] Người quyết định Cut-over/Rollback đã xác định.
- [ ] Người dùng đã được thông báo maintenance window.

---

## 3. T-0 — Freeze old system

### Step 1 — Announce maintenance

Thông báo thời điểm bắt đầu freeze và downtime.

### Step 2 — Stop business changes

Tạm dừng thao tác tạo/chỉnh sửa dữ liệu theo kế hoạch.

### Step 3 — Final backup

Tạo backup Database và Web/source sau khi freeze.

📸 Evidence: timestamp + backup file + checksum.

---

## 4. Final restore to target

### Step 1 — Restore Web

```bash
tar -xzvf /path/to/final_backup.tar.gz -C /home/bitrix/www
```

### Step 2 — Restore DB

```bash
mysql -u <DB_USER> -p <DB_NAME> < /path/to/final_backup.sql
```

### Step 3 — Verify

```bash
systemctl is-active httpd
systemctl is-active php-fpm
systemctl is-active mariadb
```

Run functional smoke test.

---

## 5. Switch Domain/DNS

1. Change DNS/domain endpoint according to approved plan.
2. Wait/observe propagation according to TTL and DNS architecture.
3. Verify from multiple locations where practical.
4. Confirm HTTPS reaches new system.

```bash
getent hosts <DOMAIN>
curl -I https://<DOMAIN>
```

---

## 6. Production smoke test

Run immediately after DNS switch:

- [ ] Login.
- [ ] CRM.
- [ ] Deals.
- [ ] Users.
- [ ] Tasks.
- [ ] Workflows.
- [ ] Upload/download.
- [ ] Database read/write.
- [ ] Email/notification if in scope.
- [ ] Logs show no critical errors.

Record start/end and actual downtime:

```text
Cut-over start: __________
Service available: _______
Actual downtime: _________
```

---

## 7. Rollback trigger

Rollback should be considered when a critical failure prevents safe business operation, for example:

- Login unavailable for critical users.
- CRM/data corruption or missing critical data.
- Database errors that cannot be resolved within the approved window.
- Core application functions unavailable.
- Irrecoverable Web/PHP-FPM failure.
- Security issue that requires immediate retreat to the known-good environment.

The actual threshold must be approved before Cut-over.

---

## 8. Rollback procedure

### Step 1 — Declare rollback

Record decision maker and timestamp.

### Step 2 — Freeze new target

Stop business writes on the new system.

### Step 3 — Switch Domain/DNS back

Point Domain/DNS to the old system.

### Step 4 — Verify old system

```bash
getent hosts <DOMAIN>
curl -I https://<DOMAIN>
```

Run Login / CRM / Deals / Users / Tasks checks.

### Step 5 — Reconcile data

Record any transactions attempted on the new system between Cut-over and Rollback. Do not assume these can be automatically merged.

### Step 6 — Communicate

Inform users that service has been restored on the old system and that the migration is paused/rolled back.

---

## 9. Post-cut-over stabilization

Keep old system available according to retention/rollback policy until the agreed stabilization period is completed.

Monitor:

```bash
systemctl --failed
journalctl -u httpd --since "-2 hours" --no-pager
journalctl -u php-fpm --since "-2 hours" --no-pager
journalctl -u mariadb --since "-2 hours" --no-pager
```

---

## 10. Cut-over sign-off

| Role | Name | Timestamp | Result |
|---|---|---|---|
| Executor | __________ | __________ | PASS / FAIL |
| Technical checker | __________ | __________ | PASS / FAIL |
| Business owner | __________ | __________ | ACCEPT / REJECT |
| Rollback decision owner | __________ | __________ | N/A / Used |

