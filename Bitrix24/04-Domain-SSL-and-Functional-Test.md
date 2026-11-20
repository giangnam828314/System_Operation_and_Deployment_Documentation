# 04 — Domain, SSL & Functional Test

## 1. Domain / DNS

### Step 1 — Configure application domain

Cấu hình Domain/FQDN đúng môi trường target:

```text
https://<DOMAIN>
```

Kiểm tra từ server:

```bash
getent hosts <DOMAIN>
```

### Step 2 — Verify DNS

Xác nhận A/AAAA/CNAME record trỏ tới đúng endpoint tại thời điểm Cut-over.

> Không chuyển DNS Production sớm. DNS switch là một bước trong Cut-over.

📸 Evidence: DNS result + server endpoint.

---

## 2. SSL

### Step 1 — Install/configure certificate

Dùng certificate method đã được phê duyệt.

### Step 2 — Validate HTTPS

```bash
curl -I https://<DOMAIN>
```

**Expected:** HTTP response hợp lệ, HTTPS handshake thành công.

### Step 3 — Browser validation

Mở `https://<DOMAIN>` và kiểm tra:

- Certificate valid.
- Domain đúng.
- Không có mixed content nghiêm trọng.
- Redirect HTTP → HTTPS đúng thiết kế.

📸 Evidence: browser certificate + HTTPS page.

---

## 3. Bitrix system check

Sau migration, kiểm tra administrative settings và chạy công cụ kiểm tra hệ thống của Bitrix để xác nhận server mới đáp ứng technical requirements. Bitrix cũng lưu ý rằng sau khi domain thay đổi cần rà soát các thiết lập liên quan tới domain. citeturn871693search9

---

## 4. Functional test matrix

| Test ID | Test | Expected | Status |
|---|---|---|---|
| FT-01 | Login | User login successful | ☐ |
| FT-02 | Dashboard | Dashboard loads normally | ☐ |
| FT-03 | CRM | CRM opens and data visible | ☐ |
| FT-04 | Companies/Contacts | Data visible if enabled | ☐ |
| FT-05 | Deals | Existing deals visible | ☐ |
| FT-06 | Users | User list accessible | ☐ |
| FT-07 | Tasks | Tasks accessible/create/update | ☐ |
| FT-08 | Kanban | Columns/cards load correctly | ☐ |
| FT-09 | Workflow | Workflow functions correctly | ☐ |
| FT-10 | File Upload | Upload/download successful | ☐ |
| FT-11 | File permissions | No access denied | ☐ |
| FT-12 | Database | Read/write operations succeed | ☐ |
| FT-13 | Domain | Correct FQDN | ☐ |
| FT-14 | SSL | Valid HTTPS certificate | ☐ |
| FT-15 | Web Server | No critical errors in log | ☐ |
| FT-16 | PHP-FPM | Service active / no fatal errors | ☐ |
| FT-17 | Background jobs | Scheduled/agent jobs behave correctly | ☐ |
| FT-18 | Email/notification | Test succeeds if in scope | ☐ |

---

## 5. Data integrity spot check

Chọn dữ liệu đại diện trước/sau migration và đối chiếu:

- Số lượng Users.
- Một số CRM records.
- Một số Deals.
- Một số Tasks.
- Một số file attachment.
- Dữ liệu mới tạo sau restore test.

Ghi nhận:

```text
Baseline count: __________
Post-restore count: _______
Difference: ______________
```

---

## 6. Logs

Check:

```bash
journalctl -u httpd --since "-1 hour" --no-pager
journalctl -u php-fpm --since "-1 hour" --no-pager
journalctl --since "-1 hour" --no-pager | grep -iE 'error|fatal|permission denied'
```

Nếu dùng NGINX, thay `httpd` bằng `nginx`.

**Expected:** không có Critical Error liên quan tới application.

---

## 7. Evidence package

```text
04-acceptance/
├── 01-domain.png
├── 02-dns.png
├── 03-ssl-browser.png
├── 04-login.png
├── 05-crm.png
├── 06-deals.png
├── 07-users.png
├── 08-tasks.png
├── 09-workflow.png
├── 10-upload.png
├── 11-db-validation.png
└── 12-logs.png
```
