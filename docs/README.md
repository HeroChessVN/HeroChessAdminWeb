# AdminWeb — Tài liệu

Tài liệu của Hero Chess AdminWeb (React + TypeScript + Vite). Thư mục này tự chứa
đủ để phát triển, vận hành và deploy AdminWeb mà **không cần** source Backend nằm chung
thư mục.

## Danh mục

| Tài liệu | Nội dung |
|---|---|
| [`../README.md`](../README.md) | Tổng quan, cài đặt, chạy, build, deploy, xử lý sự cố. |
| [`ARCHITECTURE.md`](./ARCHITECTURE.md) | Routes, API client, auth, protected routes, error handling, shared components, tích hợp Backend. |
| [`DEVELOPMENT.md`](./DEVELOPMENT.md) | Quy trình dev hằng ngày: env, proxy, lint/test/typecheck, quy ước. |
| [`DEPLOYMENT.md`](./DEPLOYMENT.md) | Build production, `base` path, các phương án hosting. |
| [`API_INTEGRATION.md`](./API_INTEGRATION.md) | Endpoint Admin API mà AdminWeb thực sự gọi + hợp đồng xác thực. |
| [`ADMIN_CONSOLE.md`](./ADMIN_CONSOLE.md) | Bảng route, mô hình phân quyền, luồng quản lý tài khoản admin. |
| [`../ui-audit/screenshots`](../ui-audit/screenshots) | Ảnh kiểm thử UI (desktop / tablet / mobile, các trạng thái lỗi). |

## Quan hệ với tài liệu Backend

Contract API **thuộc sở hữu của Backend repository** (`HeroChessBackend`). Bản tài liệu
đầy đủ về contract (mọi endpoint, kể cả endpoint AdminWeb chưa dùng) nằm ở:

- `HeroChessBackend/docs/admin/ADMIN_API_GUIDE.md` — contract đầy đủ của Admin API.
- `HeroChessBackend/docs/admin/ACCOUNT_ADMIN.md` — hướng dẫn vận hành console (bản gốc).
- `HeroChessBackend/docs/admin/IMPLEMENTATION_AUDIT.md` — rà soát triển khai module Admin.

`API_INTEGRATION.md` trong thư mục này là **bản tham khảo rút gọn, chỉ liệt kê endpoint
AdminWeb thực sự gọi** và được ghi rõ nguồn. Khi hai bên lệch nhau, **Backend là nguồn
sự thật**; hãy sửa tài liệu ở Backend repository rồi cập nhật bản tham khảo ở đây.

## Nguồn gốc các tài liệu trong thư mục này

| File | Nguồn | Xử lý |
|---|---|---|
| `API_INTEGRATION.md` | `docs/admin/ADMIN_API_GUIDE.md` (Backend) | Rút gọn, thêm mục "Nguồn". Không copy contract đầy đủ. |
| `ADMIN_CONSOLE.md` | `docs/admin/ACCOUNT_ADMIN.md` (Backend) | Rút gọn phần liên quan AdminWeb; bỏ thông tin đăng nhập/nhạy cảm. |
| `ARCHITECTURE.md`, `DEVELOPMENT.md`, `DEPLOYMENT.md`, `README.md` | — | Viết mới từ source code AdminWeb. |

Bản gốc trong Backend repository **không bị xoá** — nếu tách AdminWeb ra repository riêng,
vẫn cần giữ Backend repository để tham chiếu contract API.
