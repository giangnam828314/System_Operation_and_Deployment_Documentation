# Quy trình Migration Hệ thống từ VPS cũ sang VPS mới - Sử dụng Docker

> **Hệ thống tham chiếu:** AgriBase / Thiên Tâm (`sauriengthientam.vn`)
>
> **Mục tiêu:** Di chuyển nguyên trạng hệ thống Docker từ VPS cũ sang VPS mới, giữ nguyên application images, source code, PostgreSQL/PostGIS, MinIO, Caddy và cấu hình reverse proxy; cho phép dựng VPS mới và cho khách nghiệm thu bằng domain test trước khi cutover production.
>
> **Nguyên tắc bắt buộc:** Mỗi bước phải đi theo chuỗi **Thực hiện → Kiểm tra → Kết quả đạt → mới chuyển bước tiếp theo**.
>
> **Nguồn dữ liệu production:** Trong suốt giai đoạn nghiệm thu, VPS cũ là **source of truth**. Khi khách xác nhận OK, phải thực hiện **Final Cutover Backup** từ VPS cũ rồi mới restore dữ liệu cuối cùng sang VPS mới và đổi DNS.
>
> **Lưu ý bảo mật:** Tài liệu này cố ý **không ghi lại mật khẩu/secret thực tế** đã xuất hiện trong log hoặc `docker-compose.yml`. Không commit password, API key, private key, token hoặc file `.env` chứa secret lên Git. Nếu cần dùng lại secret, lấy trực tiếp từ cấu hình production đã được quản lý an toàn.

---

## 1. Khung tổng thể của quy trình

Quy trình được chia thành 12 giai đoạn:

1. **Khảo sát và đóng băng hiện trạng VPS cũ**
2. **Chuẩn bị VPS mới** (Ubuntu 24.04 + aaPanel + Docker)
3. **Backup hệ thống từ VPS cũ**
4. **Kiểm tra tính toàn vẹn backup bằng SHA-256**
5. **Đóng gói backup để chuyển sang VPS mới**
6. **Khôi phục project và Docker images trên VPS mới**
7. **Khôi phục Docker volumes / dữ liệu nền tảng**
8. **Khởi động và kiểm tra DB → MinIO → Backend → Frontend → Caddy**
9. **Cấu hình HTTP/HTTPS và domain test**
10. **Nghiệm thu hệ thống trên domain test mà không ảnh hưởng production**
11. **Final Cutover: lấy dữ liệu mới nhất từ VPS cũ và restore sang VPS mới**
12. **Đổi DNS production, xác nhận dịch vụ và giữ phương án rollback**

### Mô hình sau migration

```text
                    ┌─────────────────────────┐
                    │       VPS CŨ            │
                    │  Production / Source    │
                    │                         │
                    │  Docker                 │
                    │  PostgreSQL/PostGIS     │
                    │  MinIO                  │
                    │  Caddy                  │
                    └────────────┬────────────┘
                                 │
                         Backup / Final Backup
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │       VPS MỚI           │
                    │  Ubuntu 24.04           │
                    │  aaPanel                │
                    │  Docker                 │
                    │                         │
                    │  DB / MinIO / Backend   │
                    │  Frontend / Caddy       │
                    └────────────┬────────────┘
                                 │
                     Domain test / nghiệm thu
                                 │
                                 ▼
                  thientam.review360.id.vn

Sau khi nghiệm thu:

sauriengthientam.vn  ─────────────► VPS MỚI
```

---

# PHẦN A — KHẢO SÁT VÀ CHUẨN BỊ

## Bước 1. Xác định trạng thái VPS cũ

### Thực hiện

Trên VPS cũ xác định:

- OS
- hostname
- dung lượng đĩa
- RAM/swap
- Docker version
- Docker Compose version
- danh sách container
- image đang dùng
- volume đang dùng
- project path
- domain/IP hiện tại

Các lệnh khảo sát chuẩn:

```bash
cat /etc/os-release
uname -a
df -h /
free -h
docker --version
docker compose version
docker ps
docker images
docker volume ls
```

### Kiểm tra

Ghi lại tối thiểu:

```text
Project path: /opt/agribase/thientam
Backup path:  /backup/thientam
```

Các thành phần của Thiên Tâm đã xác định trong lần migration này:

```text
thientam-backend:latest
thientam-frontend:latest
postgis/postgis:16-3.4
quay.io/minio/minio:latest
caddy:2
```

### Kết quả đạt

- Xác định đủ 5 Docker images.
- Xác định project Compose.
- Xác định PostgreSQL/PostGIS là dữ liệu database cần backup.
- Xác định MinIO volume là dữ liệu file/media cần backup.
- Xác định Caddy data/config cần backup để giữ certificate/config state.

→ **Đạt mới chuyển Bước 2.**

---

## Bước 2. Kiểm tra VPS mới

### Thực hiện

VPS mới được kiểm tra thực tế trong migration:

```text
Ubuntu 24.04.4 LTS
Kernel: 6.8.0-137-generic x86_64
Disk: 490G total / khoảng 456G available lúc khảo sát
RAM: 7.8GiB
Swap: 4.0GiB
Docker: 29.7.2
Docker Compose: v5.4.0
aaPanel: đã cài và đang chạy
```

Kiểm tra:

```bash
cat /etc/os-release
uname -a
df -h /
free -h
docker --version
docker compose version
bt status
```

### Kiểm tra aaPanel / port

Trong lần migration thực tế:

```bash
ss -lntp
```

cho thấy aaPanel/webserver đang sử dụng các port riêng, trong đó panel/SSH của máy sử dụng port tùy chỉnh; Nginx của aaPanel có thời điểm đã được stop để nhường 80/443 cho Caddy.

Kiểm tra:

```bash
systemctl status nginx --no-pager
systemctl is-enabled nginx
```

Nếu Nginx aaPanel không còn giữ 80/443 thì Caddy mới có thể publish 80/443.

### Kết quả đạt

VPS mới đủ tài nguyên để nhận hệ thống. Docker và Compose chạy bình thường. aaPanel không chiếm 80/443 khi Caddy chạy production.

→ **Đạt mới chuyển Bước 3.**

---

# PHẦN B — BACKUP VPS CŨ

## Bước 3. Backup Docker images

### Thực hiện

Đã thực hiện trên VPS cũ:

```bash
docker save \
  thientam-backend:latest \
  thientam-frontend:latest \
  postgis/postgis:16-3.4 \
  quay.io/minio/minio:latest \
  caddy:2 \
  | gzip -1 > /backup/thientam/images/thientam-images.tar.gz
```

### Tối ưu dung lượng

Sử dụng:

```bash
gzip -1
```

thay vì mức nén cao hơn để giảm thời gian CPU trong khi vẫn có mức nén tốt. Docker image layer đã được đóng gói một lần trong tar rồi gzip.

Không nên chạy `zip` thêm lần nữa trên `tar.gz` chỉ để giảm dung lượng; `tar.gz` đã là archive nén phù hợp. Nếu cần một file duy nhất để tải về thì có thể bọc toàn bộ thư mục backup thành một `tar`/`tar.gz` duy nhất, nhưng không nên nén lồng nhiều lớp một cách không cần thiết.

### Kiểm tra

```bash
ls -lh /backup/thientam/images/
gzip -t /backup/thientam/images/thientam-images.tar.gz
du -sh /backup/thientam
```

Kết quả thực tế:

```text
thientam-images.tar.gz ≈ 811M
```

### Kết quả đạt

`gzip -t` không trả lỗi → file gzip hợp lệ.

→ **Đạt mới chuyển Bước 4.**

---

## Bước 4. Backup project/source/config

### Thực hiện

Project được đóng gói riêng:

```text
/backup/thientam/project/thientam-project.tar.gz
```

Kích thước thực tế:

```text
≈ 26M
```

Backup này chứa project Docker/Source/Caddyfile/Compose và các tài liệu/source cần thiết theo hiện trạng lúc backup.

### Kiểm tra

```bash
ls -lh /backup/thientam/project/thientam-project.tar.gz
tar -tzf /backup/thientam/project/thientam-project.tar.gz | head -30
```

Trong lần restore thực tế archive có cấu trúc:

```text
thientam/
thientam/backend-src/
thientam/frontend-src/
...
```

### Kết quả đạt

Archive đọc được và chứa project.

→ **Đạt mới chuyển Bước 5.**

---

## Bước 5. Backup PostgreSQL/PostGIS

### Thực hiện

Database được dump riêng thành SQL gzip:

```text
/backup/thientam/database/thientam_db.sql.gz
```

Kích thước thực tế của backup ban đầu:

```text
≈ 1.1M
```

Cách backup chuẩn cho **final cutover** nên dùng chính user/database của ứng dụng, không dùng `pg_dumpall` nếu không cần dump toàn cluster/roles.

Ví dụ:

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
```

Kiểm tra:

```bash
gzip -t /backup/thientam-final/database/thientam_db_final.sql.gz
zcat /backup/thientam-final/database/thientam_db_final.sql.gz | head -20
```

### Kết quả đạt

- SQL gzip đọc được.
- Database dump có thể dùng để restore độc lập.
- PostGIS extensions được khôi phục trên DB mới.

→ **Đạt mới chuyển Bước 6.**

---

## Bước 6. Backup Docker volumes

Các volume đã backup trong migration thực tế:

```text
thientam_caddydata.tar.gz      ≈ 5.8K
thientam_miniodata.tar.gz      ≈ 13M
thientam_caddyconfig.tar.gz    ≈ 530B
```

### 6.1 MinIO

Backup volume:

```bash
tar -czf \
/backup/thientam/volumes/thientam_miniodata.tar.gz \
-C /var/lib/docker/volumes/thientam_miniodata/_data .
```

Kiểm tra:

```bash
tar -tzf /backup/thientam/volumes/thientam_miniodata.tar.gz | head -30
du -sh /var/lib/docker/volumes/thientam_miniodata/_data
```

Trong lần restore thực tế, dữ liệu khoảng 14M và có:

```text
thientam-media/
thientam-media/uploads/
thientam-media/image/
.minio.sys/
```

### 6.2 Caddy config

```bash
tar -czf \
/backup/thientam/volumes/thientam_caddyconfig.tar.gz \
-C /var/lib/docker/volumes/thientam_caddyconfig/_data .
```

### 6.3 Caddy data / certificate storage

```bash
tar -czf \
/backup/thientam/volumes/thientam_caddydata.tar.gz \
-C /var/lib/docker/volumes/thientam_caddydata/_data .
```

### Kết quả đạt

Các archive volume tồn tại và `tar -tzf` đọc được.

→ **Đạt mới chuyển Bước 7.**

---

## Bước 7. Tạo SHA256SUMS

### Thực hiện

Đã thực hiện:

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

Backup thực tế đã tạo các checksum:

```text
images/thientam-images.tar.gz
project/thientam-project.tar.gz
database/thientam_db.sql.gz
volumes/thientam_miniodata.tar.gz
volumes/thientam_caddydata.tar.gz
volumes/thientam_caddyconfig.tar.gz
```

### Kiểm tra

```bash
cat /backup/thientam/SHA256SUMS
```

### Kết quả đạt

Có checksum SHA-256 cho toàn bộ 6 file backup.

→ **Đạt mới chuyển Bước 8.**

---

# PHẦN C — ĐÓNG GÓI VÀ CHUYỂN BACKUP

## Bước 8. Đóng gói thành một file để tải/chuyển

### Thực hiện

Do VPS cũ có dung lượng trống thấp, không nên tạo nhiều bản sao không cần thiết.

Trong lần migration thực tế, tổng backup:

```text
≈ 850M
```

Đã đóng gói thành:

```text
/opt/agribase/thientam-backup.tar
```

kích thước:

```text
≈ 850M
```

Cấu trúc archive:

```text
thientam/
thientam/images/thientam-images.tar.gz
thientam/project/thientam-project.tar.gz
thientam/database/thientam_db.sql.gz
thientam/volumes/thientam_caddydata.tar.gz
thientam/volumes/thientam_miniodata.tar.gz
thientam/volumes/thientam_caddyconfig.tar.gz
thientam/SHA256SUMS
```

### Kiểm tra

```bash
ls -lh /opt/agribase/thientam-backup.tar
tar -tf /opt/agribase/thientam-backup.tar | head -30
```

### Kết quả đạt

Archive đọc được và chứa đủ 6 thành phần backup + SHA256SUMS.

→ **Đạt mới chuyển Bước 9.**

---

# PHẦN D — KHÔI PHỤC TRÊN VPS MỚI

## Bước 9. Tạo thư mục và giải nén backup

### Thực hiện

Trên VPS mới:

```bash
mkdir -p /opt/agribase
```

Đưa `thientam-backup.tar` vào:

```text
/opt/agribase/thientam-backup.tar
```

Giải nén:

```bash
tar -xf /opt/agribase/thientam-backup.tar -C /opt/agribase
```

### Kiểm tra

```bash
ls -lh /opt/agribase/
cd /opt/agribase/thientam
sha256sum -c SHA256SUMS
```

Trong migration thực tế, **tất cả đều OK**:

```text
images/thientam-images.tar.gz: OK
project/thientam-project.tar.gz: OK
database/thientam_db.sql.gz: OK
volumes/thientam_miniodata.tar.gz: OK
volumes/thientam_caddydata.tar.gz: OK
volumes/thientam_caddyconfig.tar.gz: OK
```

### Kết quả đạt

Checksum trên VPS mới khớp 100% với VPS cũ.

→ **Đạt mới chuyển Bước 10.**

---

## Bước 10. Load Docker images

### Thực hiện

```bash
gunzip -c images/thientam-images.tar.gz | docker load
```

### Kiểm tra

```bash
docker images

docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

Kết quả thực tế:

```text
thientam-backend      latest    ≈ 1.97GB
thientam-frontend     latest    ≈ 1.7GB
caddy                 2         ≈ 128MB
quay.io/minio/minio   latest    ≈ 355MB
postgis/postgis       16-3.4    ≈ 1.26GB
```

### Kết quả đạt

Đủ 5 images, đúng tên/tag cần dùng.

→ **Đạt mới chuyển Bước 11.**

---

## Bước 11. Restore project đúng cấp thư mục

### Thực hiện

Archive project có thư mục gốc `thientam/`, vì vậy lần restore đầu tiên tạo:

```text
/opt/agribase/thientam/thientam/
```

Để project thực tế nằm đúng:

```text
/opt/agribase/thientam/
```

đã thực hiện:

```bash
cd /opt/agribase
mv thientam thientam-backup
mkdir -p /opt/agribase/thientam

tar -xzf \
  /opt/agribase/thientam-backup/project/thientam-project.tar.gz \
  -C /opt/agribase/thientam

cp -a /opt/agribase/thientam/thientam/. \
      /opt/agribase/thientam/

rm -rf /opt/agribase/thientam/thientam
```

### Kiểm tra

```bash
ls -lah /opt/agribase/thientam
```

Phải thấy trực tiếp:

```text
backend.env
backend.env.bak
backend-src/
backup.sh
Caddyfile
Caddyfile.bak
docker-compose.yml
docker-compose.yml.bak
.env
frontend.env
frontend-src/
```

### Kết quả đạt

Project nằm đúng `/opt/agribase/thientam`, Compose file đọc được.

→ **Đạt mới chuyển Bước 12.**

---

## Bước 12. Validate Docker Compose trước khi tạo container

### Thực hiện

```bash
cd /opt/agribase/thientam

docker compose config --services
docker compose config --volumes
docker compose config -q
```

Trong migration thực tế:

```text
services:
db
minio
backend
frontend
caddy
```

Volumes:

```text
caddyconfig
pgdata
miniodata
caddydata
```

`docker compose config -q` không báo lỗi.

### Kết quả đạt

Compose syntax và interpolation hợp lệ.

→ **Đạt mới chuyển Bước 13.**

---

# PHẦN E — RESTORE VOLUMES

## Bước 13. Tạo Docker volumes

### Thực hiện

```bash
docker volume create thientam_pgdata
docker volume create thientam_miniodata
docker volume create thientam_caddyconfig
docker volume create thientam_caddydata
```

### Kiểm tra

```bash
docker volume ls | grep thientam
```

Kết quả thực tế:

```text
thientam_caddyconfig
thientam_caddydata
thientam_miniodata
thientam_pgdata
```

### Lưu ý Compose warning

Compose đã cảnh báo:

```text
volume "thientam_pgdata" already exists but was not created by Docker Compose.
Use `external: true` to use an existing volume
```

Điều này xảy ra vì volume được tạo thủ công trước Compose. Nó **không làm migration thất bại**; container vẫn mount đúng volume. Tuy nhiên, sau khi hệ thống ổn định nên chuẩn hóa `docker-compose.yml` để khai báo các volume đã quản lý thủ công là `external: true`, hoặc để Compose tự tạo volume theo một quy ước thống nhất.

Không tự ý đổi tên volume trong lúc migration.

→ **Đạt mới chuyển Bước 14.**

---

## Bước 14. Restore MinIO volume

### Thực hiện

Khi MinIO chưa chạy:

```bash
tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_miniodata.tar.gz \
-C /var/lib/docker/volumes/thientam_miniodata/_data
```

### Kiểm tra

```bash
du -sh /var/lib/docker/volumes/thientam_miniodata/_data
find /var/lib/docker/volumes/thientam_miniodata/_data -maxdepth 2 -type d | head -20
```

Migration thực tế cho thấy:

```text
≈14M
```

và có:

```text
thientam-media/
thientam-media/uploads/
thientam-media/image/
.minio.sys/
```

### Kết quả đạt

MinIO data đã tồn tại trước khi container MinIO được start.

→ **Đạt mới chuyển Bước 15.**

---

## Bước 15. Restore Caddy config/data

### Thực hiện

```bash
tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_caddyconfig.tar.gz \
-C /var/lib/docker/volumes/thientam_caddyconfig/_data

tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_caddydata.tar.gz \
-C /var/lib/docker/volumes/thientam_caddydata/_data
```

### Kiểm tra

```bash
ls -lah /var/lib/docker/volumes/thientam_caddyconfig/_data
ls -lah /var/lib/docker/volumes/thientam_caddydata/_data
```

Migration thực tế đã thấy:

```text
caddyconfig/_data/caddy
caddydata/_data/caddy
```

### Kết quả đạt

Caddy storage/config đã được khôi phục.

→ **Đạt mới chuyển Bước 16.**

---

# PHẦN F — KHỞI ĐỘNG TỪNG DỊCH VỤ

## Bước 16. Khởi động PostgreSQL/PostGIS

### Thực hiện

```bash
cd /opt/agribase/thientam
docker compose up -d db
```

### Kiểm tra

```bash
docker compose ps
docker compose logs --tail=50 db
```

Kiểm tra PostgreSQL:

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d thientam_db \
-c "SELECT version();"
```

Kiểm tra PostGIS:

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d thientam_db \
-c "SELECT PostGIS_Version();"
```

Migration thực tế:

```text
PostgreSQL 16.4
PostGIS 3.4
```

### Kết quả đạt

DB container `Up`, PostgreSQL accept connections, PostGIS extension hoạt động.

**Quan trọng:** Khi container DB vừa init lần đầu, `\dt` có thể chỉ thấy các bảng PostGIS/TIGER/topology. Đó **chưa phải** là dữ liệu nghiệp vụ nếu database dump chưa được restore. Sau final restore phải kiểm tra lại các bảng nghiệp vụ.

→ **Đạt mới chuyển Bước 17.**

---

## Bước 17. Restore database nghiệp vụ

### Thực hiện

Trong migration ban đầu DB mới đã được init bằng:

```text
DB: thientam_db
User: thientam_user
```

Sau khi DB chạy ổn, ở giai đoạn restore chính thức có thể restore dump SQL.

Đối với **final cutover**, cách an toàn là dừng backend/frontend trước, terminate connections, drop/recreate database rồi restore dump final:

```bash
docker compose stop backend frontend
```

Kiểm tra DB:

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d postgres \
-c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='thientam_db' AND pid <> pg_backend_pid();"
```

Drop/recreate:

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d postgres \
-c "DROP DATABASE thientam_db;"

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

### Kiểm tra

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d thientam_db \
-c "SELECT PostGIS_Version();"

docker exec thientam-db-1 \
psql -U thientam_user -d thientam_db \
-c "\dt public.*"
```

Kiểm tra thêm một số bảng nghiệp vụ quan trọng bằng `SELECT count(*) ...`.

### Kết quả đạt

- Có bảng nghiệp vụ.
- Dữ liệu count khớp với VPS cũ tại thời điểm backup.
- PostGIS hoạt động.

→ **Đạt mới chuyển Bước 18.**

---

## Bước 18. Khởi động MinIO

### Thực hiện

```bash
docker compose up -d minio
```

### Kiểm tra

```bash
docker compose ps
docker compose logs --tail=50 minio
```

Migration thực tế cho thấy MinIO chạy:

```text
API: http://172.18.0.x:9000
WebUI: https://sauriengthientam.vn:9001
```

### Kiểm tra trực tiếp

API root không nhất thiết trả 200 vì anonymous root access có thể bị từ chối:

```bash
curl -ik --resolve sauriengthientam.vn:9000:127.0.0.1 \
https://sauriengthientam.vn:9000/
```

Migration thực tế trả:

```text
HTTP/2 403
Server: MinIO
<Code>AccessDenied</Code>
```

Đây là kết quả phù hợp với anonymous request vào root MinIO.

Console:

```bash
curl -ik --resolve sauriengthientam.vn:9001:127.0.0.1 \
https://sauriengthientam.vn:9001/
```

Migration thực tế trả:

```text
HTTP/2 200
server: MinIO Console
```

### Kết quả đạt

MinIO container chạy, volume có dữ liệu, Console trả 200.

→ **Đạt mới chuyển Bước 19.**

---

## Bước 19. Khởi động Backend

### Thực hiện

```bash
docker compose up -d backend
```

### Kiểm tra container

```bash
docker compose ps
```

Backend không publish port 3000 ra host; port này chỉ tồn tại trong Docker network.

Do container backend không có `curl`/`wget`, không dùng:

```bash
docker exec thientam-backend-1 curl ...
```

hoặc:

```bash
docker exec thientam-backend-1 wget ...
```

vì image không có hai binary này.

### Kiểm tra đúng bằng Node

```bash
docker exec thientam-backend-1 \
node -e "fetch('http://127.0.0.1:3000/health').then(async r=>{console.log('HTTP',r.status);console.log(await r.text())}).catch(e=>{console.error(e);process.exit(1)})"
```

Migration thực tế:

```text
HTTP 200
{"status":"OK","environment":"production",...}
```

Kiểm tra port:

```bash
docker exec thientam-backend-1 \
node -e "require('net').createConnection({host:'127.0.0.1',port:3000}).on('connect',()=>{console.log('PORT 3000 OPEN');process.exit(0)}).on('error',e=>{console.error('PORT 3000 ERROR:',e.message);process.exit(1)})"
```

Container IP trong migration:

```text
172.18.0.4
```

### Kiểm tra API Docs nội bộ

Sau khi Caddy chạy, endpoint public sẽ được test bằng `/api-docs`.

### Kết quả đạt

Backend health 200 và port 3000 mở trong container.

→ **Đạt mới chuyển Bước 20.**

---

## Bước 20. Khởi động Frontend

### Thực hiện

```bash
docker compose up -d frontend
```

### Kiểm tra

```bash
docker compose logs --tail=100 frontend
```

Migration thực tế:

```text
[@astrojs/node] Server listening on
local: http://localhost:4321
network: http://172.18.0.5:4321
```

Kiểm tra bằng Node:

```bash
docker exec thientam-frontend-1 \
node -e "fetch('http://127.0.0.1:4321/').then(async r=>{console.log('HTTP',r.status);console.log((await r.text()).slice(0,500))}).catch(e=>{console.error(e);process.exit(1)})"
```

Migration thực tế trả `HTTP 200` và HTML AgriBase.

### Kết quả đạt

Frontend chạy ổn định trên port 4321 trong Docker network.

→ **Đạt mới chuyển Bước 21.**

---

# PHẦN G — CADDY / HTTP / HTTPS

## Bước 21. Kiểm tra Caddyfile

Cấu hình production đã sử dụng:

```caddyfile
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

### Domain test

Để nghiệm thu trước khi đổi production DNS, bổ sung:

```caddyfile
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

### Kiểm tra Caddyfile

```bash
docker compose exec caddy \
caddy validate --config /etc/caddy/Caddyfile
```

Migration thực tế:

```text
Valid configuration
```

Có warning format:

```text
Caddyfile input is not formatted; run 'caddy fmt --overwrite'
```

Đây là warning format, không phải lỗi cấu hình.

Có thể chuẩn hóa sau:

```bash
docker compose exec caddy \
caddy fmt --overwrite /etc/caddy/Caddyfile
```

### Reload

```bash
docker compose exec caddy \
caddy reload --config /etc/caddy/Caddyfile
```

### Kết quả đạt

Caddy validate = `Valid configuration`.

→ **Đạt mới chuyển Bước 22.**

---

## Bước 22. Publish port 80 và 443

Ban đầu Compose chỉ publish 443:

```text
443 -> 443
9000 -> 9000
9001 -> 9001
```

Điều này làm HTTP port 80 chưa ra host.

Đã sửa `docker-compose.yml` để Caddy publish thêm 80:

```yaml
ports:
  - "80:80"
  - "443:443"
  - "9000:9000"
  - "9001:9001"
```

### Kiểm tra

```bash
docker compose config -q

docker compose up -d caddy

docker compose ps

ss -lntp | grep -E ':80 |:443 '
```

Migration thực tế:

```text
0.0.0.0:80->80/tcp
0.0.0.0:443->443/tcp
```

Test HTTP:

```bash
curl -I http://127.0.0.1
```

Kết quả:

```text
HTTP/1.1 308 Permanent Redirect
Location: https://127.0.0.1/
Server: Caddy
```

### Lưu ý test HTTPS bằng IP

Lệnh:

```bash
curl -Ik https://127.0.0.1
```

có thể trả lỗi TLS vì hostname/SNI là `127.0.0.1`, không phải domain có certificate.

Phải test bằng hostname:

```bash
curl -Ik --resolve sauriengthientam.vn:443:127.0.0.1 \
https://sauriengthientam.vn/
```

Migration thực tế trả:

```text
HTTP/2 200
Via: 1.1 Caddy
```

### Kết quả đạt

- TCP 80 mở trên host.
- TCP 443 mở trên host.
- HTTP tự redirect sang HTTPS.
- HTTPS theo đúng hostname trả 200.

→ **Đạt mới chuyển Bước 23.**

---

# PHẦN H — KIỂM TRA ROUTING

## Bước 23. Kiểm tra `/health`

### Thực hiện

Không dùng:

```bash
https://domain/health
```

để kiểm tra backend vì Caddyfile hiện tại chỉ proxy các path `/api*` và `/graphql*` sang backend. `/health` sẽ đi vào frontend.

Migration thực tế:

```text
GET /health -> HTTP 404
```

HTML 404 là trang Astro frontend.

### Kết quả đạt

`/health` 404 qua domain là **đúng với routing hiện tại**, không phải backend chết.

Backend đã được kiểm tra riêng bằng Node trong container với HTTP 200.

→ **Đạt mới chuyển Bước 24.**

---

## Bước 24. Kiểm tra API Docs

### Thực hiện

```bash
curl -ik --resolve sauriengthientam.vn:443:127.0.0.1 \
https://sauriengthientam.vn/api-docs
```

### Kiểm tra

Migration thực tế trả:

```text
HTTP/2 200
content-type: text/html; charset=utf-8
```

Nội dung là `AgriBase API Docs` / Swagger UI.

### Kết quả đạt

Caddy route `/api*` → backend:3000 hoạt động.

→ **Đạt mới chuyển Bước 25.**

---

## Bước 25. Kiểm tra GraphQL

### Thực hiện

```bash
curl -ik --resolve sauriengthientam.vn:443:127.0.0.1 \
https://sauriengthientam.vn/graphql
```

Migration thực tế trả:

```text
HTTP/2 400
{"errors":[{"message":"Internal Server Error",...}]}
```

### Giải thích

GET `/graphql` không phải một GraphQL operation hợp lệ nếu API yêu cầu query/body/operation tương ứng. Vì vậy không dùng một GET rỗng để kết luận GraphQL hỏng.

Phải test bằng request GraphQL thực tế của ứng dụng hoặc từ frontend.

Một lần test `docker exec ... node -e ...` bị:

```text
SyntaxError: Unexpected end of input
```

Đó là lỗi do câu lệnh `node -e` truyền vào bị thiếu cú pháp, **không phải bằng chứng backend/GraphQL bị lỗi**.

### Kết quả đạt

Routing `/graphql` đã tới backend vì response là JSON/GraphQL-style, nhưng nghiệm thu cuối cùng phải thực hiện bằng query thật.

→ **Đạt mới chuyển Bước 26.**

---

## Bước 26. Kiểm tra MinIO qua Caddy

### `/media/`

```bash
curl -ik --resolve sauriengthientam.vn:443:127.0.0.1 \
https://sauriengthientam.vn/media/
```

Migration thực tế trả:

```text
HTTP/2 403
Server: MinIO
AccessDenied
```

Điều này cho thấy request đã tới MinIO. Root/bucket listing anonymous có thể bị từ chối; cần test bằng **URL object thật** của hệ thống.

### Console

```bash
curl -ik --resolve sauriengthientam.vn:9001:127.0.0.1 \
https://sauriengthientam.vn:9001/
```

Migration thực tế trả `HTTP/2 200` và HTML MinIO Console.

### Kết quả đạt

MinIO routing hoạt động. Nghiệm thu file phải dùng object thật: mở ảnh, download file, upload file và đọc lại file.

→ **Đạt mới chuyển Bước 27.**

---

# PHẦN I — SSL / DNS / DOMAIN TEST

## Bước 27. Kiểm tra certificate tự động

Caddy tự quản lý TLS cho domain khi domain trỏ tới máy và port 80/443 được mở.

Log migration thực tế với domain test:

```text
obtaining certificate
trying to solve challenge tls-alpn-01
challenge failed
trying http-01
served key authentication
authorization finalized
certificate obtained successfully
```

Cụ thể:

```text
identifier: thientam.review360.id.vn
challenge_type: http-01
CA: Let's Encrypt
certificate obtained successfully
```

### Kết quả đạt

Caddy đã tự cấp certificate Let's Encrypt cho:

```text
thientam.review360.id.vn
```

Không cần tự tạo certificate bằng tay trong trường hợp này.

→ **Đạt mới chuyển Bước 28.**

---

## Bước 28. Kiểm tra domain test mà không đổi DNS production

### Mục tiêu

Không ảnh hưởng:

```text
sauriengthientam.vn
```

trên VPS cũ.

Domain test:

```text
thientam.review360.id.vn
```

được dùng để nghiệm thu VPS mới.

Trong trường hợp domain có Cloudflare, request public có thể đi:

```text
Client
  ↓
Cloudflare
  ↓
NAT / public IP
  ↓
VPS mới
  ↓
Caddy
```

### Kiểm tra

```bash
curl -Ik https://thientam.review360.id.vn/
curl -Ik https://thientam.review360.id.vn/api-docs
```

Migration thực tế:

```text
HTTP/2 200
server: cloudflare
via: 1.1 Caddy
cf-cache-status: DYNAMIC
```

### Kiểm tra từ Windows

Có thể dùng `curl.exe --resolve` để test hostname/SNI vào một IP mà **không sửa DNS công khai**:

```powershell
curl.exe -Ik --resolve sauriengthientam.vn:443:<PUBLIC-IP-MỚI> https://sauriengthientam.vn/
curl.exe -I --resolve sauriengthientam.vn:80:<PUBLIC-IP-MỚI> http://sauriengthientam.vn/
curl.exe -Ik --resolve sauriengthientam.vn:443:<PUBLIC-IP-MỚI> https://sauriengthientam.vn/api-docs
```

Trong migration thực tế, IP NAT/public dùng để test từ bên ngoài được xác định là `27.74.254.52`. IP mà VPS báo qua `ifconfig.me` từng là `14.161.10.44`; đây có thể là khác biệt giữa IP outbound và IP inbound/NAT. Không suy đoán hai IP là một; luôn xác định bằng NAT/network provider.

### Kết quả đạt

Khách có thể truy cập domain test, test HTTPS, API Docs và frontend mà production domain cũ chưa cần đổi.

→ **Đạt mới chuyển Bước 29.**

---

# PHẦN J — NGHIỆM THU TRƯỚC CUTOVER

## Bước 29. Checklist nghiệm thu với khách

Khách kiểm tra tối thiểu:

### Authentication

- [ ] Đăng nhập
- [ ] Đăng xuất
- [ ] Session/cookie/token
- [ ] Phân quyền

### Database

- [ ] Danh sách dữ liệu
- [ ] Chi tiết dữ liệu
- [ ] Tạo mới
- [ ] Cập nhật
- [ ] Xóa nếu có quyền
- [ ] Tìm kiếm/lọc

### Media / MinIO

- [ ] Hiển thị ảnh cũ
- [ ] Upload ảnh
- [ ] Upload file
- [ ] Download file
- [ ] Mở lại file sau upload
- [ ] Kiểm tra đường dẫn `/media/...`

### API

- [ ] `/api-docs`
- [ ] API đăng nhập
- [ ] API nghiệp vụ
- [ ] GraphQL query/mutation thực tế

### Frontend

- [ ] Trang chủ
- [ ] Menu
- [ ] Các route chính
- [ ] Không có lỗi JavaScript nghiêm trọng
- [ ] Network request tới API đúng domain test

### Caddy/HTTPS

- [ ] HTTP 80 redirect HTTPS
- [ ] HTTPS 443 = 200
- [ ] Certificate hợp lệ
- [ ] Không có lỗi TLS

### Kết quả đạt

Khách xác nhận hệ thống mới đáp ứng nghiệp vụ.

→ **Chỉ khi khách xác nhận OK mới chuyển Final Cutover.**

---

# PHẦN K — FINAL CUTOVER: LẤY DỮ LIỆU MỚI NHẤT

## Bước 30. Nguyên tắc Final Cutover

Trong thời gian khách test:

```text
VPS CŨ = production thật = source of truth
VPS MỚI = môi trường nghiệm thu
```

Không lấy dữ liệu từ VPS mới quay ngược về VPS cũ.

Khi khách xác nhận OK:

```text
VPS CŨ
  ↓
Final Backup
  ↓
VPS MỚI
  ↓
Final Restore
  ↓
Final Verify
  ↓
DNS Cutover
```

Điều này bảo đảm dữ liệu phát sinh trong thời gian khách test trên hệ thống cũ không bị mất.

---

## Bước 31. Khóa ghi hệ thống cũ

### Thực hiện

Tại thời điểm cutover, cần ngăn phát sinh dữ liệu mới trong lúc backup.

Ví dụ nếu backend của VPS cũ là container Compose:

```bash
cd /opt/agribase/thientam
docker compose stop backend
```

Nếu cần maintenance page thì thực hiện theo cơ chế ứng dụng hiện có.

### Kiểm tra

```bash
docker compose ps
```

Xác nhận backend không còn xử lý request ghi dữ liệu/upload.

### Kết quả đạt

Không còn transaction/file upload mới trong lúc final backup.

→ **Đạt mới chuyển Bước 32.**

---

## Bước 32. Final Backup PostgreSQL

### Thực hiện trên VPS cũ

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
```

### Kiểm tra

```bash
gzip -t /backup/thientam-final/database/thientam_db_final.sql.gz
ls -lh /backup/thientam-final/database/thientam_db_final.sql.gz
```

### Kết quả đạt

SQL final hợp lệ và là snapshot cuối cùng của production.

→ **Đạt mới chuyển Bước 33.**

---

## Bước 33. Final Backup MinIO

### Thực hiện

Backend đã stop để không còn upload mới.

```bash
mkdir -p /backup/thientam-final/volumes

tar -czf \
/backup/thientam-final/volumes/thientam_miniodata_final.tar.gz \
-C /var/lib/docker/volumes/thientam_miniodata/_data .
```

### Kiểm tra

```bash
tar -tzf \
/backup/thientam-final/volumes/thientam_miniodata_final.tar.gz \
| head -30

ls -lh \
/backup/thientam-final/volumes/thientam_miniodata_final.tar.gz
```

### Kết quả đạt

Archive chứa đầy đủ `.minio.sys` và bucket `thientam-media` cùng dữ liệu upload/image.

→ **Đạt mới chuyển Bước 34.**

---

## Bước 34. Chuyển final backup sang VPS mới

### Thực hiện

Nếu SSH trực tiếp:

```bash
scp \
/backup/thientam-final/database/thientam_db_final.sql.gz \
root@<VPS-MỚI>:/opt/agribase/thientam-backup/database/

scp \
/backup/thientam-final/volumes/thientam_miniodata_final.tar.gz \
root@<VPS-MỚI>:/opt/agribase/thientam-backup/volumes/
```

Nếu không có SSH trực tiếp, tải file về máy quản trị rồi upload sang VPS mới.

### Kiểm tra

```bash
ls -lh /opt/agribase/thientam-backup/database/
ls -lh /opt/agribase/thientam-backup/volumes/
```

### Kết quả đạt

Hai final backup đã có mặt trên VPS mới.

→ **Đạt mới chuyển Bước 35.**

---

## Bước 35. Restore Final PostgreSQL trên VPS mới

### Thực hiện

Dừng backend/frontend:

```bash
cd /opt/agribase/thientam
docker compose stop backend frontend
```

Terminate DB connections:

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d postgres \
-c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='thientam_db' AND pid <> pg_backend_pid();"
```

Drop/recreate DB:

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d postgres \
-c "DROP DATABASE thientam_db;"

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

### Kiểm tra

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d thientam_db \
-c "SELECT PostGIS_Version();"

docker exec thientam-db-1 \
psql -U thientam_user -d thientam_db \
-c "\dt public.*"
```

So sánh các count quan trọng với VPS cũ.

### Kết quả đạt

Database mới đúng dữ liệu tại thời điểm final cutover.

→ **Đạt mới chuyển Bước 36.**

---

## Bước 36. Restore Final MinIO

### Thực hiện

Dừng MinIO:

```bash
docker compose stop minio
```

Xóa dữ liệu cũ trên volume mới **chỉ sau khi đã xác nhận final backup hợp lệ**:

```bash
rm -rf /var/lib/docker/volumes/thientam_miniodata/_data/*
```

Restore:

```bash
tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_miniodata_final.tar.gz \
-C /var/lib/docker/volumes/thientam_miniodata/_data
```

### Kiểm tra

```bash
du -sh /var/lib/docker/volumes/thientam_miniodata/_data
find /var/lib/docker/volumes/thientam_miniodata/_data -maxdepth 2 -type d
```

### Kết quả đạt

MinIO mới chứa dữ liệu cuối cùng từ VPS cũ.

→ **Đạt mới chuyển Bước 37.**

---

## Bước 37. Khởi động lại toàn bộ stack

### Thực hiện

```bash
cd /opt/agribase/thientam
docker compose up -d
```

### Kiểm tra

```bash
docker compose ps
```

Phải có:

```text
backend   Up
frontend  Up
db        Up
minio     Up
caddy     Up
```

Kiểm tra logs:

```bash
docker compose logs --tail=100 backend
docker compose logs --tail=100 frontend
docker compose logs --tail=100 minio
docker compose logs --tail=100 caddy
```

### Kết quả đạt

Không có crash loop và tất cả container `Up`.

→ **Đạt mới chuyển Bước 38.**

---

# PHẦN L — FINAL VERIFY TRƯỚC KHI ĐỔI DNS

## Bước 38. Kiểm tra backend

```bash
docker exec thientam-backend-1 \
node -e "fetch('http://127.0.0.1:3000/health').then(async r=>{console.log('HTTP',r.status);console.log(await r.text())}).catch(e=>{console.error(e);process.exit(1)})"
```

Kỳ vọng:

```text
HTTP 200
```

---

## Bước 39. Kiểm tra frontend

```bash
docker exec thientam-frontend-1 \
node -e "fetch('http://127.0.0.1:4321/').then(async r=>{console.log('HTTP',r.status)})"
```

Kỳ vọng:

```text
HTTP 200
```

---

## Bước 40. Kiểm tra API Docs bằng domain test

```bash
curl -Ik https://thientam.review360.id.vn/api-docs
```

Kỳ vọng:

```text
HTTP/2 200
```

---

## Bước 41. Kiểm tra dữ liệu mới nhất

Trên domain test, kiểm tra:

- record mới nhất trên VPS cũ có xuất hiện trên VPS mới;
- record vừa tạo trước khi khóa ghi có tồn tại;
- ảnh/file mới nhất có tồn tại;
- download được;
- API trả đúng dữ liệu;
- frontend hiển thị đúng.

### Kết quả đạt

Dữ liệu VPS mới khớp dữ liệu final của VPS cũ.

→ **Đạt mới chuyển Bước 42.**

---

# PHẦN M — DNS CUTOVER PRODUCTION

## Bước 42. Đổi DNS production

### Trước cutover

```text
sauriengthientam.vn
        ↓
VPS CŨ
```

### Sau cutover

```text
sauriengthientam.vn
        ↓
Public/NAT IP mới
        ↓
VPS MỚI
        ↓
Caddy
```

Không đổi DNS trước khi final restore và final verification hoàn tất.

### Kiểm tra

Sau khi DNS cập nhật:

```bash
dig +short sauriengthientam.vn
```

Từ máy bên ngoài:

```powershell
curl.exe -Ik https://sauriengthientam.vn/
curl.exe -Ik https://sauriengthientam.vn/api-docs
```

HTTP:

```powershell
curl.exe -I http://sauriengthientam.vn/
```

Kỳ vọng:

```text
HTTP -> 308/redirect -> HTTPS
HTTPS -> 200
API Docs -> 200
```

### Kết quả đạt

Production domain phục vụ từ VPS mới.

→ **Đạt mới chuyển Bước 43.**

---

# PHẦN N — ROLLBACK

## Bước 43. Chuẩn bị rollback

Không xóa VPS cũ ngay sau cutover.

Giữ VPS cũ ở trạng thái có thể khởi động lại trong một khoảng thời gian đủ để xác nhận production ổn định.

Nếu phát hiện lỗi nghiêm trọng:

```text
Production DNS
      ↓
quay lại VPS cũ
```

### Điều kiện rollback

- Lỗi database nghiêm trọng.
- Mất file/media.
- API nghiệp vụ lỗi.
- Login không hoạt động.
- Dữ liệu không khớp.
- Caddy/SSL lỗi.

### Kết quả đạt

Có phương án quay lại hệ thống cũ mà không mất source/data.

→ **Chỉ kết thúc migration khi production ổn định.**

---

# PHẦN O — CÁC VẤN ĐỀ ĐÃ GẶP VÀ CÁCH XỬ LÝ

## 1. `/health` domain trả 404

Không phải backend chết.

Lý do:

```caddyfile
handle /api* { reverse_proxy backend:3000 }
handle /graphql* { reverse_proxy backend:3000 }
handle { reverse_proxy frontend:4321 }
```

`/health` không match `/api*`, nên đi frontend.

Backend health phải test trong container.

---

## 2. Backend không có curl/wget

Không chạy:

```bash
docker exec thientam-backend-1 curl ...
docker exec thientam-backend-1 wget ...
```

Thay bằng Node 24 có sẵn:

```bash
docker exec thientam-backend-1 \
node -e "fetch('http://127.0.0.1:3000/health').then(async r=>{console.log(r.status);console.log(await r.text())})"
```

---

## 3. Caddy không publish port 80

Ban đầu chỉ có 443.

Đã bổ sung:

```yaml
- "80:80"
```

Sau đó HTTP redirect hoạt động.

---

## 4. `curl https://127.0.0.1` lỗi TLS

Không test certificate bằng IP loopback.

Dùng hostname/SNI:

```bash
curl -Ik --resolve sauriengthientam.vn:443:127.0.0.1 \
https://sauriengthientam.vn/
```

---

## 5. MinIO root trả 403

```text
HTTP/2 403
AccessDenied
```

Đây không tự động có nghĩa MinIO hỏng. Root anonymous access có thể bị từ chối.

Test bằng object thật hoặc Console.

---

## 6. GraphQL GET rỗng trả 400

Không dùng GET `/graphql` rỗng làm bài test nghiệp vụ. Dùng query/mutation thật.

---

## 7. `docker compose config` cảnh báo volume đã tồn tại

Do volume được tạo thủ công:

```bash
docker volume create thientam_pgdata
```

Compose cảnh báo chưa được tạo bởi Compose. Container vẫn chạy đúng.

Sau migration nên chuẩn hóa `external: true` hoặc để Compose quản lý volume nhất quán.

---

## 8. Project archive có thêm một cấp `thientam/`

Archive có:

```text
thientam/
    backend-src/
    frontend-src/
    docker-compose.yml
```

Do đó cần copy nội dung một cấp lên `/opt/agribase/thientam/`.

---

## 9. Caddy tự cấp certificate

Đúng. Caddy đã tự dùng Let's Encrypt và log:

```text
certificate obtained successfully
```

Nếu TLS-ALPN-01 không thành công nhưng HTTP-01 thành công thì vẫn có thể cấp certificate bình thường miễn challenge HTTP-01 đi được tới Caddy.

---

# PHẦN P — TỐI ƯU BACKUP DUNG LƯỢNG

## Hiện trạng thực tế

Backup lần migration:

| Thành phần | Kích thước gần đúng |
|---|---:|
| Docker images | 811M |
| Project | 26M |
| PostgreSQL dump | 1.1M |
| MinIO volume | 13M |
| Caddy data | 5.8K |
| Caddy config | 530B |
| **Tổng** | **~850M** |

VPS cũ lúc backup:

```text
48G disk
41G used
4.7G available
90% used
```

Vì vậy chiến lược backup phải tránh tạo nhiều bản sao.

## Nguyên tắc tối ưu

### 1. Database

Luôn dump SQL rồi gzip:

```bash
pg_dump ... | gzip -1 > thientam_db.sql.gz
```

### 2. Volume

Dùng `tar.gz` trực tiếp.

Không cần zip thêm.

### 3. Docker images

Dùng một `docker save` cho toàn bộ image rồi gzip một lần:

```bash
docker save image1 image2 ... | gzip -1 > images.tar.gz
```

### 4. Final cutover

Không cần save lại Docker images nếu application image không thay đổi.

Chỉ final backup:

```text
PostgreSQL
MinIO
```

### 5. Không giữ bản sao dư thừa trên VPS cũ

Sau khi đã chuyển backup an toàn sang VPS mới và xác nhận checksum, có thể xóa bản sao trung gian không cần thiết để giải phóng disk. Không xóa bản backup duy nhất trước khi migration được nghiệm thu.

---

# PHẦN Q — BẢNG CHECKLIST THỰC HIỆN 1 LẦN

## A. VPS cũ

- [ ] OS kiểm tra
- [ ] Disk kiểm tra
- [ ] RAM kiểm tra
- [ ] Docker kiểm tra
- [ ] Compose kiểm tra
- [ ] Containers kiểm tra
- [ ] Images kiểm tra
- [ ] Volumes kiểm tra
- [ ] Project path kiểm tra
- [ ] Domain/IP kiểm tra

## B. Backup

- [ ] Docker images backup
- [ ] Project backup
- [ ] PostgreSQL backup
- [ ] MinIO backup
- [ ] Caddy data backup
- [ ] Caddy config backup
- [ ] SHA256SUMS tạo
- [ ] gzip/tar test
- [ ] Tổng dung lượng kiểm tra

## C. Chuyển backup

- [ ] Đóng gói một file
- [ ] Upload/download hoàn tất
- [ ] `tar -tf` OK
- [ ] `sha256sum -c` = OK toàn bộ

## D. VPS mới

- [ ] Ubuntu 24.04 OK
- [ ] aaPanel OK
- [ ] Docker OK
- [ ] Compose OK
- [ ] 80/443 không bị aaPanel Nginx chiếm
- [ ] Project restore
- [ ] Images load
- [ ] Volumes create
- [ ] MinIO restore
- [ ] Caddy restore
- [ ] Compose validate

## E. Service

- [ ] DB Up
- [ ] PostgreSQL OK
- [ ] PostGIS OK
- [ ] Database data OK
- [ ] MinIO Up
- [ ] MinIO data OK
- [ ] Backend Up
- [ ] Backend health 200
- [ ] Frontend Up
- [ ] Frontend 200
- [ ] Caddy Up
- [ ] HTTP 80 redirect
- [ ] HTTPS 443 200

## F. Domain test

- [ ] DNS/domain test hoạt động
- [ ] Let's Encrypt certificate OK
- [ ] Frontend 200
- [ ] API Docs 200
- [ ] API nghiệp vụ OK
- [ ] GraphQL query thật OK
- [ ] Upload OK
- [ ] Download OK
- [ ] Media OK
- [ ] Khách xác nhận OK

## G. Final Cutover

- [ ] Khóa ghi VPS cũ
- [ ] Final DB backup
- [ ] Final MinIO backup
- [ ] Kiểm tra final backup
- [ ] Chuyển final backup
- [ ] Restore final DB
- [ ] Restore final MinIO
- [ ] Start toàn bộ stack
- [ ] Final verify
- [ ] Đổi DNS production
- [ ] Verify production
- [ ] Giữ VPS cũ để rollback

---

# PHẦN R — MẪU TRÌNH TỰ LỆNH FINAL CUTOVER

> Đây là phần rút gọn để dùng trực tiếp trong ngày cutover. Phải thực hiện sau khi khách xác nhận nghiệm thu domain test.

## 1. VPS cũ — khóa ghi

```bash
cd /opt/agribase/thientam
docker compose stop backend
```

## 2. VPS cũ — final DB

```bash
mkdir -p /backup/thientam-final/database

docker exec thientam-db-1 \
  pg_dump -U thientam_user -d thientam_db \
  --format=plain --no-owner --no-privileges \
  > /backup/thientam-final/database/thientam_db_final.sql

gzip -1 /backup/thientam-final/database/thientam_db_final.sql

gzip -t /backup/thientam-final/database/thientam_db_final.sql.gz
```

## 3. VPS cũ — final MinIO

```bash
mkdir -p /backup/thientam-final/volumes

tar -czf \
/backup/thientam-final/volumes/thientam_miniodata_final.tar.gz \
-C /var/lib/docker/volumes/thientam_miniodata/_data .

tar -tzf \
/backup/thientam-final/volumes/thientam_miniodata_final.tar.gz \
| head
```

## 4. Chuyển sang VPS mới

```bash
scp /backup/thientam-final/database/thientam_db_final.sql.gz \
root@<VPS-MOI>:/opt/agribase/thientam-backup/database/

scp /backup/thientam-final/volumes/thientam_miniodata_final.tar.gz \
root@<VPS-MOI>:/opt/agribase/thientam-backup/volumes/
```

## 5. VPS mới — stop application

```bash
cd /opt/agribase/thientam
docker compose stop backend frontend minio
```

## 6. VPS mới — restore DB

```bash
docker exec thientam-db-1 \
psql -U thientam_user -d postgres \
-c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname='thientam_db' AND pid <> pg_backend_pid();"

docker exec thientam-db-1 \
psql -U thientam_user -d postgres \
-c "DROP DATABASE thientam_db;"

docker exec thientam-db-1 \
psql -U thientam_user -d postgres \
-c "CREATE DATABASE thientam_db OWNER thientam_user;"

gunzip -c \
/opt/agribase/thientam-backup/database/thientam_db_final.sql.gz \
| docker exec -i thientam-db-1 \
psql -U thientam_user -d thientam_db
```

## 7. VPS mới — restore MinIO

```bash
rm -rf /var/lib/docker/volumes/thientam_miniodata/_data/*

tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_miniodata_final.tar.gz \
-C /var/lib/docker/volumes/thientam_miniodata/_data
```

## 8. VPS mới — start

```bash
docker compose up -d
```

## 9. VPS mới — kiểm tra

```bash
docker compose ps

docker exec thientam-backend-1 \
node -e "fetch('http://127.0.0.1:3000/health').then(async r=>{console.log(r.status);console.log(await r.text())})"

curl -Ik https://thientam.review360.id.vn/
curl -Ik https://thientam.review360.id.vn/api-docs
```

## 10. Đổi DNS production

Chỉ thực hiện khi tất cả kiểm tra trên đều đạt.

Sau đó:

```bash
dig +short sauriengthientam.vn
```

và kiểm tra từ máy bên ngoài:

```powershell
curl.exe -Ik https://sauriengthientam.vn/
curl.exe -Ik https://sauriengthientam.vn/api-docs
curl.exe -I http://sauriengthientam.vn/
```

---

# PHẦN S — KHẢO SÁT BẢO MẬT SAU MIGRATION

Trong migration thực tế, `docker compose ps` cho thấy DB và Caddy/MinIO có các port publish ra host:

```text
5432 -> PostgreSQL
9000 -> MinIO API qua Caddy
9001 -> MinIO Console qua Caddy
80 -> Caddy
443 -> Caddy
```

Trước production chính thức, cần rà soát firewall và exposure. PostgreSQL **không nên mở public Internet** nếu backend chỉ cần truy cập qua Docker network. MinIO API/Console cũng nên cân nhắc chỉ expose qua Caddy/domain hoặc giới hạn firewall theo nhu cầu.

Kiểm tra:

```bash
ss -lntp
ufw status
```

Trong lần migration thực tế UFW đang active và đã allow 80/443 cùng nhiều port quản trị. Không tự động xóa rule hiện có trong lúc migration; cần lập kế hoạch hardening riêng để không khóa mất SSH/aaPanel.

### Nguyên tắc

```text
Internet
   │
   ├── 80  → Caddy
   └── 443 → Caddy
             │
             ├── Frontend :4321
             ├── Backend  :3000
             └── MinIO    :9000

PostgreSQL :5432
→ ưu tiên chỉ Docker network / localhost / trusted network
```

---

# PHẦN T — TRẠNG THÁI ĐÃ XÁC NHẬN TRONG LẦN MIGRATION NÀY

Các kết quả đã thực sự chạy thành công:

- [x] Backup images ≈ 811M.
- [x] Project backup ≈ 26M.
- [x] PostgreSQL dump ≈ 1.1M.
- [x] MinIO volume ≈ 13M.
- [x] Caddy data/config backup.
- [x] SHA256SUMS.
- [x] Gộp thành `/opt/agribase/thientam-backup.tar` ≈ 850M.
- [x] Upload sang VPS mới.
- [x] `sha256sum -c SHA256SUMS` = tất cả `OK`.
- [x] Docker images load thành công.
- [x] Project restore thành công.
- [x] Compose services xác định đủ 5 service.
- [x] Compose config validate thành công.
- [x] Docker volumes tạo thành công.
- [x] MinIO volume restore thành công.
- [x] Caddy data/config restore thành công.
- [x] PostgreSQL 16.4 chạy.
- [x] PostGIS 3.4 chạy.
- [x] MinIO chạy.
- [x] MinIO Console trả 200.
- [x] Backend port 3000 mở.
- [x] Backend `/health` nội bộ trả 200.
- [x] Frontend port 4321 chạy.
- [x] Frontend nội bộ trả 200.
- [x] Caddy chạy.
- [x] Port 80 được publish.
- [x] Port 443 được publish.
- [x] HTTP → HTTPS redirect hoạt động.
- [x] HTTPS hostname test trả 200.
- [x] `/api-docs` qua Caddy trả 200.
- [x] Domain test `thientam.review360.id.vn` hoạt động.
- [x] Caddy tự cấp Let's Encrypt certificate cho domain test.
- [x] Domain test có thể dùng để khách nghiệm thu mà không đổi production DNS.

---

# PHẦN U — QUY TẮC VẬN HÀNH SAU MIGRATION

1. **Không xóa VPS cũ ngay sau cutover.**
2. Giữ backup cuối cùng ít nhất trong thời gian nghiệm thu production.
3. Không commit `.env`, password, secret vào Git.
4. Khi application image thay đổi, backup lại Docker image nếu cần rollback image.
5. Khi chỉ có dữ liệu thay đổi, không cần `docker save` lại images.
6. PostgreSQL và MinIO là hai thành phần phải ưu tiên backup cuối cùng.
7. Luôn kiểm tra checksum khi chuyển backup giữa máy.
8. Luôn `docker compose config -q` trước khi `up -d` sau khi sửa Compose.
9. Luôn validate Caddy trước reload.
10. Luôn test bằng hostname/SNI khi kiểm tra HTTPS.
11. Không dùng `/health` qua domain nếu Caddy chưa route `/health` vào backend; dùng health nội bộ hoặc một endpoint `/api/...` phù hợp.
12. Trước production phải kiểm tra các port public và firewall.
13. Khi khách test trong nhiều ngày, VPS cũ vẫn là source of truth.
14. Khi khách xác nhận OK, thực hiện Final Cutover Backup ngay trước khi đổi DNS.
15. Chỉ đổi DNS sau khi Final Restore + Final Verify đạt.

---

# KẾT LUẬN

Quy trình migration này được xây dựng từ toàn bộ quá trình thực tế đã thực hiện với hệ thống Thiên Tâm/AgriBase: từ backup Docker images, project, PostgreSQL, MinIO và Caddy; tạo SHA-256; đóng gói một file backup khoảng 850M; chuyển sang VPS mới; kiểm tra checksum; load images; restore project/volumes; khởi động từng container; xử lý vấn đề port 80/443; kiểm tra backend/frontend/MinIO/Caddy; cấu hình domain test; xác nhận Caddy tự cấp Let's Encrypt; đến chiến lược Final Cutover để lấy **dữ liệu mới nhất từ VPS cũ** trước khi đổi production DNS.

Mục tiêu của quy trình là có thể lặp lại migration một cách có kiểm soát, không phụ thuộc vào việc nhớ lại các lệnh đã chạy trong phiên làm việc trước. Mỗi bước đều có điều kiện kiểm tra rõ ràng và chỉ chuyển sang bước tiếp theo khi kết quả đạt.

**Nguyên tắc cuối cùng:**

```text
BACKUP
  ↓
VERIFY CHECKSUM
  ↓
RESTORE INFRASTRUCTURE
  ↓
RESTORE DATA
  ↓
TEST INTERNAL
  ↓
TEST DOMAIN
  ↓
CUSTOMER ACCEPTANCE
  ↓
FINAL BACKUP FROM OLD VPS
  ↓
FINAL RESTORE TO NEW VPS
  ↓
FINAL VERIFY
  ↓
DNS CUTOVER
  ↓
MONITOR + KEEP ROLLBACK
```

---

## Thông tin tài liệu

- **Tên tài liệu:** Quy trình Migration Hệ thống từ VPS cũ sang VPS mới - Sử dụng Docker
- **Hệ thống:** Thiên Tâm / AgriBase
- **Production domain:** `sauriengthientam.vn`
- **Test domain:** `thientam.review360.id.vn`
- **Project path:** `/opt/agribase/thientam`
- **Backup path:** `/backup/thientam`
- **Docker images:** backend, frontend, PostGIS, MinIO, Caddy
- **Database:** PostgreSQL 16 + PostGIS 3.4
- **Object storage:** MinIO
- **Reverse proxy/TLS:** Caddy
- **VPS mới:** Ubuntu 24.04.4 LTS + aaPanel + Docker
- **Tài liệu được tạo từ quá trình migration thực tế ngày 15–16/08/2026**
