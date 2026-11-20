# Role matrix MCP cho ERPNext staging

## Mục đích và nguyên tắc

Built-in MCP tại `/api/v2/mcp` thực thi bằng user Frappe đang đăng nhập và gọi cùng REST permission checks. MCP có tool tạo, sửa, xóa và gọi whitelisted method, nên quyền phải được chặn ở server bằng Role Permission Manager, User Permission, Workflow và account scope; mô tả agent hoặc prompt không phải security boundary.

Không cấp `Administrator`, không dùng chung user giữa agent và không cấp `System Manager` chỉ để MCP hoạt động. Mỗi user MCP phải là System User riêng, có API/OAuth credential riêng và có thể revoke độc lập.

## Vai trò đề xuất

| Role | Đối tượng | Quyền cho phép | Quyền phải tắt |
| --- | --- | --- | --- |
| `MCP Staging Viewer` | Lead, agent tra cứu | read, report trên master data và chứng từ được cấp; xem metadata cần thiết | create, write, delete, submit, cancel, amend, import, export, share |
| `MCP Staging Draft Operator` | Functional/Engineer staging | read, report, create và write chứng từ `docstatus=0` trong company/project được cấp | delete, submit, cancel, amend, payment, stock posting, role/user changes, import, export, share |
| `MCP Staging Assurance` | Assurance | read/report; tạo/sửa dữ liệu test trong company/warehouse test nếu test case yêu cầu | mọi thao tác production; delete, submit, cancel, amend, role/user changes |

Không tạo role production-writable trong giai đoạn này. Nếu sau này có production MCP, mặc định dùng viewer và approval phải được enforce ngoài prompt trước mọi mutation.

## Scope dữ liệu

- Tạo User Permission cho `Company`, `Project`, `Warehouse`, `Cost Center` và các dimension liên quan; không để agent thấy toàn bộ company nếu không cần.
- Draft Operator chỉ thao tác trên staging company/test master data. Các DocType nhạy cảm như `User`, `Role`, `Role Profile`, `DocPerm`, `Payment Entry`, `Journal Entry`, `Stock Entry` và cấu hình tích hợp không được cấp write mặc định.
- N9/NEM82 RBAC là hệ thống external-app authorization riêng. Không tự map LDAP group thành ERPNext role trước khi có thiết kế identity linking và authoritative mapping được duyệt.

## Kiểm thử bắt buộc

Positive tests:

1. Viewer list/get một master data và chạy report được phép.
2. Draft Operator tạo và cập nhật một draft trong staging scope.
3. Assurance đọc report và fixture trong test scope.

Negative tests:

1. Viewer không thể create/update/delete.
2. Draft Operator không thể submit/cancel/amend/delete chứng từ.
3. Mọi role không thể tạo User, sửa Role/DocPerm hoặc cấp quyền cho chính mình.
4. User Permission chặn truy cập company/project/warehouse ngoài scope.
5. Whitelisted method có side effect bị từ chối nếu user thiếu permission.

Lưu request ID, user, tool, DocType, kết quả và bằng chứng dữ liệu không đổi cho từng negative test. Chỉ bật MCP sau khi cả positive và negative suite pass.
