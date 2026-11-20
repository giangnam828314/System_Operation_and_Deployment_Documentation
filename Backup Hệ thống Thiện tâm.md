# Backup Hệ thống Thiện tâm

## 1. Mục tiêu

Tài liệu này mô tả quy trình backup, restore và migration hệ thống Thiện Tâm (AgriBase) từ VPS cũ sang VPS mới chạy Ubuntu 24.04, aaPanel, Docker và Docker Compose.

Nguyên tắc xuyên suốt: **mỗi bước phải kiểm tra kết quả trước khi chuyển sang bước tiếp theo**.

Mục tiêu:

- Không mất dữ liệu.
- Giữ nguyên source code, Docker images và cấu hình.
- Khôi phục PostgreSQL/PostGIS.
- Khôi phục MinIO và dữ liệu media.
- Khôi phục Caddy và HTTPS.
- Cho khách hàng nghiệm thu trên domain test mà không ảnh hưởng production.
- Khi cutover, lấy **dữ liệu mới nhất từ VPS cũ** rồi mới chuyển DNS production.
- Giữ VPS cũ làm phương án rollback.

---

## 2. Kiến trúc hệ thống

```text
Internet
   |
   v
Public/NAT
   |
   v
VPS mới
   |
   +-- Caddy :80/:443
          |
          +-- Frontend :4321
          +-- Backend :3000
          +-- MinIO :9000/:9001
                    |
                    +-- PostgreSQL/PostGIS :5432
```

Trong giai đoạn nghiệm thu:

```text
sauriengthientam.vn
    -> VPS cũ (production)

thientam.review360.id.vn
    -> VPS mới (test)
```

Sau khi nghiệm thu:

```text
sauriengthientam.vn
    -> VPS mới
```

---

# PHẦN A — KIỂM TRA VPS MỚI

## 3. Kiểm tra hệ điều hành

```bash
cat /etc/os-release
uname -a
```

**Đạt khi:** Ubuntu 24.04 LTS, kiến trúc x86_64 và kernel hoạt động bình thường.

---

## 4. Kiểm tra disk

```bash
df -h /
```

Đảm bảo đủ chỗ cho Docker images, PostgreSQL, MinIO, source, log và backup.

---

## 5. Kiểm tra RAM/Swap

```bash
free -h
```

Đảm bảo đủ RAM/Swap để chạy PostgreSQL, MinIO, Backend, Frontend và Caddy.

---

## 6. Kiểm tra Docker

```bash
docker --version
docker compose version
docker ps
```

**Đạt khi:** Docker và Docker Compose hoạt động bình thường, chưa có container rác gây xung đột.

---

## 7. Kiểm tra aaPanel

```bash
bt status
```

`bt --version` có thể không được aaPanel hỗ trợ; không dùng lệnh đó làm tiêu chí kiểm tra chính.

---

## 8. Kiểm tra port

```bash
ss -lntp
aaPanel/Nginx không được chiếm 80/443 nếu Caddy Docker là reverse proxy chính.
```

Sau khi Caddy chạy phải thấy host listen:

```text
:80
:443
```

---

# PHẦN B — BACKUP VPS CŨ

## 9. Cấu trúc hệ thống đã xác định

Project:

```text
/opt/agribase/thientam
```

Compose files:

```text
docker-compose.yml
docker-compose.override.yml
```

Service:

```text
db
minio
backend
frontend
caddy
```

Volumes:

```text
thientam_pgdata
thientam_miniodata
thientam_caddyconfig
thientam_caddydata
```

---

## 10. Kiểm tra dung lượng trước backup

```bash
du -sh /opt/agribase/thientam
du -sh /var/lib/docker/volumes/thientam_miniodata/_data
du -sh /var/lib/docker/volumes/thientam_pgdata/_data
docker system df
df -h /
```

Mục đích là đảm bảo backup không làm đầy disk.

---

## 11. Backup Docker images

Các image của hệ thống:

```text
thientam-backend:latest
thientam-frontend:latest
postgis/postgis:16-3.4
quay.io/minio/minio:latest
caddy:2
```

Tạo thư mục:

```bash
mkdir -p /backup/thientam/{images,project,database,volumes}
```

Backup:

```bash
docker save \
  thientam-backend:latest \
  thientam-frontend:latest \
  postgis/postgis:16-3.4 \
  quay.io/minio/minio:latest \
  caddy:2 \
  | gzip -1 > /backup/thientam/images/thientam-images.tar.gz
```

Kiểm tra:

```bash
ls -lh /backup/thientam/images/
gzip -t /backup/thientam/images/thientam-images.tar.gz
```

**Chỉ qua bước tiếp theo khi `gzip -t` không báo lỗi.**

---

## 12. Backup project/config

```bash
tar -czpf /backup/thientam/project/thientam-project.tar.gz \
  -C /opt/agribase thientam
```

Kiểm tra:

```bash
ls -lh /backup/thientam/project/
tar -tzf /backup/thientam/project/thientam-project.tar.gz | head -30
```

Archive phải chứa các file/thư mục quan trọng như:

```text
backend-src/
frontend-src/
.env
backend.env
frontend.env
Caddyfile
docker-compose.yml
docker-compose.override.yml
```

**Không đưa secret thật lên GitHub/public repository.**

---

## 13. Backup PostgreSQL/PostGIS

Không dùng `pg_dumpall` nếu role `postgres` không tồn tại. Dùng đúng user/database của ứng dụng:

```bash

docker exec thientam-db-1 \
  pg_dump \
  -U thientam_user \
  -d thientam_db \
  > /backup/thientam/database/thientam_db.sql
```

Nén:

```bash
gzip -1 /backup/thientam/database/thientam_db.sql
```

Kiểm tra:

```bash
ls -lh /backup/thientam/database/
gzip -t /backup/thientam/database/thientam_db.sql.gz
```

Kiểm tra nội dung:

```bash
zcat /backup/thientam/database/thientam_db.sql.gz | head -20
```

Database của hệ thống dùng PostGIS; phải tiếp tục restore trên image:

```text
postgis/postgis:16-3.4
```

---

## 14. Backup MinIO

Dữ liệu nằm ở:

```text
/var/lib/docker/volumes/thientam_miniodata/_data
```

Backup:

```bash
tar -czpf \
  /backup/thientam/volumes/thientam_miniodata.tar.gz \
  -C /var/lib/docker/volumes/thientam_miniodata/_data .
```

Kiểm tra:

```bash
ls -lh /backup/thientam/volumes/
tar -tzf /backup/thientam/volumes/thientam_miniodata.tar.gz | head -30
```

---

## 15. Backup Caddy config/data

```bash
tar -czpf \
  /backup/thientam/volumes/thientam_caddyconfig.tar.gz \
  -C /var/lib/docker/volumes/thientam_caddyconfig/_data .
```

```bash
tar -czpf \
  /backup/thientam/volumes/thientam_caddydata.tar.gz \
  -C /var/lib/docker/volumes/thientam_caddydata/_data .
```

Kiểm tra từng archive trước khi qua bước kế tiếp.

---

## 16. Kiểm tra toàn bộ backup

```bash
find /backup/thientam -type f -exec ls -lh {} \;
du -sh /backup/thientam
```

Kỳ vọng có 6 file chính:

```text
images/thientam-images.tar.gz
project/thientam-project.tar.gz
database/thientam_db.sql.gz
volumes/thientam_miniodata.tar.gz
volumes/thientam_caddydata.tar.gz
volumes/thientam_caddyconfig.tar.gz
```

---

## 17. Tạo SHA256 checksum

```bash
cd /backup/thientam
sha256sum \
  images/thientam-images.tar.gz \
  project/thientam-project.tar.gz \
  database/thientam_db.sql.gz \
  volumes/thientam_miniodata.tar.gz \
  volumes/thientam_caddydata.tar.gz \
  volumes/thientam_caddyconfig.tar.gz \
  > SHA256SUMS
```

Kiểm tra:

```bash
cat SHA256SUMS
```

---

## 18. Đóng gói để chuyển sang VPS mới

```bash
cd /backup
tar -cf thientam-backup.tar thientam/
```

Kiểm tra:

```bash
ls -lh /backup/thientam-backup.tar
tar -tf /backup/thientam-backup.tar | head -30
```

Có thể tải file này về máy tính để lưu offline.

---

# PHẦN C — CHUYỂN BACKUP SANG VPS MỚI

## 19. Tạo thư mục

```bash
mkdir -p /opt/agribase
```

Đưa file:

```text
/opt/agribase/thientam-backup.tar
```

---

## 20. Kiểm tra archive

```bash
ls -lh /opt/agribase/thientam-backup.tar
tar -tf /opt/agribase/thientam-backup.tar | head -30
```

---

## 21. Giải nén

```bash
cd /opt/agribase
tar -xf thientam-backup.tar
```

---

## 22. Kiểm tra checksum trên VPS mới

```bash
cd /opt/agribase/thientam
sha256sum -c SHA256SUMS
```

Kết quả bắt buộc:

```text
images/thientam-images.tar.gz: OK
project/thientam-project.tar.gz: OK
database/thientam_db.sql.gz: OK
volumes/thientam_miniodata.tar.gz: OK
volumes/thientam_caddydata.tar.gz: OK
volumes/thientam_caddyconfig.tar.gz: OK
```

**Nếu có một dòng FAILED: dừng migration và xử lý checksum.**

---

# PHẦN D — RESTORE IMAGES

## 23. Load 5 image

```bash
cd /opt/agribase/thientam
gunzip -c images/thientam-images.tar.gz | docker load
```

Kiểm tra:

```bash
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

Phải có đủ 5 image.

---

# PHẦN E — RESTORE PROJECT

## 24. Khôi phục source/config

Nếu archive tạo thư mục lồng `thientam/thientam`, dùng:

```bash
mv /opt/agribase/thientam /opt/agribase/thientam-backup
mkdir -p /opt/agribase/thientam

tar -xzf /opt/agribase/thientam-backup/project/thientam-project.tar.gz \
  -C /opt/agribase/thientam
```

Nếu kết quả là:

```text
/opt/agribase/thientam/thientam/
```

thì:

```bash
cp -a /opt/agribase/thientam/thientam/. /opt/agribase/thientam/
rm -rf /opt/agribase/thientam/thientam
```

Kiểm tra:

```bash
ls -lah /opt/agribase/thientam
```

---

# PHẦN F — KIỂM TRA COMPOSE

## 25. Service list

```bash
cd /opt/agribase/thientam
docker compose config --services
```

Kỳ vọng:

```text
db
minio
backend
frontend
caddy
```

---

## 26. Volume list

```bash
docker compose config --volumes
```

Kỳ vọng:

```text
caddyconfig
pgdata
miniodata
caddydata
```

---

## 27. Validate

```bash
docker compose config -q
```

**Không có output/error = đạt.**

---

# PHẦN G — TẠO/RESTORE VOLUME

## 28. Tạo 4 volume

```bash
docker volume create thientam_pgdata
docker volume create thientam_miniodata
docker volume create thientam_caddyconfig
docker volume create thientam_caddydata
```

Kiểm tra:

```bash
docker volume ls | grep thientam
```

---

## 29. Restore MinIO

```bash
tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_miniodata.tar.gz \
-C /var/lib/docker/volumes/thientam_miniodata/_data
```

Kiểm tra:

```bash
du -sh /var/lib/docker/volumes/thientam_miniodata/_data
find /var/lib/docker/volumes/thientam_miniodata/_data -maxdepth 2 -type d | head -20
```

Phải thấy bucket/directory `thientam-media` và `.minio.sys`.

---

## 30. Restore Caddy volumes

```bash
tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_caddyconfig.tar.gz \
-C /var/lib/docker/volumes/thientam_caddyconfig/_data
```

```bash
tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_caddydata.tar.gz \
-C /var/lib/docker/volumes/thientam_caddydata/_data
```

Kiểm tra hai volume trước khi chạy Caddy.

---

# PHẦN H — POSTGRESQL/POSTGIS

## 31. Start riêng DB

```bash
cd /opt/agribase/thientam
docker compose up -d db
```

Kiểm tra:

```bash
docker compose ps
docker compose logs --tail=50 db
```

Phải thấy:

```text
database system is ready to accept connections
```

---

## 32. Kiểm tra PostgreSQL

```bash
docker exec thientam-db-1 \
  psql -U thientam_user -d thientam_db -c "SELECT version();"
```

---

## 33. Kiểm tra PostGIS

```bash
docker exec thientam-db-1 \
  psql -U thientam_user -d thientam_db -c "SELECT PostGIS_Version();"
```

Phải ra PostGIS 3.4.x theo image đã restore.

---

## 34. Restore database

```bash
gunzip -c \
/opt/agribase/thientam-backup/database/thientam_db.sql.gz \
| docker exec -i thientam-db-1 \
psql -U thientam_user -d thientam_db
```

Kiểm tra:

```bash
docker exec thientam-db-1 \
  psql -U thientam_user -d thientam_db -c "\dt public.*"
```

Kiểm tra số bảng:

```bash
docker exec thientam-db-1 \
  psql -U thientam_user -d thientam_db \
  -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';"
```

Ở hệ thống đã migrate thành công, tổng số bảng public + các bảng PostGIS hệ thống đã được kiểm tra; database ứng dụng có đầy đủ các bảng nghiệp vụ.

---

# PHẦN I — START MINIO/BACKEND/FRONTEND

## 35. Start MinIO

```bash
docker compose up -d minio
docker compose logs --tail=50 minio
```

Kiểm tra `Up` và không có lỗi.

---

## 36. Start Backend

```bash
docker compose up -d backend
docker compose ps
docker compose logs --tail=100 backend
```

Test health trong container:

```bash
docker exec thientam-backend-1 \
node -e "fetch('http://127.0.0.1:3000/health').then(async r=>{console.log('HTTP',r.status);console.log(await r.text())}).catch(e=>{console.error(e);process.exit(1)})"
```

Phải:

```text
HTTP 200
{"status":"OK",...}
```

---

## 37. Start Frontend

```bash
docker compose up -d frontend
docker compose logs --tail=100 frontend
```

Test:

```bash
docker exec thientam-frontend-1 \
node -e "fetch('http://127.0.0.1:4321/').then(async r=>{console.log('HTTP',r.status);console.log((await r.text()).slice(0,500))}).catch(e=>{console.error(e);process.exit(1)})"
```

Phải:

```text
HTTP 200
```

---

# PHẦN J — CADDY

## 38. Cấu hình Caddy

Production:

```caddy
{
    email admin@salefronts.net
}

sauriengthientam.vn {
    handle /api* {
        reverse_proxy backend:3000
    }

    handle /graphql* {
        reverse_proxy backend:3000
    }

    handle_path /media* {
        reverse_proxy minio:9000
    }

    handle {
        reverse_proxy frontend:4321
    }
}

sauriengthientam.vn:9000 {
    reverse_proxy minio:9000
}

sauriengthientam.vn:9001 {
    reverse_proxy minio:9001
}
```

Domain test:

```caddy
thientam.review360.id.vn {
    handle /api* {
        reverse_proxy backend:3000
    }

    handle /graphql* {
        reverse_proxy backend:3000
    }

    handle_path /media* {
        reverse_proxy minio:9000
    }

    handle {
        reverse_proxy frontend:4321
    }
}
```

---

## 39. Validate Caddy

```bash
docker compose exec caddy \
caddy validate --config /etc/caddy/Caddyfile
```

Phải có:

```text
Valid configuration
```

Cảnh báo `Caddyfile input is not formatted` chỉ là cảnh báo format, không phải lỗi cấu hình.

---

## 40. Reload Caddy

```bash
docker compose exec caddy \
caddy reload --config /etc/caddy/Caddyfile
```

---

## 41. Publish ports

`docker-compose.yml` của Caddy phải có:

```yaml
ports:
  - "80:80"
  - "443:443"
  - "9000:9000"
  - "9001:9001"
```

Kiểm tra:

```bash
docker compose config -q
```

Sau đó:

```bash
docker compose up -d caddy
```

Kiểm tra:

```bash
docker compose ps
ss -lntp | grep -E ':80 |:443 '
```

---

# PHẦN K — TEST LOCAL CADDY

## 42. HTTP redirect

```bash
curl -I http://127.0.0.1
```

Kỳ vọng:

```text
308 Permanent Redirect
Server: Caddy
```

---

## 43. HTTPS Frontend

```bash
curl -Ik \
  --resolve sauriengthientam.vn:443:127.0.0.1 \
  https://sauriengthientam.vn/
```

Kỳ vọng:

```text
HTTP/2 200
Via: 1.1 Caddy
```

---

## 44. API Docs

```bash
curl -ik \
  --resolve sauriengthientam.vn:443:127.0.0.1 \
  https://sauriengthientam.vn/api-docs
```

Kỳ vọng HTTP 200.

---

## 45. MinIO API

```bash
curl -ik \
  --resolve sauriengthientam.vn:9000:127.0.0.1 \
  https://sauriengthientam.vn:9000/
```

`403 AccessDenied` từ MinIO vẫn có thể là kết quả đúng vì root anonymous không được phép truy cập; điều quan trọng là response đến từ MinIO qua Caddy.

---

## 46. MinIO Console

```bash
curl -ik \
  --resolve sauriengthientam.vn:9001:127.0.0.1 \
  https://sauriengthientam.vn:9001/
```

Kỳ vọng HTTP 200 và `Server: MinIO Console`.

---

# PHẦN L — FIREWALL/NAT

## 47. Kiểm tra UFW

```bash
ufw status
```

Tối thiểu phải cho phép:

```text
80/tcp
443/tcp
```

Không public PostgreSQL 5432 nếu không cần.

---

## 48. Kiểm tra host listening

```bash
ss -lntp | grep -E ':80 |:443 |:9000 |:9001 '
```

---

# PHẦN M — DOMAIN TEST

## 49. DNS test

Tạo record DNS:

```text
A  thientam  <Public/NAT IP của VPS mới>
```

Kiểm tra:

```bash
dig +short thientam.review360.id.vn
```

---

## 50. Test SSL domain test

Xem log:

```bash
docker compose logs --tail=100 caddy | grep -i "thientam.review360.id.vn"
```

Kết quả cuối cùng phải là tương tự:

```text
certificate obtained successfully
```

Caddy có thể thử `tls-alpn-01` trước và nếu không được thì chuyển sang `http-01`; chỉ cần cuối cùng certificate được cấp thành công.

---

## 51. Test domain test

```bash
curl -Ik https://thientam.review360.id.vn/
```

```bash
curl -Ik https://thientam.review360.id.vn/api-docs
```

Kỳ vọng HTTP 200 và response qua Caddy.

---

# PHẦN N — NGHIỆM THU KHÁCH HÀNG

Khách test trên:

```text
https://thientam.review360.id.vn
```

Checklist:

- [ ] Đăng nhập
- [ ] Logout
- [ ] Session/cookie
- [ ] Phân quyền
- [ ] Xem dữ liệu cũ
- [ ] Tìm kiếm
- [ ] Thêm dữ liệu
- [ ] Sửa dữ liệu
- [ ] Xóa dữ liệu
- [ ] Upload ảnh/file
- [ ] Hiển thị ảnh
- [ ] Download file
- [ ] REST API
- [ ] GraphQL
- [ ] Các nghiệp vụ chính

Trong thời gian này:

```text
sauriengthientam.vn        -> VPS cũ
thientam.review360.id.vn   -> VPS mới
```

**Không chuyển DNS production trước khi khách xác nhận OK.**

---

# PHẦN O — FINAL CUTOVER

## 52. Nguyên tắc dữ liệu

Trong toàn bộ thời gian nghiệm thu:

```text
VPS cũ = Source of Truth
```

Không lấy dữ liệu từ VPS test để ghi ngược về production.

Khi khách xác nhận OK, mới lấy dữ liệu mới nhất từ VPS cũ.

---

## 53. Đóng băng production cũ

Đưa hệ thống vào maintenance hoặc dừng backend:

```bash
cd /opt/agribase/thientam
docker compose stop backend
```

Mục tiêu: không còn ghi DB hoặc upload media mới trong thời gian final backup.

---

## 54. FINAL BACKUP PostgreSQL trên VPS cũ

```bash
mkdir -p /backup/thientam-final/database

docker exec thientam-db-1 \
pg_dump \
  -U thientam_user \
  -d thientam_db \
  --format=plain \
  --no-owner \
  --no-privileges \
  > /backup/thientam-final/database/thientam_db_final.sql

gzip -1 /backup/thientam-final/database/thientam_db_final.sql

gzip -t /backup/thientam-final/database/thientam_db_final.sql.gz
```

---

## 55. FINAL BACKUP MinIO trên VPS cũ

Sau khi backend đã dừng:

```bash
mkdir -p /backup/thientam-final/volumes

tar -czf \
/backup/thientam-final/volumes/thientam_miniodata_final.tar.gz \
-C /var/lib/docker/volumes/thientam_miniodata/_data .
```

Kiểm tra:

```bash
tar -tzf \
/backup/thientam-final/volumes/thientam_miniodata_final.tar.gz \
| head -30
```

---

## 56. Chuyển final backup sang VPS mới

```bash
scp \
/backup/thientam-final/database/thientam_db_final.sql.gz \
root@NEW_VPS:/opt/agribase/thientam-backup/database/
```

```bash
scp \
/backup/thientam-final/volumes/thientam_miniodata_final.tar.gz \
root@NEW_VPS:/opt/agribase/thientam-backup/volumes/
```

Có thể dùng rsync nếu file lớn.

---

## 57. Dừng service trên VPS mới để restore

```bash
cd /opt/agribase/thientam
docker compose stop backend frontend minio
```

DB vẫn có thể chạy.

---

## 58. Restore FINAL database

Nếu cần refresh hoàn toàn database:

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d postgres \
-c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='thientam_db' AND pid <> pg_backend_pid();"
```

**Cảnh báo:** các lệnh `DROP DATABASE`/`CREATE DATABASE` là destructive. Chỉ chạy sau khi final backup đã kiểm tra và chắc chắn không cần giữ dữ liệu mới phát sinh trên VPS test.

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d postgres \
-c "DROP DATABASE thientam_db;"
```

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d postgres \
-c "CREATE DATABASE thientam_db OWNER thientam_user;"
```

Restore:

```bash
gunzip -c \
/opt/agribase/thientam-backup/database/thientam_db_final.sql.gz \
| docker exec -i thientam-db-1 \
psql -U thientam_user -d thientam_db
```

Kiểm tra bảng và dữ liệu nghiệp vụ trước khi start backend.

---

## 59. Restore FINAL MinIO

Xóa dữ liệu MinIO cũ trên VPS mới sau khi final backup được xác nhận:

```bash
rm -rf /var/lib/docker/volumes/thientam_miniodata/_data/*
```

Restore:

```bash
tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_miniodata_final.tar.gz \
-C /var/lib/docker/volumes/thientam_miniodata/_data
```

Kiểm tra:

```bash
du -sh /var/lib/docker/volumes/thientam_miniodata/_data
find /var/lib/docker/volumes/thientam_miniodata/_data -maxdepth 2 -type d
```

---

## 60. Start lại hệ thống

```bash
docker compose up -d
```

Kiểm tra:

```bash
docker compose ps
```

Tất cả 5 service phải `Up`.

---

## 61. Final test nội bộ

Backend:

```bash
docker exec thientam-backend-1 \
node -e "fetch('http://127.0.0.1:3000/health').then(async r=>{console.log(r.status);console.log(await r.text())})"
```

Frontend:

```bash
docker exec thientam-frontend-1 \
node -e "fetch('http://127.0.0.1:4321/').then(async r=>console.log(r.status))"
```

Database:

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d thientam_db -c "SELECT version();"
```

PostGIS:

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d thientam_db -c "SELECT PostGIS_Version();"
```

---

## 62. Final test domain test

```bash
curl -Ik https://thientam.review360.id.vn/
curl -Ik https://thientam.review360.id.vn/api-docs
```

Khách hoặc kỹ thuật viên mở browser và kiểm tra dữ liệu mới nhất, media mới nhất và nghiệp vụ chính.

---

# PHẦN P — CHUYỂN DNS PRODUCTION

## 63. Đổi DNS

Từ:

```text
sauriengthientam.vn -> VPS cũ
```

sang:

```text
sauriengthientam.vn -> NAT/Public IP của VPS mới
```

Không tắt VPS cũ ngay.

---

## 64. Kiểm tra DNS

```bash
dig +short sauriengthientam.vn
```

Kiểm tra từ máy bên ngoài:

```powershell
Resolve-DnsName sauriengthientam.vn
```

---

## 65. Kiểm tra production sau DNS cutover

```bash
curl -Ik https://sauriengthientam.vn/
curl -Ik https://sauriengthientam.vn/api-docs
```

Kiểm tra trên browser:

- [ ] Login
- [ ] Dữ liệu mới nhất
- [ ] Upload
- [ ] Download
- [ ] Ảnh/media
- [ ] API
- [ ] GraphQL
- [ ] Nghiệp vụ chính

---

# PHẦN Q — ROLLBACK

Nếu VPS mới có lỗi nghiêm trọng sau cutover:

1. Không xóa VPS cũ.
2. Đổi DNS production lại VPS cũ.
3. Kiểm tra production cũ.
4. Xác định nguyên nhân VPS mới.
5. Chỉ cutover lại sau khi đã khắc phục và nghiệm thu.

VPS cũ phải được giữ nguyên trong thời gian rollback window.

---

# PHẦN R — CHECKLIST CUỐI

| # | Hạng mục | Điều kiện đạt |
|---|---|---|
| 1 | Ubuntu | 24.04 LTS |
| 2 | Docker | Hoạt động |
| 3 | Compose | Hoạt động |
| 4 | Images | 5 image có đủ |
| 5 | Project | Đủ source/config |
| 6 | SHA256 | ALL OK |
| 7 | PostgreSQL | Ready |
| 8 | PostGIS | 3.4.x |
| 9 | Database | Đủ bảng/dữ liệu |
| 10 | MinIO | Đủ bucket/dữ liệu |
| 11 | Backend | `/health` 200 |
| 12 | Frontend | `/` 200 |
| 13 | Caddy | Valid configuration |
| 14 | Port 80 | Listen |
| 15 | Port 443 | Listen |
| 16 | SSL | Certificate OK |
| 17 | Domain test | 200 |
| 18 | API Docs | 200 |
| 19 | Khách nghiệm thu | OK |
| 20 | Final DB backup | OK |
| 21 | Final MinIO backup | OK |
| 22 | Final restore | OK |
| 23 | DNS production | VPS mới |
| 24 | Production test | OK |
| 25 | VPS cũ | Giữ lại rollback |

---

# PHẦN S — NGUYÊN TẮC VẬN HÀNH

1. **Không xóa VPS cũ trước khi migration được nghiệm thu hoàn toàn.**
2. **Không restore database production khi backend còn ghi dữ liệu.**
3. **Không restore MinIO khi MinIO đang chạy.**
4. **Luôn kiểm tra SHA256 sau khi chuyển backup.**
5. **VPS cũ là source of truth cho dữ liệu trong suốt thời gian nghiệm thu.**
6. **Final backup DB + MinIO phải lấy từ VPS cũ ngay trước cutover.**
7. **Domain test dùng để nghiệm thu; domain production chỉ chuyển sau khi khách xác nhận.**
8. **Không đưa `.env`, password, API key hoặc secret thật lên GitHub/public repository.**
9. **Mỗi checkpoint phải đạt rồi mới thực hiện bước tiếp theo.**
10. **Luôn có phương án rollback về VPS cũ.**

---

## Phụ lục — Bộ lệnh kiểm tra nhanh

### Container

```bash
docker compose ps
```

### DB

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d thientam_db -c "SELECT version();"
```

### PostGIS

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d thientam_db -c "SELECT PostGIS_Version();"
```

### Backend

```bash
docker exec thientam-backend-1 \
node -e "fetch('http://127.0.0.1:3000/health').then(async r=>{console.log(r.status);console.log(await r.text())})"
```

### Frontend

```bash
docker exec thientam-frontend-1 \
node -e "fetch('http://127.0.0.1:4321/').then(async r=>console.log(r.status))"
```

### Caddy

```bash
docker compose exec caddy \
caddy validate --config /etc/caddy/Caddyfile
```

### Port

```bash
ss -lntp | grep -E ':80 |:443 '
```

### DNS

```bash
dig +short sauriengthientam.vn
dig +short thientam.review360.id.vn
```

### Domain test

```bash
curl -Ik https://thientam.review360.id.vn/
curl -Ik https://thientam.review360.id.vn/api-docs
```

---

## Ghi chú bảo mật

Tài liệu chỉ chứa quy trình và câu lệnh. Không lưu password thật, token, private key, `.env` production hoặc dữ liệu database thật trong repository tài liệu.
