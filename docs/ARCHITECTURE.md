# AdminWeb — Kiến trúc

Mô tả dựa trên source code hiện tại trong `src/`. Mọi khẳng định dưới đây đều có thể
kiểm chứng trong mã nguồn.

## 1. Tổng quan

AdminWeb là một SPA (single-page app) React độc lập, giao tiếp với HeroChess Backend
**chỉ qua HTTP API** `/api/v1`. Nó không đọc/ghi database trực tiếp, không dùng
Supabase SDK và không chứa secret nào.

```
Browser ──► AdminWeb (Vite) ──► /api/v1  ──► HeroChess.Api (ASP.NET Core)
                    │                ▲
                    └── Vite proxy ──┘  (chỉ ở dev, port 5175 → 5012)
```

## 2. Tech stack

| Thành phần | Phiên bản | Ghi chú |
|---|---|---|
| React | 19 | `react`, `react-dom` |
| TypeScript | 5.9 | `strict`, `noUncheckedIndexedAccess`, `verbatimModuleSyntax` |
| Vite | 7 | Dev server + bundler |
| Tailwind CSS | 4 | Qua `@tailwindcss/vite`, cấu hình CSS-first trong `src/styles.css` |
| react-router-dom | 7 | Dùng `HashRouter` |
| lucide-react | 1.x | Icon |
| recharts | 3.x | Khai báo trong `package.json` |

> Ghi chú: `recharts` nằm trong `dependencies` nhưng hiện **không có file nào import nó**
> (`src/pages/AnalyticsPage.tsx` hiện dùng CSS thuần để vẽ thanh tỉ lệ). Xem
> [`../README.md`](../README.md) § "Ghi chú về dependency".

## 3. Cấu trúc thư mục

```
admin-web/
├── src/
│   ├── main.tsx                  # Điểm khởi động: createRoot + StrictMode
│   ├── App.tsx                   # Bảng định tuyến + provider hierarchy
│   ├── styles.css                # Design system (Tailwind v4 @theme + component layer)
│   ├── auth/
│   │   ├── AuthContext.tsx       # Nguồn sự thật về phiên đăng nhập
│   │   └── ProtectedRoute.tsx     # Chặn route khi chưa đăng nhập / thiếu quyền
│   ├── lib/
│   │   ├── api.ts                # HTTP client (fetch wrapper) + ApiError
│   │   ├── session.ts            # Đọc/ghi phiên trong sessionStorage
│   │   ├── permissions.ts        # Ma trận quyền fine-grained
│   │   ├── types.ts              # TypeScript types bám 1-1 AdminContracts.cs
│   │   ├── format.ts             # Định dạng tiền / số / phần trăm / ngày giờ
│   │   ├── format.test.ts        # Unit test (node --test)
│   │   ├── classLabel.ts         # Nhãn lớp quân + fallback
│   │   ├── chessClasses.ts       # Nạp lớp quân từ API
│   │   └── useResource.ts        # Hook tải dữ liệu: useResource / usePagedResource
│   ├── components/
│   │   ├── layout/               # AppLayout, Sidebar, Header
│   │   ├── ui/                   # Button, Card, Badge, Skeleton, Field,
│   │   │                         # ConfirmDialog, Drawer, Toast, JsonViewer,
│   │   │                         # EmptyState, ErrorState
│   │   ├── heroes/               # HeroForm, HeroDetailDrawer
│   │   ├── players/              # PlayerDetailDrawer
│   │   └── matches/              # MatchDetailDrawer
│   └── pages/                    # 12 page component
├── ui-audit/screenshots/         # Ảnh kiểm thử UI
├── docs/                         # Tài liệu (README nằm ở gốc project)
├── index.html                    # Vite entry
├── vite.config.ts                # base path, dev proxy
└── package.json
```

## 4. Routing

`App.tsx` dùng `HashRouter` (lý do: backend phục vụ static file không có SPA fallback, nên
deep-link kiểu `/admin/heroes` sẽ 404; hash `/#/heroes` không cần sửa backend).

| Route | Component | Quyền miền (ProtectedRoute `domain`) |
|---|---|---|
| `/login` | `LoginPage` | — (công khai) |
| `/` (index) | `DashboardPage` | `read` |
| `/heroes` | `HeroesPage` | `hero` |
| `/shop` | `ShopPage` | `shop` |
| `/players` | `PlayersPage` | `player` |
| `/gift-codes` | `GiftCodesPage` | `giftcode` |
| `/matches` | `MatchesPage` | `analytics` |
| `/analytics` | `AnalyticsPage` | `analytics` |
| `/admin` | `AdminAccountsPage` | `admin` |
| `/audit` | `AuditPage` | `audit` |
| `/403` | `ForbiddenPage` | — (nằm trong layout đã đăng nhập) |
| `*` | `NotFoundPage` | — |

Provider hierarchy (`App.tsx`):

```
HashRouter
└── AuthProvider
    └── ToastProvider
        └── Routes
```

`AppLayout` là layout route (dùng `<Outlet />`), bọc bởi một `ProtectedRoute` không
`domain` — tức mọi route con đều yêu cầu đã đăng nhập.

## 5. API client

`src/lib/api.ts` — wrapper mỏng quanh `fetch`:

- **Base URL**: `import.meta.env.VITE_ADMIN_API_URL`, bỏ dấu `/` cuối. Mặc định
  `'/api/v1'` (đường dẫn tương đối).
- **Auth header**: `Authorization: Bearer <accessToken>` khi có token.
- **Body**: `JSON.stringify` + header `Content-Type: application/json` khi body khác
  `undefined`.
- **204 No Content** → trả `undefined` (không parse JSON).
- **401** → gọi `refreshHandler()` (refresh token) một lần rồi thử lại request; nếu
  refresh hỏng → `onUnauthorized()` (logout) và ném lỗi.
- **Lỗi khác** → `parseError()` đọc `{ code, message, title, detail }` và ném `ApiError`.
  Response không phải JSON (lỗi gateway) → giữ message mặc định `"<status> <statusText>"`.

`ApiError` có hai getter phân biệt nghiệp vụ:

- `isUnauthorized` — 401.
- `isPermission` — 403 (đã đăng nhập nhưng thiếu quyền miền; **không phải** lỗi hệ thống).

Hàm viết dấu `_` được bỏ qua.

Hai hàm auth nằm ngoài `request()` vì không dùng Bearer token:

- `login(email, password)` → `POST /api/v1/auth/login?useCookies=false`.
- `refresh(refreshToken)` → `POST /api/v1/auth/refresh`.

`buildQuery(params)` ghép query string, bỏ qua giá trị `null` / `undefined` / `''`.

## 6. Authentication

`src/auth/AuthContext.tsx` là nguồn sự thật về phiên:

- **Lưu trữ**: `sessionStorage`, key `hero-chess-admin-session` (`src/lib/session.ts`).
  Token không ghi rõ vào ổ đĩa; mất khi đóng tab.
- **Nội dung phiên**: `accessToken`, `refreshToken`, `expiresAt`, `user`, `permissions`.
  `permissions` lưu dạng mảng string (vì `Set` không JSON được) → `parsePermissions()`
  dựng lại thành `Set` khi đọc.
- **Bootstrap**: khi mount, `readSession()`; nếu có token → `loadProfile()`:
  - `GET /me` → thông tin tài khoản (`MeDto`).
  - `GET /admin/accounts` → tìm chính mình trong `items` để lấy `permissions`.
- **Endpoint không có quyền riêng**: backend không có API trả quyền của chính mình;
  `GET /admin/accounts` lại cần policy `admin-read:admin`. Vì vậy admin bị giới hạn sẽ
  nhận 403 → `permissions = null` (nghĩa là **chưa xác định**) → UI hiển thị đầy đủ,
  còn server vẫn là nơi quyết định thật.
- **Refresh**: `refreshInFlight` ref để mọi request 401 đồng thời chỉ refresh một lần.
  Refresh thất bại → `logout()`.
- **`configureAuth()`**: gắn `getAccessToken` / `onUnauthorized` / `refresh` vào module
  `api.ts` (dùng `setAccessToken` để cập nhật token sau login/refresh).

## 7. Protected routes & phân quyền

`src/auth/ProtectedRoute.tsx`:

1. `!ready` → hiện màn hình "Đang khôi phục phiên đăng nhập…".
2. `!authenticated` → `<Navigate to="/login" replace state={{ from: location.pathname }} />`.
3. `domain` mà `hasPermission(permissions, domain)` = false → render `ForbiddenPage`
   (route `/403`).
4. Ngược lại → render children.

Ma trận quyền ở `src/lib/permissions.ts`:

| Hàm | Quy tắc |
|---|---|
| `hasPermission(set, domain)` | `set === null` → `true` (chưa xác định → hiển thị). Có `'*'` → `true`. Có `domain` → `true`. `domain === 'read'` → mở khi có key `'read'`. |
| `canWrite(set, domain)` | `set === null` → `true`. `'*'` hoặc có `domain` → `true`. |

Ghi chú: quyền ghi của một miền **bao hàm** quyền đọc cùng miền đó (khớp
`AccountPolicies.HasPermissionAsync` ở backend). Miền `'read'` chỉ mở dashboard,
không mở các miền khác.

Các miền: `read`, `hero`, `shop`, `player`, `giftcode`, `analytics`, `ruleset`, `audit`,
`admin`, cùng wildcard `'*'`.

`Sidebar.tsx` lọc menu theo `hasPermission(permissions, item.domain)` — ẩn/hiện để
tiện, **không phải** cơ chế bảo mật.

## 8. Error handling

| Tình huống | Xử lý |
|---|---|
| Lỗi mạng / 5xx | `ErrorState` với nút "Thử lại" (gọi `reload()` của hook). |
| 403 | `ErrorState` hiện "Bạn không có quyền xem phần này", **không** hiện nút thử lại. |
| 401 | Refresh 1 lần → thử lại; thất bại → logout → chuyển về `/login`. |
| Dữ liệu rỗng | `EmptyState` với tiêu đề/mô tả tuỳ ngữ cảnh. |
| Đang tải | `Skeleton` / `TableSkeleton` / `CardSkeleton`. |
| Thao tác ghi thất bại | `useToast().notify(message, 'danger')`. |
| Thao tác ghi thành công | `useToast().notify(message, 'ok')` (tự ẩn sau 5s). |

`src/components/ui/ErrorState.tsx` phân biệt `ApiError.isPermission` (403) với lỗi hệ
thống, và hiện `apiError.code` (mã lỗi nghiệp vụ của backend) bằng font mono.

## 9. Shared components

### `components/ui/`

| Component | Vai trò |
|---|---|
| `Button` | Nút có `variant` (`primary` / `secondary` / `danger` / `ghost`), `loading`, `icon`. |
| `Card` | Khung panel với `title`, `description`, `actions`. |
| `Badge` | Nhãn trạng thái + `statusTone` map màu theo trạng thái. |
| `Skeleton` / `TableSkeleton` / `CardSkeleton` | Placeholder khi đang tải. |
| `Field` | Nhãn + control (`useId` để gắn `htmlFor`/`id` — accessibility). |
| `ConfirmDialog` | Hộp thoại xác nhận cho thao tác ghi. |
| `Drawer` | Panel trượt từ bên phải, dùng cho chi tiết/form. |
| `Toast` / `ToastProvider` / `useToast` | Thông báo kết quả thao tác. |
| `JsonViewer` | Hiển thị JSON thô (`request`/`events` của match action, `before`/`after` của audit). |
| `EmptyState` | Trạng thái rỗng. |
| `ErrorState` | Trạng thái lỗi + nút thử lại. |

### `components/layout/`

- `AppLayout` — sidebar cố định (desktop ≥ `lg`), sidebar overlay (tablet/mobile),
  `Header`, `<Outlet />`. Tiêu đề header suy ra từ `location.pathname`.
- `Sidebar` — menu 9 mục, ẩn/hiện theo quyền.
- `Header` — tiêu đề trang, nút menu (mobile), nút đăng xuất, monogram tài khoản.

### `components/heroes|players|matches/`

- `HeroForm` — tạo/sửa hero.
- `HeroDetailDrawer` — chi tiết hero + cosmetics + skills + lịch sử đổi giá.
- `PlayerDetailDrawer` — hồ sơ người chơi, lịch sử ví, cấp/thu hero.
- `MatchDetailDrawer` — chi tiết trận, lineup, chuỗi action (JSON viewer).

## 10. Quy ước dữ liệu quan trọng

| Quy ước | Cách làm |
|---|---|
| Tiền (`coinPrice`, `balance`, `amount`) | Backend trả **chuỗi**; `formatCoin()` nhóm nghìn bằng regex, **không** ép `Number` — giữ đúng giá trị trên 2^53. |
| Phần trăm | Backend trả thang **0–100**; `formatPercent()` hiển thị thẳng, không nhân lần nữa. |
| Thiếu dữ liệu | `null` → hiển thị "chưa đủ dữ liệu" / `—`, không hiện `0%`. |
| Phân trang | **Keyset cursor**: `{ items, nextCursor, total }`; `usePagedResource` nối chuỗi cursor, không dùng offset. `nextCursor === null` → hết trang. |
| Lớp quân | Nguồn duy nhất là `GET /admin/chess-classes` (đọc bảng `hero_chess.chess_class`). `FALLBACK_CLASSES` trong `classLabel.ts` chỉ dùng khi endpoint lỗi. Hero mới mặc định lớp `ADVISOR` (`DEFAULT_CLASS_CODE`), không có dropdown chọn lớp. |
| `setupPoints` | Legacy read-only — API trả về để hiển thị, không có luật gameplay nào enforce. |
| `reason` | Bắt buộc cho các thao tác ghi nhạy cảm; client validate trước khi gửi. |
| `hasReplay` | Luôn `false` — UI không hiện nút replay. |

## 11. Tích hợp Backend

- Base URL tương đối `/api/v1` → dev đi qua Vite proxy, production đi qua cùng-origin
  (reverse proxy) hoặc static hosting có cấu hình rewrite tương ứng.
- Backend **không bật CORS** → tuyệt đối không chuyển sang gọi cross-origin từ browser.
- AdminWeb **không** gọi Supabase Data API, **không** có service-role key, **không**
  sửa coin/ownership phía client.
- Chi tiết endpoint: [`API_INTEGRATION.md`](./API_INTEGRATION.md).

## 12. Tính năng đã triển khai (kiểm chứng từ source)

| Trang | Nội dung thực thi |
|---|---|
| Login | Gọi `POST /auth/login?useCookies=false`, lưu phiên, chuyển hướng về trang đã định. |
| Dashboard | `GET /admin/stats` (6 KPI) + `GET /admin/analytics/balance?mode=` + `GET /admin/analytics/bot` (chỉ khi có quyền `analytics`). |
| Hero | Danh sách keyset + tìm kiếm `query` + lọc `classCode`/`factionCode`/`enabled`/`inShop`; tạo (`POST /admin/heroes`), sửa (`PATCH /admin/heroes/{id}`), bật/tắt kèm `reason`; drawer chi tiết + lịch sử giá; gán phe (`PUT /admin/heroes/{id}/faction`). |
| Cửa hàng | Danh sách hero trong shop (`GET /admin/shop/heroes`), đổi giá/bật-tắt bán (`PATCH /admin/shop/heroes/{id}`), đổi giá hàng loạt (`PATCH /admin/shop/prices`), lịch sử mua (`GET /admin/shop/purchases`). |
| Người chơi | Danh sách + tìm kiếm + lọc `status`; khoá/mở khoá (`PATCH /admin/players/{id}/status`); cộng/trừ coin (`POST /admin/players/{id}/coins`); cấp/thu hero (`POST`/`DELETE /admin/players/{id}/heroes[...]`); drawer chi tiết + lịch sử ví (`GET /admin/players/{id}/transactions`). |
| Gift Code | Danh sách + tìm kiếm + lọc `active`; tạo (`POST /admin/gift-codes`, để trống `code` để backend tự sinh), sửa cấu hình (`PATCH /admin/gift-codes/{id}`), bật/tắt; drawer chi tiết + lịch sử đổi. |
| Trận đấu | Danh sách + lọc `mode`/`status`/`from`/`to`; drawer chi tiết (`GET /admin/matches/{id}`) + chuỗi action (`GET /admin/matches/{id}/actions?limit=200`). |
| Phân tích | `GET /admin/analytics/balance` (tỉ lệ chọn/thắng theo hero, phe, skill) + `GET /admin/analytics/bot` (theo độ khó, theo ruleset). |
| Tài khoản Admin | Danh sách tài khoản (`GET /admin/accounts`), catalogue quyền (`GET /admin/permissions`), tạo tài khoản (`POST /admin/accounts`), đặt lại quyền (`PUT /admin/accounts/{id}/permissions`). |
| Nhật ký thao tác | `GET /admin/audit` + lọc `action`/`entityType`/`entityId`/`actorUserId`/`from`/`to`; hiển thị `before`/`after` bằng `JsonViewer`. |

Không có chức năng nào trong bảng trên được tuyên bố là "đã xong" nếu chưa có code tương ứng
trong `src/`.
