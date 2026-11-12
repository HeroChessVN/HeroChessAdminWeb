# AdminWeb — Triển khai production

## 1. Build

```bash
npm ci
npm run build      # = tsc -b && vite build
```

Kết quả: thư mục `dist/` gồm `index.html` + `assets/*.js` + `assets/*.css`.

- `npm run build` **chạy typecheck trước** → build fail nếu có lỗi TypeScript.
- `dist/` được `.gitignore` loại — không commit output build.

### 1.1. Kiểm tra nhanh output

```bash
npm run preview    # vite preview --port 5175
```

`vite preview` phục vụ `dist/` với `base` tương ứng. Lưu ý: `preview` **không** có
proxy `/api/v1` (chỉ `server.proxy` của dev server mới có), nên API sẽ trả 404 khi
preview — đó là hành vi đúng, dùng để kiểm tra asset/base path chứ không dùng để test
tích hợp API.

## 2. `base` path — điểm cần quyết định khi tách repository

`vite.config.ts`:

```ts
base: command === 'build' ? '/admin/' : '/',
```

Tức là:

| Lệnh | `base` | Asset URL trong `dist/index.html` |
|---|---|---|
| `npm run dev` | `/` | (dev server phục vụ trực tiếp) |
| `npm run build` | `/admin/` | `/admin/assets/index-<hash>.js` |

### 2.1. Vì sao `/admin/`

Lịch sử deploy của dự án: AdminWeb được build rồi đặt vào `wwwroot/admin/` của
ASP.NET Core (`app.UseStaticFiles()` phục vụ `wwwroot`). `base = '/admin/'` khớp với
cách phục vụ đó → asset được tham chiếu đúng.

**HashRouter** là lý do không cần SPA fallback: route nằm sau `#`
(`/admin/#/heroes`), nên server chỉ cần phục vụ `index.html` tại `/admin/`.

### 2.2. Các phương án hosting khi AdminWeb tách thành repo riêng

Chọn **một** phương án và cập nhật `base` cho khớp. **Chưa thay đổi `base` trong repo
này** — cần được phê duyệt trước khi deploy thật.

| Phương án | `base` cần đặt | Điều kiện hosting | Ghi chú |
|---|---|---|---|
| **A. Serve từ `wwwroot/admin/` của ASP.NET** (hiện tại) | `/admin/` | Copy `dist/*` → `HeroChess.Api/wwwroot/admin/` | Backend không có SPA fallback nhưng không cần (HashRouter). |
| **B. Host tĩnh riêng (Nginx / CDN / GitHub Pages)** | `/` (root của domain) hoặc `/admin/` nếu mount vào subpath | Reverse proxy `location /api/v1 { proxy_pass http://backend/api/v1; }` | Cần reverse proxy để `/api/v1` về backend — không bật CORS ở backend. |
| **C. Sub-path trên domain chính** | `/admin/` | Nginx: `location /admin/ { alias /var/www/admin-web/; }` + `location /api/v1 { proxy_pass ...; }` | Khớp cấu hình hiện tại, ít thay đổi nhất. |
| **D. Subdomain riêng** (`admin.example.com`) | `/` | Nginx/Caddy: `root /var/www/admin-web;` + `location /api/v1 { proxy_pass http://backend/api/v1; }` | `base = '/'`; URL API vẫn `/api/v1` nên cần proxy trên **cùng** origin của subdomain. |

### 2.3. Điều kiện bắt buộc cho mọi phương án

1. **`/api/v1` phải được phục vụ từ cùng origin với AdminWeb**, và trỏ tới
   HeroChess API. Vì backend không bật CORS, đây là điều kiện bắt buộc.
2. **`index.html` phải được trả về** ở đường dẫn gốc của AdminWeb (hash routing không
   cần rewrite).
3. **Không** đưa bất kỳ secret nào vào biến `VITE_*` — chúng được nhúng vào bundle
   công khai.
4. **`index.html` đã có `<meta name="robots" content="noindex, nofollow">`** → console
   admin không bị index. Giữ nguyên khi deploy.

### 2.4. Cách đổi `base` (khi được duyệt)

`vite.config.ts`:

```ts
base: command === 'build' ? '/' : '/',
```

rồi rebuild. Nhớ cập nhật bảng ở § 2.2 và `README.md` cho khớp.

## 3. Reverse proxy mẫu (Nginx)

```nginx
server {
    listen 80;
    server_name admin.example.com;

    root /var/www/admin-web;
    index index.html;

    # AdminWeb là SPA hash-routing: /admin/assets/... phục vụ thẳng từ disk.
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Backend KHÔNG bật CORS → phải proxy cùng-origin.
    location /api/v1/ {
        proxy_pass http://127.0.0.1:5012/api/v1/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 4. Checklist trước khi deploy

- [ ] `npm ci` chạy sạch (không lỗi dependency).
- [ ] `npm run typecheck` — 0 lỗi.
- [ ] `npm test` — tất cả pass.
- [ ] `npm run build` — thành công, `dist/` có `index.html` + `assets/`.
- [ ] `base` trong `dist/index.html` khớp với đường dẫn serve thật.
- [ ] `/api/v1/auth/login?useCookies=false` trả 200 từ origin của AdminWeb.
- [ ] Không có secret nào trong `dist/assets/*.js` (grep cho `service_role`, `SUPABASE`,
      `PRIVATE KEY`, `eyJhbGciOi` token cứng).
- [ ] Đăng nhập thành công bằng tài khoản admin thật.
- [ ] Các route chính tải được dữ liệu thật (Dashboard, Hero, Cửa hàng, Người chơi,
      Gift Code, Trận đấu, Phân tích, Tài khoản Admin, Nhật ký).
- [ ] Một thao tác ghi có `reason` chạy thành công và xuất hiện trong nhật ký thao tác.
- [ ] Deep-link `/#/heroes` tải được (HashRouter).

## 5. Kiểm tra secret trong bundle

```bash
grep -rniE "service_role|SUPABASE_SERVICE|private key|BEGIN RSA|eyJhbGciOi" dist/assets/ || echo "OK: không thấy secret"
```

## 6. Quay lui

Bản deploy là file tĩnh — quay lui chỉ cần copy lại bản `dist/` cũ (hoặc bản build từ
commit trước). Không có migration database nào liên quan tới AdminWeb.
