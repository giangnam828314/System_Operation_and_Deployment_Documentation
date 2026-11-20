# 03 — Permission & Security — Step by Step

## 1. Mục tiêu

Xác nhận Bitrix24 có quyền truy cập đúng, không dùng SSH root trực tiếp, và các thư mục runtime có quyền ghi cần thiết.

---

## 2. Owner / Group baseline

### Step 1 — Check application tree

```bash
ls -ld /home/bitrix/www
find /home/bitrix/www -maxdepth 2 -type d -printf '%u:%g %m %p\n' | head -100
```

### Step 2 — Check critical directories

```bash
ls -ld /home/bitrix/www/upload
ls -ld /home/bitrix/www/bitrix/cache
ls -ld /home/bitrix/www/bitrix/managed_cache
```

**Expected:** owner/group đúng theo runtime account thực tế của hệ thống.

> Không dùng một lệnh `chown -R` hoặc `chmod` hàng loạt chỉ để “cho chạy được”. Quyền phải được kiểm tra theo từng khu vực và xác nhận bằng test thực tế.

---

## 3. Read / Write / Execute test

### Step 1 — Verify permissions

```bash
namei -l /home/bitrix/www/upload
namei -l /home/bitrix/www/bitrix/cache
namei -l /home/bitrix/www/bitrix/managed_cache
```

### Step 2 — Write test

Thực hiện bằng đúng account/service context đã xác định:

```bash
sudo -u <RUNTIME_USER> touch /home/bitrix/www/upload/.permission-test
sudo -u <RUNTIME_USER> rm -f /home/bitrix/www/upload/.permission-test
```

Lặp lại cho cache nếu policy cho phép test trực tiếp.

**Expected:** create/delete thành công.

📸 Evidence: terminal output.

---

## 4. Functional permission test

### Upload file

1. Đăng nhập Bitrix24.
2. Vào module có chức năng upload.
3. Upload file test.
4. Mở lại file.
5. Xóa file test.

**Expected:** upload, read, delete hoạt động bình thường.

### CRM create/update

1. Tạo một record test.
2. Chỉnh sửa.
3. Lưu.
4. Mở lại.
5. Xóa record test sau khi hoàn tất kiểm tra nếu phù hợp.

**Expected:** dữ liệu tạo/cập nhật bình thường.

---

## 5. Permission denied monitoring

### Web Server log

```bash
journalctl -u httpd --since "-30 min" --no-pager
```

Nếu dùng NGINX:

```bash
journalctl -u nginx --since "-30 min" --no-pager
```

### System log

```bash
journalctl --since "-30 min" --no-pager | grep -iE 'permission denied|denied|selinux'
```

**Expected:** không còn lỗi permission liên quan tới application sau functional test.

---

## 6. SELinux

Kiểm tra trạng thái:

```bash
getenforce
```

Nếu `Enforcing`, phải xác nhận context phù hợp với Web Server/PHP-FPM. Không disable SELinux chỉ để bỏ qua lỗi; cần xử lý đúng context/policy theo chuẩn của môi trường.

📸 Evidence: `getenforce` và context liên quan nếu có.

---

## 7. SSH security checklist

- [ ] Admin account riêng đã hoạt động.
- [ ] `sudo` hoạt động.
- [ ] `PermitRootLogin no`.
- [ ] SSH port đúng policy.
- [ ] Không commit private key/password vào repository.
- [ ] Có ít nhất một phương án truy cập quản trị dự phòng trước khi thay đổi SSH config.

---

## 8. Evidence package

```text
03-security/
├── 01-owner-group.png
├── 02-critical-permissions.png
├── 03-write-test.png
├── 04-upload-test.png
├── 05-crm-test.png
├── 06-logs-no-permission-denied.png
├── 07-selinux.png
└── 08-ssh-hardening.png
```

---

## 9. Source alignment

Tài liệu nguồn yêu cầu kiểm tra Owner/Group, Read/Write/Execute, write permission tại `/upload/`, `/bitrix/cache/`, `/bitrix/managed_cache/`, test Upload/CRM và xác nhận không có `Permission Denied` trong log. Đây là cơ sở của checklist trong tài liệu này.
