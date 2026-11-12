# AdminWeb — Tích hợp API

> **Nguồn:** bản rút gọn từ `HeroChessBackend/docs/admin/ADMIN_API_GUIDE.md` (Backend
> repository) và đối chiếu lại với source `admin-web/src/`.
> **Nguồn sự thật của contract API là Backend repository.** Nếu tài liệu này lệch với
> backend, hãy sửa ở Backend rồi cập nhật lại bản tham khảo ở đây.
> Tài liệu này chỉ liệt kê **endpoint AdminWeb thực sự gọi** — không phải toàn bộ
> Admin API (bản đầy đủ nằm ở Backend).

Base URL của AdminWeb: **`/api/v1`** (đường dẫn tương đối).

- Dev: đi qua Vite proxy → `HERO_CHESS_API_URL` (mặc định `http://localhost:5012`).
- Production: đi qua cùng-origin (reverse proxy) hoặc rewrite tương ứng.
- Backend **không bật CORS** → không được đổi sang URL tuyệt đối ở production.

---

## 1. Xác thực

### 1.1 Đăng nhập

```http
POST /api/v1/auth/login?useCookies=false
Content-Type: application/json

{ "email": "…", "password": "…" }
```

```json
{ "tokenType": "Bearer", "accessToken": "…", "expiresIn": 3600, "refreshToken": "…" }
```

AdminWeb dùng `useCookies=false` để nhận token trong body (không dùng cookie).

### 1.2 Làm mới token

```http
POST /api/v1/auth/refresh
Content-Type: application/json

{ "refreshToken": "…" }
```

### 1.3 Profile

```http
GET /api/v1/me
Authorization: Bearer <accessToken>
```

### 1.4 Catalogue quyền

```http
GET /api/v1/admin/permissions
```

Dùng để dựng form phân quyền trong trang Tài khoản Admin.

### 1.5 Danh sách tài khoản admin (nguồn quyền của chính mình)

```http
GET /api/v1/admin/accounts
```

```json
{ "items": [{ "id": "…", "displayName": "ops1", "email": "…", "status": "active",
              "permissions": ["read", "player", "giftcode"], "createdAt": "…" }] }
```

> Backend **không** có endpoint trả quyền của chính mình. AdminWeb dùng endpoint này
> và tự lọc theo `id` của `GET /me`. Endpoint cần policy `admin-read:admin`, nên admin
> bị giới hạn sẽ nhận 403 → `permissions = null` (chưa xác định) → UI hiển thị đầy đủ,
> server vẫn là nơi quyết định thật.

### 1.6 Mã lỗi

| HTTP | Ý nghĩa | Xử lý ở AdminWeb |
|---|---|---|
| 401 | Chưa đăng nhập / token hết hạn | Refresh 1 lần → thử lại; thất bại → logout → `/login`. |
| 403 | Đã đăng nhập nhưng thiếu quyền miền | `ErrorState` hiện "không có quyền", không hiện nút thử lại. |
| 400 | Payload sai | Hiện `message` trong `ErrorState` / toast. |
| 404 | Không tìm thấy entity | Sames. |
| 409 | Xung đột nghiệp vụ | Sames. |
| 422 | Dữ liệu tham chiếu không tồn tại | Sames. |

Response lỗi: `{ "code": "HERO_NOT_FOUND", "message": "The hero was not found." }` —
`code` được hiển thị bằng font mono trong `ErrorState`.

---

## 2. Phân trang (keyset cursor)

Mọi danh sách dùng **keyset cursor**, không offset:

```json
{ "items": [ ... ], "nextCursor": "eyJ...", "total": 123 }
```

- `nextCursor === null` → hết trang.
- Trang sau: `?cursor=<nextCursor>&limit=<n>`.
- AdminWeb dùng `usePagedResource` (`src/lib/useResource.ts`) để nối chuỗi cursor.

---

## 3. Endpoint AdminWeb thực sự gọi

### 3.1 Dashboard & phân tích

| Method | Path | Quyền | Dùng ở |
|---|---|---|---|
| GET | `/admin/stats` | `read` | `DashboardPage` |
| GET | `/admin/analytics/balance?mode=ranked\|bot` | `analytics` | `DashboardPage`, `AnalyticsPage` |
| GET | `/admin/analytics/bot` | `analytics` | `DashboardPage`, `AnalyticsPage` |

> `winRate` / `pickRate` / `firstMoveAdvantage.value` là `null` khi chưa đủ dữ liệu —
> AdminWeb hiển thị "chưa đủ dữ liệu", không hiện `0%`.
> Thang phần trăm của backend là **0–100**.

### 3.2 Hero

| Method | Path | Quyền | Dùng ở |
|---|---|---|---|
| GET | `/admin/heroes?query=&classCode=&factionCode=&enabled=&inShop=&cursor=&limit=50` | `hero` | `HeroesPage` |
| GET | `/admin/heroes/{id}` | `hero` | `HeroDetailDrawer` |
| POST | `/admin/heroes` | `hero` | `HeroForm` (tạo) |
| PATCH | `/admin/heroes/{id}` | `hero` | `HeroForm` (sửa), `HeroesPage` (bật/tắt) |
| PUT | `/admin/heroes/{id}/faction` | `hero` | `HeroDetailDrawer` (gán phe) |
| GET | `/admin/factions` | `hero` | `HeroesPage`, `HeroDetailDrawer` |
| GET | `/admin/chess-classes` | `hero` | `src/lib/chessClasses.ts` |

### 3.3 Cửa hàng

| Method | Path | Quyền | Dùng ở |
|---|---|---|---|
| GET | `/admin/shop/heroes?query=&inShop=&cursor=&limit=50` | `shop` | `ShopPage` |
| PATCH | `/admin/shop/heroes/{id}` | `shop` | `ShopPage` (đổi giá / bật-tắt bán) |
| PATCH | `/admin/shop/prices` | `shop` | `ShopPage` (đổi giá hàng loạt) |
| GET | `/admin/shop/purchases?cursor=&limit=25` | `shop` | `ShopPage` |

### 3.4 Người chơi

| Method | Path | Quyền | Dùng ở |
|---|---|---|---|
| GET | `/admin/players?query=&status=&cursor=&limit=50` | `player` | `PlayersPage` |
| GET | `/admin/players/{id}` | `player` | `PlayerDetailDrawer` |
| GET | `/admin/players/{id}/transactions` | `player` | `PlayerDetailDrawer` |
| PATCH | `/admin/players/{id}/status` | `player` | `PlayersPage` (khoá / mở khoá) |
| POST | `/admin/players/{id}/coins` | `player` | `PlayersPage` (cộng / trừ coin) |
| POST | `/admin/players/{id}/heroes` | `player` | `PlayerDetailDrawer` (cấp hero) |
| DELETE | `/admin/players/{id}/heroes/{heroId}` | `player` | `PlayerDetailDrawer` (thu hero) |

> `PlayerDetailDrawer` còn gọi `GET /admin/heroes?limit=200` để lấy danh sách hero
> dùng cho form cấp hero.

### 3.5 Gift code

| Method | Path | Quyền | Dùng ở |
|---|---|---|---|
| GET | `/admin/gift-codes?query=&active=&cursor=&limit=50` | `giftcode` | `GiftCodesPage` |
| POST | `/admin/gift-codes` | `giftcode` | `GiftCodesPage` (tạo; `code` rỗng → backend tự sinh) |
| PATCH | `/admin/gift-codes/{id}` | `giftcode` | `GiftCodesPage` (sửa cấu hình / bật-tắt) |
| GET | `/admin/gift-codes/{id}/redemptions?limit=100` | `giftcode` | `GiftCodesPage` (drawer lịch sử đổi) |

### 3.6 Trận đấu

| Method | Path | Quyền | Dùng ở |
|---|---|---|---|
| GET | `/admin/matches?mode=&status=&cursor=&limit=50` | `analytics` | `MatchesPage` |
| GET | `/admin/matches/{id}` | `analytics` | `MatchDetailDrawer` |
| GET | `/admin/matches/{id}/actions?limit=200` | `analytics` | `MatchDetailDrawer` |

> `hasReplay` luôn `false` — AdminWeb không hiện nút replay.
> `request` / `events` của action là JSON thô → hiển thị bằng `JsonViewer`.

### 3.7 Nhật ký thao tác

| Method | Path | Quyền | Dùng ở |
|---|---|---|---|
| GET | `/admin/audit?action=&entityType=&entityId=&actorUserId=&from=&to=&cursor=&limit=25` | `audit` | `AuditPage` |

`from` / `to` được AdminWeb gửi ở dạng ISO (`new Date(...).toISOString()`).
`before` / `after` hiển thị bằng `JsonViewer`.

### 3.8 Tài khoản admin

| Method | Path | Quyền | Dùng ở |
|---|---|---|---|
| GET | `/admin/accounts` | `admin` | `AdminAccountsPage`, `AuthContext` |
| POST | `/admin/accounts` | `admin` | `AdminAccountsPage` (tạo) |
| PUT | `/admin/accounts/{id}/permissions` | `admin` | `AdminAccountsPage` (đặt lại quyền) |

---

## 4. Quy tắc ghi dữ liệu bắt buộc

| Quy tắc | Áp dụng cho |
|---|---|
| `reason` bắt buộc | Ban/mở khoá người chơi, cộng/trừ coin, cấp/thu hero, tạo/sửa gift code, bật/tắt hero, bật/tắt gift code, gán phe, đổi giá hàng loạt. Client validate trước khi gửi. |
| Confirm dialog | Ban tài khoản, điều chỉnh coin, thu hero, tắt gift code, đổi giá hàng loạt, cấp/thu quyền admin. |
| Tiền gửi dạng chuỗi | `coinPrice`, `amount`, `balance`. Không ép `Number` trước khi hiển thị. |
| `setupPoints` | Chỉ lớp `GENERAL` được `0`; lớp khác cần ≥ 1. Vi phạm → `400 INVALID_SETUP_POINTS`. |
| `reason` của `PUT /admin/accounts/{id}/permissions` | Thay **toàn bộ** danh sách quyền. `["*"]` = toàn quyền. |

---

## 5. Những gì AdminWeb không làm

| Không làm | Vì sao |
|---|---|
| Gọi Supabase / Data API trực tiếp | Schema đã `REVOKE` cho `anon`/`authenticated`; mọi truy cập phải qua API có authorization. |
| Đưa service-role key vào browser | Rò rỉ toàn quyền DB. |
| Tự sửa coin / hero ownership phía client | Phải qua endpoint admin có transaction + audit. |
| Xoá hero / xoá tài khoản người chơi | Phá lịch sử trận. Chỉ soft-disable. |
| Hiển thị deterministic replay | `hasReplay` luôn `false`. |
| Hiện win-rate khi chưa đủ dữ liệu | Giá trị là `null`; phải hiện "chưa đủ dữ liệu". |
| Áp dụng luật Setup Points | Legacy read-only, chưa được duyệt. |
| Tự tính số liệu thay vì gọi API | Analytics đã tính sẵn từ dữ liệu thật. |
| Bật CORS ở Backend | AdminWeb dùng proxy / same-origin. |
