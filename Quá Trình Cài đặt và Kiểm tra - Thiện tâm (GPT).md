# Quá Trình Cài đặt và Kiểm tra - Thiện tâm (GPT)

> **Mục đích:** Lưu lại quá trình trao đổi và các bước thao tác đã thực hiện khi cài đặt, khôi phục, kiểm tra và đưa hệ thống Thiện tâm lên VPS mới sử dụng Ubuntu 24.04 + aaPanel + Docker Compose + Caddy.
>
> **Phạm vi:** Tài liệu này ghi lại chuỗi thao tác trước khi bắt đầu bộ tài liệu vận hành/backup chính. Nội dung ưu tiên giữ nguyên câu lệnh, kết quả kiểm tra, các lỗi/cảnh báo và quyết định xử lý đã xuất hiện trong cuộc trao đổi.
>
> **Lưu ý bảo mật:** Các secret/password thực tế xuất hiện trong log trao đổi không được lặp lại trong tài liệu Git này. Khi cần kiểm tra trên máy thật, lấy giá trị hiện tại từ `.env`, `backend.env` và cấu hình Docker tương ứng.

---

## 1. Bắt đầu trên VPS mới

VPS mới sử dụng:

- Ubuntu 24.04.4 LTS (Noble Numbat)
- Kernel `6.8.0-137-generic`
- CPU architecture: `x86_64`
- Root filesystem khoảng 490 GB
- RAM khoảng 7.8 GiB
- Swap 4 GiB
- Docker 29.7.2
- Docker Compose v5.4.0
- aaPanel đã được cài đặt

Kiểm tra hệ điều hành:

```bash
cat /etc/os-release
```

Kết quả:

```text
PRETTY_NAME="Ubuntu 24.04.4 LTS"
NAME="Ubuntu"
VERSION_ID="24.04"
VERSION="24.04.4 LTS (Noble Numbat)"
VERSION_CODENAME=noble
```

Kiểm tra kernel:

```bash
uname -a
```

Kết quả cho thấy Ubuntu 24.04, kernel 6.8 và x86_64.

Kiểm tra dung lượng:

```bash
df -h /
```

Kết quả:

```text
/dev/mapper/ubuntu--vg-ubuntu--lv  490G  9.0G  456G   2% /
```

Kiểm tra RAM/swap:

```bash
free -h
```

Kết quả:

```text
Mem:           7.8Gi       962Mi       3.5Gi       2.1Mi       3.6Gi       6.8Gi
Swap:          4.0Gi          0B       4.0Gi
```

Kiểm tra Docker:

```bash
docker --version
docker compose version
```

Kết quả:

```text
Docker version 29.7.2, build a7dcaa6
Docker Compose version v5.4.0
```

---

## 2. Kiểm tra aaPanel và các port hiện có

Lệnh:

```bash
bt --version
```

Kết quả:

```text
Unsupported command
===============================================
Cancelled!
```

Do đó dùng:

```bash
bt status
```

Kết quả:

```text
Bt-Panel (pid 29206) already running
Bt-Task (pid 29200) already running
```

Kiểm tra port:

```bash
ss -lntp
```

Ban đầu thấy các port liên quan aaPanel/SSH và webserver. SSH đang dùng port `10012`, aaPanel đang có port `888`, và một webserver của aaPanel đang dùng port `2891`.

Kiểm tra riêng các port thông dụng:

```bash
ss -lntp | grep -E ':22 |:80 |:443 |:7800 |:8080 |:8888 '
```

Lúc đầu chỉ thấy port 80 được nginx của aaPanel listen.

Sau đó kiểm tra nginx systemd:

```bash
systemctl status nginx --no-pager
```

Kết quả:

```text
○ nginx.service - LSB: starts the nginx web server
     Loaded: loaded (/etc/init.d/nginx; generated)
     Active: inactive (dead)
```

Kiểm tra enabled:

```bash
systemctl is-enabled nginx
```

Kết quả:

```text
generated
```

Điểm quan trọng: nginx systemd không phải thành phần cuối cùng phục vụ hệ thống Thiện tâm; sau khi Docker/Caddy được dựng, Caddy sẽ đảm nhiệm reverse proxy cho hệ thống.

---

## 3. Xác nhận VPS mới chưa có container/volume cũ

```bash
docker ps
```

Kết quả:

```text
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS   PORTS   NAMES
```

Không có container đang chạy.

Kiểm tra volume:

```bash
docker volume ls
```

Kết quả ban đầu không có volume liên quan Thiện tâm.

---

## 4. Chuẩn bị thư mục restore

Tạo thư mục:

```bash
mkdir -p /opt/agribase
ls -ld /opt/agribase
```

Kết quả:

```text
drwxr-xr-x 2 root root 4096 Aug 16 09:42 /opt/agribase
```

File backup đã được đưa lên VPS:

```bash
ls -lh /opt/agribase/thientam-backup.tar
```

Kết quả:

```text
-rw-r--r-- 1 root root 850M Aug 16 09:46 /opt/agribase/thientam-backup.tar
```

Kiểm tra cấu trúc archive:

```bash
tar -tf /opt/agribase/thientam-backup.tar | head -30
```

Archive gồm:

```text
thientam/
thientam/images/
thientam/images/thientam-images.tar.gz
thientam/project/
thientam/project/thientam-project.tar.gz
thientam/database/
thientam/database/thientam_db.sql.gz
thientam/volumes/
thientam/volumes/thientam_caddydata.tar.gz
thientam/volumes/thientam_miniodata.tar.gz
thientam/volumes/thientam_caddyconfig.tar.gz
thientam/SHA256SUMS
```

Đây là bộ backup có 3 nhóm dữ liệu lớn:

1. Docker images.
2. Project/configuration.
3. Database và Docker volumes.

---

## 5. Giải nén bộ backup và kiểm tra checksum

Sau khi giải nén bộ backup vào `/opt/agribase/thientam`, chuyển vào thư mục:

```bash
cd /opt/agribase/thientam
```

Kiểm tra SHA256:

```bash
sha256sum -c SHA256SUMS
```

Kết quả:

```text
images/thientam-images.tar.gz: OK
project/thientam-project.tar.gz: OK
database/thientam_db.sql.gz: OK
volumes/thientam_miniodata.tar.gz: OK
volumes/thientam_caddydata.tar.gz: OK
volumes/thientam_caddyconfig.tar.gz: OK
```

**Checkpoint:** tất cả file backup đều `OK`. Chỉ sau checkpoint này mới tiếp tục restore.

---

## 6. Restore Docker images

Load toàn bộ image archive:

```bash
gunzip -c images/thientam-images.tar.gz | docker load
```

Kết quả:

```text
Loaded image: caddy:2
Loaded image: thientam-backend:latest
Loaded image: thientam-frontend:latest
Loaded image: postgis/postgis:16-3.4
Loaded image: quay.io/minio/minio:latest
```

Kiểm tra:

```bash
docker images
```

Sau đó kiểm tra gọn:

```bash
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

Kết quả:

```text
REPOSITORY            TAG       SIZE
thientam-backend      latest    1.97GB
thientam-frontend     latest    1.7GB
caddy                 2         128MB
quay.io/minio/minio   latest    355MB
postgis/postgis       16-3.4    1.26GB
```

**Checkpoint:** đủ 5 image cần thiết đã được load.

---

## 7. Restore project

Ban đầu thư mục project được giải nén thành một cấp thư mục lồng:

```bash
cd /opt/agribase
mv thientam thientam-backup
mkdir -p /opt/agribase/thientam
```

Giải nén project:

```bash
tar -xzf /opt/agribase/thientam-backup/project/thientam-project.tar.gz \
  -C /opt/agribase/thientam
```

Kết quả ban đầu:

```text
/opt/agribase/thientam/thientam/
```

Kiểm tra:

```bash
ls -lah /opt/agribase/thientam/thientam
```

Thấy các thành phần:

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

Kiểm tra archive project:

```bash
tar -tzf /opt/agribase/thientam-backup/project/thientam-project.tar.gz | head -30
```

Archive bắt đầu bằng:

```text
thientam/
thientam/backend-src/
thientam/backend-src/.env.example
thientam/backend-src/vercel.json
thientam/backend-src/docs/
...
```

Do cấu trúc archive chứa thư mục gốc `thientam/`, thực hiện đưa nội dung lên đúng vị trí:

```bash
cp -a /opt/agribase/thientam/thientam/. /opt/agribase/thientam/
rm -rf /opt/agribase/thientam/thientam
```

Kiểm tra lại:

```bash
ls -lah /opt/agribase/thientam
```

Kết quả cuối cùng có đúng:

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

**Checkpoint:** project đã nằm đúng `/opt/agribase/thientam`.

---

## 8. Kiểm tra Docker Compose trước khi chạy

```bash
cd /opt/agribase/thientam
```

Kiểm tra service:

```bash
docker compose config --services
```

Kết quả:

```text
db
minio
backend
frontend
caddy
```

Kiểm tra volume khai báo:

```bash
docker compose config --volumes
```

Kết quả:

```text
caddyconfig
pgdata
miniodata
caddydata
```

Validate Compose:

```bash
docker compose config -q
```

Không có output, nghĩa là cấu hình Compose hợp lệ.

**Checkpoint:** chỉ tiếp tục khi `docker compose config -q` không báo lỗi.

---

## 9. Tạo Docker volumes

Tạo volume:

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

Kết quả:

```text
local     thientam_caddyconfig
local     thientam_caddydata
local     thientam_miniodata
local     thientam_pgdata
```

Compose sau đó cảnh báo rằng các volume đã tồn tại nhưng không được Docker Compose tạo ra. Đây là cảnh báo do volume được tạo thủ công để phục vụ restore. Không phải lỗi dữ liệu.

---

## 10. Restore MinIO data

Restore MinIO:

```bash
tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_miniodata.tar.gz \
-C /var/lib/docker/volumes/thientam_miniodata/_data
```

Kiểm tra dung lượng:

```bash
du -sh /var/lib/docker/volumes/thientam_miniodata/_data
```

Kết quả:

```text
14M
```

Kiểm tra cấu trúc:

```bash
find /var/lib/docker/volumes/thientam_miniodata/_data -maxdepth 2 -type d | head -20
```

Kết quả có:

```text
.../_data
.../_data/thientam-media
.../_data/thientam-media/uploads
.../_data/thientam-media/image
.../_data/.minio.sys
.../_data/.minio.sys/multipart
.../_data/.minio.sys/buckets
.../_data/.minio.sys/tmp
.../_data/.minio.sys/config
.../_data/.minio.sys/pool.bin
```

**Checkpoint:** bucket `thientam-media` và dữ liệu MinIO đã xuất hiện.

---

## 11. Restore Caddy config/data

Restore Caddy config:

```bash
tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_caddyconfig.tar.gz \
-C /var/lib/docker/volumes/thientam_caddyconfig/_data
```

Kiểm tra:

```bash
ls -lah /var/lib/docker/volumes/thientam_caddyconfig/_data
```

Có thư mục:

```text
caddy/
```

Restore Caddy data:

```bash
tar -xzf \
/opt/agribase/thientam-backup/volumes/thientam_caddydata.tar.gz \
-C /var/lib/docker/volumes/thientam_caddydata/_data
```

Kiểm tra:

```bash
ls -lah /var/lib/docker/volumes/thientam_caddydata/_data
```

Có thư mục:

```text
caddy/
```

**Checkpoint:** cả `caddyconfig` và `caddydata` đã có dữ liệu.

---

## 12. Khởi động PostgreSQL/PostGIS

Chạy DB trước:

```bash
docker compose up -d db
```

Compose cảnh báo volume `thientam_pgdata` đã tồn tại và không được tạo bởi Compose. Đây là kết quả dự kiến vì volume được tạo thủ công.

Kiểm tra:

```bash
docker compose ps
```

DB chạy:

```text
thientam-db-1   postgis/postgis:16-3.4   Up
```

Tại thời điểm này DB publish port `5432` ra host theo file Compose.

Xem log:

```bash
docker compose logs --tail=50 db
```

Log cho thấy PostgreSQL 16.4 khởi động, PostGIS được tạo và database `thientam_db` được khởi tạo cùng user của hệ thống.

Có dòng initdb:

```text
initdb: warning: enabling "trust" authentication for local connections
```

Đây là warning của quá trình initdb container; sau đó server báo:

```text
database system is ready to accept connections
```

PostGIS cũng được load vào database.

---

## 13. Kiểm tra PostgreSQL và PostGIS

Kiểm tra version PostgreSQL:

```bash
docker exec thientam-db-1 \
  psql -U thientam_user -d thientam_db \
  -c "SELECT version();"
```

Kết quả:

```text
PostgreSQL 16.4 (Debian 16.4-1.pgdg110+2)
```

Kiểm tra PostGIS:

```bash
docker exec thientam-db-1 \
  psql -U thientam_user -d thientam_db \
  -c "SELECT PostGIS_Version();"
```

Kết quả:

```text
3.4 USE_GEOS=1 USE_PROJ=1 USE_STATS=1
```

Kiểm tra bảng:

```bash
docker exec thientam-db-1 \
  psql -U thientam_user -d thientam_db \
  -c "\dt"
```

Kết quả có 37 relation/bảng liên quan public/tiger/topology, trong đó `public.spatial_ref_sys` và các bảng TIGER/topology được tạo bởi PostGIS.

**Checkpoint:** PostgreSQL 16.4 + PostGIS 3.4 hoạt động.

---

## 14. Khởi động MinIO

```bash
docker compose up -d minio
```

Kiểm tra:

```bash
docker compose ps
```

MinIO chạy.

Xem log:

```bash
docker compose logs --tail=50 minio
```

Log cho thấy:

```text
MinIO Object Storage Server
Version: RELEASE.2025-09-07T16-13-09Z
API: http://172.18.0.3:9000
WebUI: https://sauriengthientam.vn:9001
```

Kiểm tra lại volume:

```bash
du -sh /var/lib/docker/volumes/thientam_miniodata/_data
```

Kết quả:

```text
14M
```

**Checkpoint:** MinIO khởi động và nhận dữ liệu backup.

---

## 15. Kiểm tra Backend trước khi publish ra ngoài

Ban đầu thử từ VPS:

```bash
curl -i http://127.0.0.1:3000/health
```

Kết quả:

```text
curl: (7) Failed to connect to 127.0.0.1 port 3000
```

Lý do: backend chạy trong Docker container, port 3000 không publish trực tiếp ra host.

Thử trong container bằng curl:

```bash
docker exec thientam-backend-1 curl -i http://127.0.0.1:3000/health
```

Không thực hiện được vì image không cài `curl`:

```text
exec: "curl": executable file not found in $PATH
```

Thử wget:

```bash
docker exec thientam-backend-1 wget -qO- http://127.0.0.1:3000/health
```

Cũng không được vì image không có `wget`.

Dùng Node.js có sẵn trong container:

```bash
docker exec thientam-backend-1 \
node -e "fetch('http://127.0.0.1:3000/health').then(async r=>{console.log('HTTP',r.status);console.log(await r.text())}).catch(e=>{console.error(e);process.exit(1)})"
```

Kết quả:

```text
HTTP 200
{"status":"OK","environment":"production","timestamp":"2026-08-16T11:18:57.543Z","uptime":154.893422133}
```

Kiểm tra TCP port 3000 trong container:

```bash
docker exec thientam-backend-1 \
node -e "require('net').createConnection({host:'127.0.0.1',port:3000}).on('connect',()=>{console.log('PORT 3000 OPEN');process.exit(0)}).on('error',e=>{console.error('PORT 3000 ERROR:',e.message);process.exit(1)})"
```

Kết quả:

```text
PORT 3000 OPEN
```

Kiểm tra IP container:

```bash
docker inspect thientam-backend-1 \
--format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

Kết quả:

```text
172.18.0.4
```

**Checkpoint:** Backend hoạt động đúng trong Docker, health trả HTTP 200.

---

## 16. Khởi động Frontend

```bash
docker compose up -d frontend
```

Kiểm tra:

```bash
docker compose ps
```

Frontend chạy với port nội bộ `4321/tcp`.

Xem log:

```bash
docker compose logs --tail=100 frontend
```

Kết quả:

```text
[@astrojs/node] Server listening on
local: http://localhost:4321
network: http://172.18.0.5:4321
```

Kiểm tra trực tiếp trong container bằng Node:

```bash
docker exec thientam-frontend-1 \
node -e "fetch('http://127.0.0.1:4321/').then(async r=>{console.log('HTTP',r.status);console.log((await r.text()).slice(0,500))}).catch(e=>{console.error(e);process.exit(1)})"
```

Kết quả:

```text
HTTP 200
<!DOCTYPE html>...
```

Kiểm tra IP:

```bash
docker inspect thientam-frontend-1 \
--format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}'
```

Kết quả:

```text
172.18.0.5
```

**Checkpoint:** Frontend trả HTTP 200 trong container.

---

## 17. Kiểm tra Caddy trước khi sửa port 80

Khởi động Caddy:

```bash
docker compose up -d caddy
```

Lúc đầu `docker compose ps` cho thấy Caddy publish:

- 443
- 9000
- 9001

nhưng **chưa publish 80 ra host**.

Kiểm tra cấu hình Caddy:

```bash
cat /opt/agribase/thientam/Caddyfile
```

Cấu hình production ban đầu:

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

Xem log Caddy:

```bash
docker compose logs --tail=100 caddy
```

Log cho thấy Caddy tự động bật HTTPS redirect và automatic TLS. Đồng thời có cảnh báo HTTP/2/HTTP/3 trên port 80 vì port 80 chưa được publish ra host.

Kiểm tra host:

```bash
ss -lntp | grep -E ':80 |:443 '
```

Kết quả tại thời điểm đó chỉ có 443.

---

## 18. Sửa Docker Compose để publish port 80

Backup file trước khi sửa:

```bash
cp docker-compose.yml docker-compose.yml.before-port80
```

Mở file:

```bash
nano docker-compose.yml
```

Thêm mapping:

```yaml
ports:
  - "80:80"
  - "443:443"
  - "9000:9000"
  - "9001:9001"
```

Kiểm tra cấu hình:

```bash
docker compose config -q
```

Không có output => hợp lệ.

Kiểm tra phần Caddy:

```bash
docker compose config | sed -n '/caddy:/,/^[^ ]/p'
```

Kết quả xác nhận:

```text
target: 80
published: "80"
target: 443
published: "443"
```

Khởi động lại/recreate Caddy để áp dụng mapping.

Kiểm tra:

```bash
docker compose ps
```

Kết quả:

```text
thientam-caddy-1 ... 0.0.0.0:80->80/tcp, [::]:80->80/tcp,
                  0.0.0.0:443->443/tcp, [::]:443->443/tcp,
                  0.0.0.0:9000-9001->9000-9001/tcp
```

Kiểm tra host:

```bash
ss -lntp | grep -E ':80 |:443 '
```

Kết quả:

```text
0.0.0.0:443 ... docker-proxy
0.0.0.0:80  ... docker-proxy
[::]:443 ... docker-proxy
[::]:80  ... docker-proxy
```

---

## 19. Kiểm tra HTTP → HTTPS và HTTPS bằng hostname

Test trực tiếp IP localhost không kèm hostname:

```bash
curl -I http://127.0.0.1
```

Kết quả:

```text
HTTP/1.1 308 Permanent Redirect
Location: https://127.0.0.1/
Server: Caddy
```

Test HTTPS vào IP thuần:

```bash
curl -Ik https://127.0.0.1
```

Kết quả lỗi TLS:

```text
SSL ... tlsv1 alert internal error
```

Điều này không phải lỗi ứng dụng; HTTPS certificate được cấp cho hostname, không phải `127.0.0.1`.

Test đúng hostname nhưng ép phân giải về localhost:

```bash
curl -Ik --resolve sauriengthientam.vn:443:127.0.0.1 \
https://sauriengthientam.vn/
```

Kết quả:

```text
HTTP/2 200
via: 1.1 Caddy
x-robots-tag: noindex, nofollow
```

**Checkpoint:** Caddy route production hoạt động và certificate/hostname hợp lệ.

---

## 20. Kiểm tra API và GraphQL

Test `/health` qua hostname:

```bash
curl -ik --resolve sauriengthientam.vn:443:127.0.0.1 \
https://sauriengthientam.vn/health
```

Kết quả:

```text
HTTP/2 404
```

Response là trang 404 của Astro frontend.

Điều này xác nhận `/health` không phải route frontend; health thật nằm bên trong backend tại `backend:3000/health`.

Test API docs:

```bash
curl -ik --resolve sauriengthientam.vn:443:127.0.0.1 \
https://sauriengthientam.vn/api-docs
```

Kết quả:

```text
HTTP/2 200
content-type: text/html; charset=utf-8
```

Trang trả về là **AgriBase API Docs**, sử dụng Swagger UI.

Test GraphQL:

```bash
curl -ik --resolve sauriengthientam.vn:443:127.0.0.1 \
https://sauriengthientam.vn/graphql
```

Kết quả:

```text
HTTP/2 400
content-type: application/json; charset=utf-8
```

Response:

```json
{"errors":[{"message":"Internal Server Error","extensions":{"code":"INTERNAL_ERROR"}}]}
```

Đây là request GET không có GraphQL query hợp lệ; việc Caddy route được tới backend thể hiện qua response header/body backend. Trong lúc debug đã có một lần chạy `node -e` bị lỗi do câu lệnh truyền vào bị thiếu:

```text
SyntaxError: Unexpected end of input
```

Đó là lỗi câu lệnh test, không phải lỗi backend.

---

## 21. Kiểm tra MinIO API và Console qua Caddy

Test port 9000:

```bash
curl -ik --resolve sauriengthientam.vn:9000:127.0.0.1 \
https://sauriengthientam.vn:9000/
```

Kết quả:

```text
HTTP/2 403
server: MinIO
```

`403 AccessDenied` khi gọi root API mà không có authentication là phù hợp với MinIO.

Test Console:

```bash
curl -ik --resolve sauriengthientam.vn:9001:127.0.0.1 \
https://sauriengthientam.vn:9001/
```

Kết quả:

```text
HTTP/2 200
server: MinIO Console
```

Console trả HTML của MinIO Console.

Test route media:

```bash
curl -ik --resolve sauriengthientam.vn:443:127.0.0.1 \
https://sauriengthientam.vn/media/
```

Kết quả:

```text
HTTP/2 403
server: MinIO
```

Đây cũng là `AccessDenied` từ MinIO khi truy cập root bucket/API mà chưa cung cấp quyền/object cụ thể.

**Checkpoint:** Caddy route đúng tới MinIO; 9000 API và 9001 Console hoạt động.

---

## 22. Kiểm tra DNS và phát hiện IP thực tế của domain production

Kiểm tra DNS:

```bash
dig +short sauriengthientam.vn
```

Kết quả:

```text
45.117.76.118
```

Kiểm tra public IP của VPS hiện tại:

```bash
curl -4 ifconfig.me
```

Kết quả:

```text
14.161.10.44
```

Như vậy domain production đang trỏ tới IP khác với VPS mới. Đây là lý do cần test domain bằng `--resolve` thay vì đổi DNS production ngay.

---

## 23. Kiểm tra firewall UFW

```bash
ufw status
```

Các rule liên quan:

```text
80/tcp                     ALLOW Anywhere
443/tcp                    ALLOW Anywhere
22/tcp                     ALLOW Anywhere
10012/tcp                  ALLOW 192.168.100.0/24
...
```

Port 80/443 đã được phép qua UFW.

Kiểm tra process listen:

```bash
ss -lntp | grep -E ':80 |:443 |:9000 |:9001 '
```

Kết quả xác nhận Docker proxy đang listen trên cả 80, 443, 9000 và 9001.

---

## 24. Test domain production mà không thay đổi DNS bên ngoài

Do domain production đang chạy thật ở IP khác, không đổi DNS ngay.

Trên Windows client, sử dụng `curl --resolve` để ép hostname về IP VPS mới.

Ban đầu đã test nhầm IP cũ/khác và nhận response từ nginx của hệ thống khác. Sau đó xác định đúng IP VPS mới là:

```text
27.74.254.52
```

Test HTTPS:

```powershell
curl.exe -Ik --resolve sauriengthientam.vn:443:27.74.254.52 https://sauriengthientam.vn/
```

Kết quả:

```text
HTTP/1.1 200 OK
Alt-Svc: h3=":443"; ma=2592000
Content-Type: text/html
Via: 1.1 Caddy
X-Robots-Tag: noindex, nofollow
```

Test HTTP:

```powershell
curl.exe -I --resolve sauriengthientam.vn:80:27.74.254.52 http://sauriengthientam.vn/
```

Kết quả:

```text
HTTP/1.1 308 Permanent Redirect
Location: https://sauriengthientam.vn/
Server: Caddy
```

Test API Docs:

```powershell
curl.exe -Ik --resolve sauriengthientam.vn:443:27.74.254.52 https://sauriengthientam.vn/api-docs
```

Kết quả:

```text
HTTP/1.1 200 OK
Access-Control-Allow-Credentials: true
Content-Type: text/html; charset=utf-8
Via: 1.1 Caddy
```

**Checkpoint:** domain production có thể được test hoàn chỉnh trên VPS mới mà không cần đổi DNS public.

---

## 25. Caddy tự cấp SSL/TLS

Trong log Caddy đã thấy automatic TLS cho domain production.

Khi cấu hình thêm domain test, Caddy log:

```text
enabling automatic TLS certificate management
```

Sau đó Caddy thử `tls-alpn-01`, nhận lỗi:

```text
Cannot negotiate ALPN protocol "acme-tls/1" for tls-alpn-01 challenge
```

Caddy tự chuyển sang `http-01`.

Log tiếp tục:

```text
trying to solve challenge
challenge_type: http-01
```

Sau đó:

```text
authorization finalized
identifier: thientam.review360.id.vn
authz_status: valid
```

Cuối cùng:

```text
certificate obtained successfully
identifier: thientam.review360.id.vn
issuer: acme-v02.api.letsencrypt.org-directory
```

**Kết luận:** Caddy tự động xin và quản lý chứng chỉ Let's Encrypt. Không cần tự tạo certificate thủ công cho hostname nếu DNS và port challenge phù hợp.

---

## 26. Tạo domain test riêng để nghiệm thu

Do domain production đang chạy thật, tạo hostname test:

```text
thientam.review360.id.vn
```

Cấu hình Caddy được bổ sung:

```caddy
# Domain Test
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

Production vẫn giữ:

```caddy
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
```

Và MinIO:

```caddy
sauriengthientam.vn:9000 {
    reverse_proxy minio:9000
}

sauriengthientam.vn:9001 {
    reverse_proxy minio:9001
}
```

Validate Caddy:

```bash
docker compose exec caddy \
caddy validate --config /etc/caddy/Caddyfile
```

Kết quả:

```text
Valid configuration
```

Có warning formatting:

```text
Caddyfile input is not formatted; run 'caddy fmt --overwrite'
```

Đây chỉ là warning format, không phải lỗi cấu hình.

Reload:

```bash
docker compose exec caddy \
caddy reload --config /etc/caddy/Caddyfile
```

Reload thành công; warning format vẫn xuất hiện.

---

## 27. Kiểm tra Caddy xin certificate cho domain test

```bash
docker compose logs --tail=100 caddy | grep -i "thientam.review360.id.vn"
```

Log cho thấy:

```text
enabling automatic TLS certificate management
```

Caddy lock certificate:

```text
acquiring lock
lock acquired
```

Thử TLS-ALPN:

```text
challenge_type: tls-alpn-01
```

Challenge này thất bại:

```text
Cannot negotiate ALPN protocol "acme-tls/1"
```

Caddy chuyển sang HTTP-01:

```text
challenge_type: http-01
```

Nhiều Let's Encrypt validation node truy cập và Caddy trả key authentication.

Cuối cùng:

```text
authorization finalized
authz_status: valid
```

Và:

```text
certificate obtained successfully
identifier: thientam.review360.id.vn
```

**Checkpoint:** domain test đã có certificate hợp lệ.

---

## 28. Kiểm tra domain test thực tế

```bash
curl -Ik https://thientam.review360.id.vn/
```

Kết quả:

```text
HTTP/2 200
content-type: text/html
via: 1.1 Caddy
x-robots-tag: noindex, nofollow
```

Do domain đi qua Cloudflare, response có thêm:

```text
server: cloudflare
cf-cache-status: DYNAMIC
```

Test API Docs:

```bash
curl -Ik https://thientam.review360.id.vn/api-docs
```

Kết quả:

```text
HTTP/2 200
content-type: text/html; charset=utf-8
via: 1.1 Caddy
```

Các security header của backend cũng xuất hiện như `strict-transport-security`, `x-content-type-options`, `x-frame-options`.

**Checkpoint:** domain test public hoạt động; frontend và API Docs đều đi qua Caddy tới đúng container.

---

## 29. Mô hình test domain không cần đổi DNS production

Có hai cách test đã được sử dụng/trao đổi:

### Cách 1 — `curl --resolve`

Không thay đổi DNS public. Ví dụ:

```powershell
curl.exe -Ik --resolve sauriengthientam.vn:443:27.74.254.52 https://sauriengthientam.vn/
```

Điểm quan trọng:

- Hostname gửi lên vẫn là `sauriengthientam.vn`.
- TLS SNI vẫn là hostname production.
- Kết nối TCP được ép tới IP VPS mới.
- DNS public không bị thay đổi.

### Cách 2 — hostname test

Dùng:

```text
thientam.review360.id.vn
```

Hostname test có thể được trỏ tới VPS mới để Khách kiểm tra độc lập. Sau khi nghiệm thu có thể giữ hoặc tháo tùy mục đích.

---

## 30. Trạng thái hệ thống sau khi hoàn thành chuỗi kiểm tra

Các container đã chạy:

```text
thientam-backend-1
thientam-caddy-1
thientam-db-1
thientam-frontend-1
thientam-minio-1
```

Các service:

```text
db
minio
backend
frontend
caddy
```

Các endpoint/route đã kiểm tra:

| Thành phần | Kiểm tra | Kết quả |
|---|---|---|
| PostgreSQL | `SELECT version()` | OK |
| PostGIS | `PostGIS_Version()` | OK |
| PostgreSQL schema | `\dt` | OK |
| MinIO | container/log | OK |
| MinIO API 9000 | HTTPS | 403 AccessDenied — đúng khi chưa auth |
| MinIO Console 9001 | HTTPS | 200 |
| Backend internal | `/health` | 200 |
| Frontend internal | `/` | 200 |
| Caddy HTTP | port 80 | 308 → HTTPS |
| Caddy HTTPS | production hostname | 200 |
| API Docs | `/api-docs` | 200 |
| GraphQL | `/graphql` GET không query | 400 — route tới backend |
| Media | `/media/` | 403 từ MinIO — route đúng |
| Domain test | `/` | 200 |
| Domain test | `/api-docs` | 200 |
| TLS domain test | Let's Encrypt | Certificate obtained successfully |

---

## 31. Các điểm cần nhớ khi làm lại từ đầu

### 31.1 Không restore DB trước khi kiểm tra checksum

Luôn:

```text
Backup archive
   ↓
SHA256SUMS
   ↓
Tất cả OK
   ↓
Load image
   ↓
Restore project
   ↓
Restore volumes
   ↓
Start DB
   ↓
Test DB/PostGIS
   ↓
Start MinIO
   ↓
Test MinIO
   ↓
Start Backend
   ↓
Test /health trong container
   ↓
Start Frontend
   ↓
Test frontend trong container
   ↓
Start Caddy
   ↓
Test HTTP/HTTPS
   ↓
Test API Docs/GraphQL/Media
   ↓
Test domain
```

### 31.2 Không dùng `curl 127.0.0.1:3000` từ host để kết luận Backend chết

Port 3000 không publish ra host. Backend nằm trong Docker network.

Test đúng bằng Node bên trong container:

```bash
docker exec thientam-backend-1 node -e "..."
```

### 31.3 Không test HTTPS hostname bằng `https://127.0.0.1`

Certificate cấp cho domain, nên phải dùng hostname thật với `--resolve`:

```bash
curl -Ik --resolve sauriengthientam.vn:443:127.0.0.1 https://sauriengthientam.vn/
```

### 31.4 Port 80 phải publish nếu muốn Caddy xử lý HTTP-01 và HTTP → HTTPS

Docker Compose cần:

```yaml
- "80:80"
- "443:443"
```

### 31.5 Domain production có thể test trước khi đổi DNS

Dùng:

```powershell
curl.exe -Ik --resolve sauriengthientam.vn:443:<IP-VPS-MỚI> https://sauriengthientam.vn/
```

Không cần thay DNS production.

### 31.6 Caddy tự quản lý certificate

Không cần copy certificate thủ công nếu Caddy có quyền thực hiện ACME challenge và volume `/data`/`/config` được giữ nguyên.

---

## 32. Bảo mật VPS/SSH đã trao đổi trước chuỗi restore

Trong quá trình chuẩn bị VPS mới, SSH được harden và giữ một session an toàn trong lúc thay đổi cấu hình.

SSH dùng port:

```text
10012
```

Port SSH được giới hạn cho mạng quản trị:

```text
192.168.100.0/24
```

User quản trị là user không phải root, có sudo. Trước khi khóa root/password login đã xác nhận key login và:

```bash
sudo whoami
```

trả:

```text
root
```

Cấu hình bảo mật SSH được đặt tại:

```text
/etc/ssh/sshd_config.d/99-security.conf
```

Effective configuration đã được kiểm tra với các giá trị chính:

```text
Port 10012
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
AllowUsers namdag
```

Có một điểm cần xử lý: file cloud-init:

```text
50-cloud-init.conf
```

có `PasswordAuthentication yes`, nên phải đổi thành `no` để không ghi đè chính sách hardening.

Sau đó kiểm tra effective SSH configuration và giữ session SSH an toàn trước khi reload.

---

## 33. Các lệnh kiểm tra nhanh có thể dùng lại

### Kiểm tra container

```bash
docker compose ps
```

### Kiểm tra image

```bash
docker images
```

### Kiểm tra volume

```bash
docker volume ls | grep thientam
```

### Validate Compose

```bash
docker compose config -q
```

### Validate Caddy

```bash
docker compose exec caddy caddy validate --config /etc/caddy/Caddyfile
```

### Reload Caddy

```bash
docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
```

### Log Caddy

```bash
docker compose logs --tail=100 caddy
```

### Log backend

```bash
docker compose logs --tail=100 backend
```

### Log frontend

```bash
docker compose logs --tail=100 frontend
```

### Log database

```bash
docker compose logs --tail=100 db
```

### Log MinIO

```bash
docker compose logs --tail=100 minio
```

### Kiểm tra port host

```bash
ss -lntp | grep -E ':80 |:443 |:9000 |:9001 '
```

### Test HTTP

```bash
curl -I http://127.0.0.1
```

### Test HTTPS theo hostname local

```bash
curl -Ik --resolve sauriengthientam.vn:443:127.0.0.1 https://sauriengthientam.vn/
```

### Test domain test

```bash
curl -Ik https://thientam.review360.id.vn/
curl -Ik https://thientam.review360.id.vn/api-docs
```

---

## 34. Kết luận của quá trình

Hệ thống Thiện tâm trên VPS mới đã được đưa qua từng checkpoint theo thứ tự:

1. Xác nhận OS/hardware/Docker.
2. Xác nhận aaPanel/SSH/port.
3. Xác nhận Docker sạch.
4. Đưa backup lên VPS.
5. Kiểm tra archive.
6. Kiểm tra toàn bộ SHA256.
7. Load Docker images.
8. Restore project.
9. Validate Docker Compose.
10. Tạo Docker volumes.
11. Restore MinIO.
12. Restore Caddy config/data.
13. Start DB.
14. Verify PostgreSQL/PostGIS/schema.
15. Start MinIO và verify data.
16. Start Backend và verify `/health` nội bộ.
17. Start Frontend và verify HTTP 200 nội bộ.
18. Start Caddy.
19. Phát hiện port 80 chưa publish.
20. Sửa Compose để publish 80.
21. Verify HTTP 308 → HTTPS.
22. Verify HTTPS bằng hostname + `--resolve`.
23. Verify API Docs.
24. Verify GraphQL route.
25. Verify MinIO API/Console/media route.
26. Verify UFW.
27. Xác định domain production đang trỏ IP khác.
28. Test production hostname tới VPS mới bằng `--resolve`, không đổi DNS.
29. Tạo hostname test `thientam.review360.id.vn`.
30. Caddy tự xin certificate Let's Encrypt.
31. Verify domain test HTTPS.
32. Verify domain test frontend/API Docs.

**Trạng thái tại thời điểm kết thúc phần trao đổi trước khi xây dựng bộ tài liệu:** hệ thống đã chạy đủ 5 service Docker, Caddy đã publish 80/443, domain test đã có certificate Let's Encrypt, frontend/API Docs hoạt động, MinIO Console hoạt động, DB/PostGIS hoạt động, và domain production có thể được kiểm tra mà không cần đổi DNS production bằng `curl --resolve`.

---

## Phụ lục A — Cấu hình Caddy tại thời điểm kiểm tra

```caddy
{
    email admin@salefronts.net
}

# Domain production
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

# Domain test
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

## Phụ lục B — Docker Compose: phần Caddy sau khi bổ sung port 80

```yaml
caddy:
  ports:
    - "80:80"
    - "443:443"
    - "9000:9000"
    - "9001:9001"
```

---

## Phụ lục C — Các IP/hostname đã xuất hiện trong quá trình kiểm tra

| Giá trị | Vai trò |
|---|---|
| `14.161.10.44` | IP được kiểm tra ban đầu nhưng không phải IP VPS mới tại thời điểm xác nhận cuối |
| `27.74.254.52` | IP VPS mới được dùng để `curl --resolve` từ Windows |
| `45.117.76.118` | IP DNS public của `sauriengthientam.vn` tại thời điểm kiểm tra; domain production chưa trỏ về VPS mới |
| `172.18.0.3` | IP container MinIO tại thời điểm log |
| `172.18.0.4` | IP container Backend |
| `172.18.0.5` | IP container Frontend |
| `sauriengthientam.vn` | Domain production |
| `thientam.review360.id.vn` | Domain test |

---

## Phụ lục D — Nguyên tắc nghiệm thu đã hình thành trong quá trình trao đổi

Không chuyển sang bước tiếp theo chỉ vì container đã `Up`. Mỗi tầng phải có test thực tế:

```text
Container Up
   ↓
Log không có lỗi nghiêm trọng
   ↓
Service nội bộ trả kết quả đúng
   ↓
Reverse proxy route đúng
   ↓
HTTP/HTTPS đúng
   ↓
TLS certificate đúng hostname
   ↓
Domain test đúng
   ↓
API Docs/API/GraphQL/Media kiểm tra
   ↓
Sau cùng mới thực hiện DNS/cutover production
```

Đây là nguyên tắc chính của quá trình cài đặt/restore Thiện tâm được ghi lại trong cuộc trao đổi này.
