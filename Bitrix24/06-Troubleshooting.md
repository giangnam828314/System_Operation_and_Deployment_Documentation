# 06 — Troubleshooting

> Dùng tài liệu này khi một bước trong Installation/Restore/Acceptance bị Fail.

## 1. PHP-FPM service stopped

### Symptom

```text
502 Bad Gateway / PHP not executed / php-fpm inactive
```

### Check

```bash
systemctl status php-fpm --no-pager
journalctl -u php-fpm -n 100 --no-pager
php -v
```

### Typical actions

```bash
dnf install php-fpm -y
systemctl enable --now php-fpm
```

Sau đó kiểm tra lại PHP-FPM configuration và Web Server integration.

**Do not:** restart loop hoặc downgrade/upgrade PHP trực tiếp trên Production mà không kiểm tra compatibility.

---

## 2. Web Server not starting

### Check

Apache:

```bash
apachectl configtest
systemctl status httpd --no-pager
journalctl -u httpd -n 100 --no-pager
```

NGINX:

```bash
nginx -t
systemctl status nginx --no-pager
journalctl -u nginx -n 100 --no-pager
```

### Resolution

Sửa lỗi config được báo bởi config test, sau đó:

```bash
systemctl restart <service>
```

---

## 3. Permission denied

### Check

```bash
namei -l /home/bitrix/www/upload
namei -l /home/bitrix/www/bitrix/cache
namei -l /home/bitrix/www/bitrix/managed_cache
```

Check current process/runtime user and logs.

```bash
journalctl --since "-30 min" --no-pager | grep -i "permission denied"
```

### Resolution principle

1. Xác định user thực sự chạy Web/PHP-FPM.
2. Xác định path đang bị deny.
3. Kiểm tra owner/group/mode.
4. Kiểm tra SELinux context nếu đang Enforcing.
5. Sửa tối thiểu đúng phạm vi.
6. Re-test bằng functional test.

Không áp dụng `chmod -R 777`.

---

## 4. Database restore failed

### Check

```bash
systemctl status mariadb --no-pager
mysql --version
mysql -u <DB_USER> -p <DB_NAME>
```

Review import command and first error in output/log.

### Common causes

- DB không tồn tại.
- User không có quyền.
- Sai charset/collation.
- SQL dump không đầy đủ/corrupt.
- Disk hết dung lượng.
- Version compatibility issue.

### Verification

```bash
df -h
mysql -u <DB_USER> -p -e "USE <DB_NAME>; SHOW TABLES;"
```

---

## 5. Site loads but data missing

Kiểm tra:

1. DB connection config.
2. Database name.
3. Imported tables.
4. Source code version.
5. File attachments/uploads.
6. Domain/application settings.

Bitrix documents that database restore is a distinct restore stage and that connection settings are read from application configuration; after migration, administrative/domain settings should also be reviewed. citeturn871693search4turn871693search9

---

## 6. File upload fails

Check:

```bash
ls -ld /home/bitrix/www/upload
namei -l /home/bitrix/www/upload
df -h /home/bitrix/www/upload
```

Then:

- Verify runtime user write permission.
- Verify disk space.
- Verify PHP upload limits relevant to deployment.
- Check Web/PHP logs.

---

## 7. HTTPS / SSL error

```bash
curl -Iv https://<DOMAIN>
```

Check:

- DNS resolves to intended endpoint.
- Certificate SAN contains domain.
- Certificate not expired.
- Web Server presents expected certificate.
- HTTP → HTTPS behavior is correct.

---

## 8. Domain points to old server

```bash
getent hosts <DOMAIN>
```

Check DNS record, TTL and resolver caching. During Cut-over, use the approved DNS switch procedure; do not modify records ad hoc.

---

## 9. System log shows unexpected errors

```bash
journalctl --since "-1 hour" --no-pager
journalctl -p err --since "-1 hour" --no-pager
```

Record:

```text
Timestamp:
Service:
Error:
Impact:
Action:
Verification:
```

---

## 10. Incident evidence

Mỗi troubleshooting case phải lưu:

```text
06-troubleshooting/
├── before.png
├── error-log.txt
├── command-output.txt
├── action-taken.txt
└── after.png
```

Không đưa password, API token, private key hoặc dữ liệu CRM thật vào evidence commit.
