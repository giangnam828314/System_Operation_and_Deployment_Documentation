# Authentik SSO – Tài liệu triển khai

Tài liệu ghi nhận triển khai Authentik bằng Docker Compose và tích hợp Google Workspace làm nguồn đăng nhập cho SSO MSTAR.

## Tài liệu

- [Cài đặt Authentik SSO (Markdown)](./Tai-lieu-cai-dat-Authentik-SSO.md)
- [Tích hợp Google Workspace với Authentik (Markdown)](./Tich-hop-Google-Workspace-voi-Authentik-SSO.md)

## Kiến trúc mục tiêu

Google Workspace → Authentik → Ứng dụng nội bộ (KPI Dashboard, Synology NAS, Bitrix24, Monitoring và các ứng dụng hỗ trợ OIDC/SAML).

## Trạng thái

- Cài đặt Authentik và truy cập giao diện: hoàn tất.
- Google OAuth Source: đã cấu hình; đăng nhập thử nghiệm quay lại callback và tạo External User thành công.
- Đồng bộ Google Directory/Groups và vòng đời tài khoản: chưa triển khai.
- OIDC Provider/Application cho KPI Dashboard: chưa triển khai.
- Tích hợp DSM/NAS: cần xác minh model NAS, phiên bản DSM và giao thức hỗ trợ.

> Không đưa file .env, Client Secret, mật khẩu, token hoặc bản dump database chứa dữ liệu định danh vào repository.
