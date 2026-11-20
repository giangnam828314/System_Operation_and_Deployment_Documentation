# TÀI LIỆU CÀI ĐẶT AUTHENTIK – SSO IDENTITY PROVIDER

Ngày lập: 11/11/2026  
Phiên bản: 1.0  
Phạm vi: Cài đặt Authentik bằng Docker Compose trên Ubuntu Server; định hướng tích hợp Synology NAS và ứng dụng nội bộ.

> Trạng thái: Authentik đã khởi tạo và truy cập được Application Dashboard. Chưa tạo OIDC Provider/Application và chưa tích hợp Synology.

## 1. Mục đích
Triển khai Authentik làm Identity Provider (IdP) tập trung để quản lý đăng nhập, xác thực và MFA cho các ứng dụng tương thích.

Định hướng mở rộng: Synology NAS (cần xác minh DSM), KPI Dashboard, ITAM, Monitoring và các ứng dụng web nội bộ hỗ trợ OIDC/SAML.

## 2. Thông số môi trường
| Thành phần | Giá trị ghi nhận |
|---|---|
| Hostname | mstarcorp |
| IP | 192.168.1.156 |
| Hệ điều hành | Ubuntu Server 20.04.6 LTS |
| Nền tảng | VMware VM |
| CPU / RAM | 1 vCPU / khoảng 3.8 GiB RAM; swap khoảng 3.8 GiB |
| Root filesystem | 28 GB, còn khoảng 12 GB tại thời điểm kiểm tra |
| /data | 98 GB, còn khoảng 87 GB tại thời điểm kiểm tra |
| Docker Root Dir | /data/docker |
| Docker / Compose | 26.1.3 / 2.27.1 |
| Thư mục | /data/authentik |
| Authentik | ghcr.io/goauthentik/server:2026.8.5 |
| HTTP / HTTPS | 192.168.1.156:9000 / 127.0.0.1:9443 |

Lưu ý tài nguyên: 1 vCPU và khoảng 4 GB RAM chỉ phù hợp thử nghiệm/POC. Máy đang chạy các container khác, bao gồm KPI Dashboard. Không dừng hoặc thay đổi container hiện hữu nếu chưa đánh giá tác động.

## 3. Kiến trúc container
| Service | Image | Vai trò |
|---|---|---|
| postgresql | docker.io/library/postgres:16-alpine | Cơ sở dữ liệu |
| server | ghcr.io/goauthentik/server:2026.8.5 | Web UI, API, dịch vụ Authentik |
| worker | ghcr.io/goauthentik/server:2026.8.5 | Tác vụ nền và blueprint |

Dữ liệu được lưu bền vững qua PostgreSQL volume và các thư mục mount dưới /data/authentik.

## 4. Chuẩn bị thư mục
Chạy bằng tài khoản có quyền sudo/root:

```bash
mkdir -p /data/authentik
cd /data/authentik
mkdir -p backups data certs custom-templates
df -h /data
```

## 5. Tải và chuẩn bị Compose
```bash
cd /data/authentik
wget https://docs.goauthentik.io/compose.yml -O compose.yml
```

Trong lần triển khai này, Docker socket mount đã được bỏ khỏi worker vì mục tiêu hiện tại là OIDC SSO cho NAS/web app và chưa cần Docker Outpost:

```bash
sed -i '\|/var/run/docker.sock:/var/run/docker.sock|d' compose.yml
```

Không cấp quyền Docker socket cho container nếu chưa có yêu cầu cụ thể và đánh giá rủi ro.

## 6. Tạo file .env và secrets
```bash
cat > .env <<EOF
PG_PASS=$(openssl rand -base64 36 | tr -d '\n')
AUTHENTIK_SECRET_KEY=$(openssl rand -base64 60 | tr -d '\n')
COMPOSE_PORT_HTTP=192.168.1.156:9000
COMPOSE_PORT_HTTPS=127.0.0.1:9443
EOF

chmod 600 .env
ls -l .env
```

Không in nội dung .env, không commit file này lên Git và không gửi secret qua email/chat. HTTP chỉ bind IP LAN; HTTPS 9443 hiện chỉ bind localhost.

## 7. Kiểm tra và khởi chạy
```bash
cd /data/authentik
docker compose config -q
docker compose pull
docker compose up -d
docker compose ps
```

Theo dõi log khởi tạo:
```bash
docker compose logs -f --tail=100 server worker postgresql
```

Lần đầu Authentik cần chạy database migrations và khởi tạo blueprint. Healthcheck có thể ở trạng thái starting lúc đầu. Không restart liên tục chỉ vì cảnh báo trong vài giây đầu.

## 8. Kiểm tra sau cài đặt
```bash
cd /data/authentik
docker compose ps
docker compose logs server --since=5m --tail=100
docker compose logs worker --since=5m --tail=100
docker compose logs postgresql --since=5m --tail=100
curl -sS -o /dev/null -w 'HTTP Status: %{http_code}\n' http://192.168.1.156:9000/
```

Dấu hiệu đã quan sát: PostgreSQL healthy; Server hoàn tất migration, chạy Gunicorn và health endpoint trả HTTP 200; Worker xử lý blueprint; embedded outpost kết nối WebSocket; trình duyệt truy cập được Application Dashboard.

## 9. Truy cập giao diện
Địa chỉ: http://192.168.1.156:9000/

Truy cập trực tiếp /if/flow/initial-setup/ đã hiển thị “Request has been denied” và hướng dẫn truy cập trang gốc. Sau đó truy cập trang gốc đã vào được Application Dashboard.

Dashboard hiển thị “No Applications available” là bình thường vì chưa tạo ứng dụng SSO.

## 10. Sự cố đã gặp
| Hiện tượng | Xử lý / nhận định |
|---|---|
| Healthcheck starting lúc mới khởi động | Chờ migration/worker hoàn tất, kiểm tra lại trạng thái và log. |
| Worker báo bảng authentik_tasks_workerstatus chưa tồn tại | Có thể xuất hiện trước khi migration hoàn tất; kiểm tra log mới hơn. |
| /setup trả HTTP 503 | Log cho biết Setup Flow chưa tồn tại và đang chờ worker; chờ blueprint xử lý rồi truy cập trang gốc. |
| Initial setup bị Request denied | Truy cập http://192.168.1.156:9000/ theo hướng dẫn trên giao diện. |
| Dashboard báo No Applications available | Bình thường khi chưa tạo Provider/Application. |

Không chạy docker compose down -v, không xóa database volume hoặc thư mục dữ liệu để xử lý lỗi khởi động thông thường.

## 11. Bảo mật và vận hành
- Bảo vệ /data/authentik/.env với quyền 600.
- Không công khai port 9000/9443 ra Internet.
- Khi vận hành chính thức cần DNS phù hợp, HTTPS/TLS, firewall và URL nhất quán.
- Bật MFA cho tài khoản quản trị sau khi thiết kế chính sách xác thực.
- Giữ tài khoản quản trị cục bộ Synology; chỉ chuyển sang SSO sau khi thử nghiệm đăng nhập và khôi phục thành công.
- Theo dõi CPU/RAM/disk do VM có tài nguyên hạn chế và dùng chung với container khác.
- Nâng cấp có kế hoạch: đọc release notes, backup, kiểm tra tương thích rồi mới cập nhật image.

## 12. Sao lưu và khôi phục
Chưa thực hiện backup/restore trong phiên cài đặt. Trước vận hành cần lịch sao lưu PostgreSQL và dữ liệu cấu hình, lưu ở vị trí an toàn và kiểm thử restore.

Ví dụ dump thủ công (phải kiểm tra tên database/user trong compose thực tế trước khi dùng):
```bash
cd /data/authentik
mkdir -p backups
docker compose exec -T postgresql sh -c \
  'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' \
  > backups/authentik-$(date +%F-%H%M).sql
```

File backup có thể chứa dữ liệu định danh, cần bảo vệ tương ứng. Không coi backup là hợp lệ nếu chưa thử restore trên môi trường kiểm thử.

## 13. Các bước tiếp theo
| Giai đoạn | Công việc | Trạng thái |
|---|---|---|
| 1 | Cài Authentik bằng Docker Compose | Hoàn tất |
| 2 | Xác minh Server/Worker và truy cập Dashboard | Hoàn tất |
| 3 | Thiết lập users, groups và MFA | Chưa thực hiện |
| 4 | Xác minh model Synology và phiên bản DSM | Chờ thông tin |
| 5 | Tạo OIDC Provider/Application nếu DSM hỗ trợ | Chưa thực hiện |
| 6 | Cấu hình DSM SSO Client, kiểm thử tài khoản thử nghiệm | Chưa thực hiện |
| 7 | Kiểm thử fallback local admin và quy trình khôi phục | Chưa thực hiện |
| 8 | Mở rộng KPI Dashboard, ITAM, Monitoring | Định hướng |

Cần model NAS và phiên bản DSM chính xác trước khi cấu hình. Khả năng hỗ trợ OIDC/SSO và claim mapping phụ thuộc DSM.

OIDC SSO cho giao diện web không tự động thay thế xác thực SMB/CIFS cho Windows/network share. Nhu cầu SMB phải được thiết kế danh tính riêng, ví dụ LDAP/Active Directory hoặc cơ chế tương thích của NAS.

## 14. Nghiệm thu trạng thái hiện tại
| Hạng mục | Kết quả |
|---|---|
| Thư mục /data/authentik | Đã tạo |
| compose.yml | Đã tải và bỏ Docker socket |
| .env và secrets | Đã tạo, chmod 600 |
| PostgreSQL | Healthy theo lần kiểm tra |
| Authentik Server | Khởi chạy; health endpoint HTTP 200 trong log |
| Authentik Worker | Đã xử lý blueprint |
| Giao diện web | Truy cập được Application Dashboard |
| Provider/Application SSO | Chưa tạo |
| Synology SSO | Chưa cấu hình |

## 15. Tài liệu tham khảo
- Compose: https://docs.goauthentik.io/compose.yml
- Image: ghcr.io/goauthentik/server:2026.8.5

Khi nâng cấp hoặc triển khai mới cần đối chiếu tài liệu chính thức và release notes tại thời điểm thực hiện.

## Phụ lục – Lệnh vận hành
```bash
cd /data/authentik
docker compose ps
docker compose logs --tail=100 server
docker compose logs --tail=100 worker
docker compose logs --tail=100 postgresql
docker compose logs -f server worker
docker compose config -q
docker compose up -d
```

Không dùng down -v hoặc xóa volume nếu chưa có bản backup đã kiểm thử.