# Runbook triển khai ERPNext staging 1.243

## Điều kiện tiên quyết

- Có phê duyệt deploy trên runtime 1.243, domain staging và quyền cấu hình reverse proxy.
- Xác nhận `127.0.0.1:18080` còn trống và không có stack `erpnext-staging` khác.
- Có nơi lưu backup ngoài cùng Docker host và ngân sách dung lượng; `/home` đang ở mức 83% khi audit.
- Tạo `deploy/staging/.env` từ `env.example`; không commit file này hoặc in secret ra issue/log.

## Validate cấu hình

```bash
cd deploy/staging
set -a
. ./.env
set +a
docker compose --env-file .env config --quiet
docker compose --env-file .env config --images
```

Kiểm tra image digest sau khi pull:

```bash
docker compose --env-file .env pull
docker image inspect frappe/erpnext:v16.60.0 --format '{{json .RepoDigests}}'
```

## Khởi tạo và chạy

```bash
cd deploy/staging
docker compose --env-file .env up -d db redis-cache redis-queue configurator
docker compose --env-file .env --profile setup run --rm create-site
docker compose --env-file .env up -d backend frontend websocket queue-short queue-long scheduler
docker compose --env-file .env ps
```

Trước khi có reverse proxy, kiểm tra nội bộ bằng `http://127.0.0.1:18080`. Không bind `0.0.0.0` nếu chưa có firewall/TLS approval.

## Nghiệm thu tối thiểu

```bash
docker compose --env-file .env exec backend bench version
docker compose --env-file .env exec backend bench --site "$SITE_NAME" doctor
docker compose --env-file .env exec backend bench --site "$SITE_NAME" list-apps
docker compose --env-file .env logs --since 10m scheduler queue-short queue-long
```

Đăng nhập bằng tài khoản staging qua kênh được phê duyệt, sau đó kiểm tra scheduler/background job và luồng Material Request -> Purchase Order -> Purchase Receipt -> Purchase Invoice. Nếu mở nhánh đánh giá PostgreSQL, chỉ chấp nhận sau khi các luồng, báo cáo tồn kho/công nợ và migration đều pass.

## Backup và restore drill

Tạo backup có file:

```bash
docker compose --env-file .env --profile ops run --rm backup
docker compose --env-file .env exec backend find "sites/$SITE_NAME/private/backups" -maxdepth 1 -type f -printf '%f %s bytes\n'
```

Sao chép backup sang storage ngoài host theo chính sách đã duyệt. Restore chỉ thử trên site tạm, không ghi đè staging đang dùng:

```bash
SITE_NAME=restore-test.example.internal docker compose --env-file .env --profile setup run --rm create-site
docker compose --env-file .env exec backend bench --site restore-test.example.internal restore "sites/$SITE_NAME/private/backups/<database-backup>" --with-public-files "sites/$SITE_NAME/private/backups/<public-files-backup>" --with-private-files "sites/$SITE_NAME/private/backups/<private-files-backup>"
docker compose --env-file .env exec backend bench --site restore-test.example.internal migrate
```

Tên file thực tế phải lấy từ thư mục backup; không đưa password vào command history nếu CLI hỗ trợ file/env tương ứng. Sau restore, chạy login, `doctor`, background job và sample report trước khi ghi nhận thành công.

## Rollback

Nếu site mới chưa có dữ liệu cần giữ:

```bash
docker compose --env-file .env down
```

Lệnh trên giữ volumes để điều tra. Chỉ xóa volumes sau khi xác nhận backup có thể restore và được người có thẩm quyền phê duyệt. Với lỗi image/app, pin lại digest đã biết tốt, chạy `pull`, `up -d`, rồi `bench migrate`; không rollback database bằng cách xóa volume.

## Dependency còn thiếu

- Domain/TLS và owner reverse proxy.
- Credential staging và policy lưu secret.
- Storage backup ngoài host, retention và encryption key.
- Phê duyệt triển khai service và cửa sổ restore drill.
- Role matrix MCP/frappectl để test quyền dương và âm.
- Nếu vẫn yêu cầu PostgreSQL: phê duyệt một evaluation stack riêng và chấp nhận trạng thái experimental được upstream cảnh báo.
