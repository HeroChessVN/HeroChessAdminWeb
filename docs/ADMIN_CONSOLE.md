# AdminWeb — Console quản trị

> **Nguồn:** rút gọn từ `HeroChessBackend/docs/admin/ACCOUNT_ADMIN.md` (Backend
> repository). Bản gốc không bị xoá.
> **Thông tin tài khoản/mật khẩu cụ thể không được nhân bản vào repo này** — tài khoản
> admin được tạo qua `POST /api/v1/admin/accounts` (không chèn hash trực tiếp vào DB).

---

## 1. Chạy console

### 1.1. Backend API (port 5012)

```powershell
cd <HeroChessBackend>\HeroChess.Api
dotnet run --launch-profile http
```

- Profile `http` → `http://0.0.0.0:5012`, `ASPNETCORE_ENVIRONMENT=Development`.
- Sống: <http://127.0.0.1:5012/health/ready> → `{"status":"ready","database":"connected"}`.
- Swagger: <http://127.0.0.1:5012/swagger> (chỉ ở Development).
- Backend **không bật CORS** → AdminWeb phải gọi qua Vite proxy `/api/v1`.

### 1.2. AdminWeb (port 5175)

```bash
cd <HeroChessAdminWeb>
npm ci
npm run dev
```

Mở <http://127.0.0.1:5175/#/login>.

Đổi backend → set `HERO_CHESS_API_URL` rồi chạy lại `npm run dev`.

---

## 2. Đăng nhập

| Bước | Endpoint | Ghi chú |
|---|---|---|
| Đăng nhập | `POST /api/v1/auth/login?useCookies=false` | Body `{ email, password }` → `accessToken` + `refreshToken`. |
| Làm mới | `POST /api/v1/auth/refresh` | Body `{ refreshToken }`. |
| Hồ sơ | `GET /api/v1/me` | `userId`, `displayName`, `role`, `status`. |
| Quyền của chính mình | `GET /api/v1/admin/accounts` | Lọc theo `id` của `/me`. Cần policy `admin-read:admin`. |

Lưu phiên: `sessionStorage`, key `hero-chess-admin-session`. Phiên **mất khi đóng tab**.

Luật mật khẩu do ASP.NET Core Identity quy định: ≥ 8 ký tự, có chữ hoa, chữ thường,
số (`RequireNonAlphanumeric = false`).

### 2.1. Tài khoản admin

- Tạo qua API `POST /api/v1/admin/accounts` — **không** chèn hash trực tiếp vào DB.
- Chỉ tạo thêm khi được phê duyệt.
- `permissions` = `null` → toàn quyền (tương thích ngược với admin tạo trước migration).
- `permissions` = `["*"]` → toàn quyền.
- `permissions` = `[]` → không có quyền miền nào.

---

## 3. Route (HashRouter)

Base: `http://127.0.0.1:5175/#<route>`

| Route | Trang | Quyền |
|---|---|---|
| `#/login` | Đăng nhập | — |
| `#/` | Tổng quan vận hành | `read` |
| `#/heroes` | Quản lý Hero | `hero` |
| `#/shop` | Quản lý Cửa hàng | `shop` |
| `#/players` | Quản lý Người chơi | `player` |
| `#/gift-codes` | Quản lý Gift Code | `giftcode` |
| `#/matches` | Trình xem Trận đấu | `analytics` |
| `#/analytics` | Phân tích cân bằng & bot | `analytics` |
| `#/admin` | Tài khoản Admin (phân quyền) | `admin` |
| `#/audit` | Nhật ký thao tác | `audit` |
| `#/403` | Không đủ quyền | — |
| bất kỳ route khác | 404 "Không tìm thấy trang" | — |

Router dùng `HashRouter` vì backend serve static file không có SPA fallback (deep-link
kiểu `/admin/heroes` sẽ 404).

---

## 4. Mô hình phân quyền

Cấu hình quyền nằm ở bảng `hero_chess.admin_permission` (`admin_user_id`,
`permissions text[]`).

- **Không có dòng quyền** → toàn quyền (tương thích ngược).
- **Có dòng** → theo danh sách; `["*"]` = toàn quyền.
- Quyền đọc: `admin-read:{miền}`. Quyền ghi: `admin-write:{miền}` — ghi bao hàm đọc
  cùng miền.
- Miền: `read`, `hero`, `shop`, `player`, `giftcode`, `analytics`, `ruleset`, `audit`,
  `admin`.

Mọi request đều đối chiếu `role` và `status` với DB ở thời điểm gọi — không tin claim cũ.

Mọi thao tác **ghi** đều được ghi vào `hero_chess.admin_audit_log` (actor, before/after,
reason) → hiển thị ở trang `#/audit`.

---

## 5. Lớp quân — nguồn duy nhất là DB

Bảng `hero_chess.chess_class` (7 dòng) là nguồn duy nhất. AdminWeb **không** hardcode
danh sách lớp:

- Backend: `GET /api/v1/admin/chess-classes` đọc thẳng bảng `chess_class`, trả
  `ChessClassDto` (`code`, `nameVi`, `baseSp`, `requiredCount`, `baseMovementCode`).
- Form tạo hero: trường "Lớp quân" là **text read-only**, nhãn lấy từ DB. Hero mới luôn
  thuộc lớp mặc định `ADVISOR` (quân Sĩ) — không có dropdown chọn.
- Form sửa hero: lớp hiển thị **read-only** (`disabled`) — không đổi được sau khi tạo;
  API `PATCH /admin/heroes/{id}` cũng không nhận `classCode`.
- `src/lib/classLabel.ts` có `FALLBACK_CLASSES` (ảnh chụp DB) chỉ để console vẫn chạy
  được khi endpoint `/admin/chess-classes` lỗi.

---

## 6. Quy tắc thao tác ghi

| Thao tác | `reason` bắt buộc | Confirm dialog |
|---|---|---|
| Bật/tắt hero | Có | Có |
| Gán phe hero | Có | — |
| Đổi giá / bật-tắt bán trong shop | Có | Có (đổi giá hàng loạt) |
| Khoá / mở khoá người chơi | Có | Có |
| Cộng / trừ coin | Có | Có |
| Cấp / thu hero | Có | Có (thu) |
| Tạo / sửa gift code | Có | — |
| Bật/tắt gift code | Có | Có |
| Tạo tài khoản admin / đặt lại quyền | — | Có (đặt lại quyền) |

Client validate `reason` trước khi gửi; backend cũng validate (`AdminValidation.RequireReason`).

---

## 7. Kiểm tra nhanh bằng PowerShell

```powershell
$base = 'http://127.0.0.1:5012/api/v1'
$login = Invoke-RestMethod "$base/auth/login?useCookies=false" -Method Post `
  -ContentType 'application/json' `
  -Body (@{ email = '<admin-email>'; password = '<admin-password>' } | ConvertTo-Json -Compress)
$h = @{ Authorization = "Bearer $($login.accessToken)" }

(Invoke-WebRequest "$base/me"             -Headers $h -UseBasicParsing).Content
(Invoke-WebRequest "$base/admin/accounts" -Headers $h -UseBasicParsing).Content
(Invoke-WebRequest "$base/admin/stats"    -Headers $h -UseBasicParsing).Content
```

Bỏ `-UseBasicParsing` sẽ lỗi `Windows PowerShell is in NonInteractive mode` khi server
trả 401.

---

## 8. Tài liệu liên quan

- [`API_INTEGRATION.md`](./API_INTEGRATION.md) — endpoint AdminWeb thực sự gọi.
- [`ARCHITECTURE.md`](./ARCHITECTURE.md) — routes, auth, component.
- Backend repository: `docs/admin/ADMIN_API_GUIDE.md` (contract đầy đủ),
  `docs/admin/IMPLEMENTATION_AUDIT.md` (rà soát triển khai module Admin).
