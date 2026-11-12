# AdminWeb — Quy trình phát triển

Tài liệu cho developer làm việc hằng ngày trên AdminWeb.

## 1. Yêu cầu môi trường

| Công cụ | Phiên bản | Ghi chú |
|---|---|---|
| Node.js | ≥ 20.19 (khuyến nghị 22/24) | Vite 7 yêu cầu Node `^20.19.0 \|\| >=22.12.0` |
| npm | ≥ 10 | Dùng `npm ci` khi có `package-lock.json` |

Không cần .NET SDK để phát triển AdminWeb — AdminWeb là SPA tĩnh, chỉ gọi API qua HTTP.

## 2. Cài đặt

```bash
cd admin-web
npm ci          # cài đúng theo package-lock.json (khuyến nghị cho CI)
# hoặc
npm install     # khi cần cập nhật dependency
```

Không copy `node_modules` giữa các máy — nó chứa binary native theo OS.

## 3. Cấu hình môi trường

```bash
cp .env.example .env      # Windows PowerShell: Copy-Item .env.example .env
```

`.env` được `.gitignore` loại — không bao giờ commit.

| Biến | Đọc ở đâu | Được expose ra bundle? | Mặc định |
|---|---|---|---|
| `HERO_CHESS_API_URL` | `vite.config.ts` (`process.env`) | **Không** — chỉ dev server dùng làm target của proxy | `http://localhost:5012` |
| `VITE_ADMIN_API_URL` | `src/lib/api.ts` (`import.meta.env`) | **Có** — nhúng thẳng vào JS | `/api/v1` |

Quy tắc an toàn: mọi biến `VITE_*` đều **public**. Không bao giờ đặt secret, service-role
key, JWT key hay access token vào biến `VITE_*`.

### 3.1. Vì sao không đặt `VITE_ADMIN_API_URL` thành URL tuyệt đối ở local

Backend **không bật CORS**. Nếu client gọi `http://localhost:5012/api/v1` trực tiếp từ
trình duyệt (origin `http://127.0.0.1:5175`) thì request sẽ bị chặn. Vì vậy:

- Local: giữ `VITE_ADMIN_API_URL=/api/v1` → request đi qua Vite proxy (cùng-origin).
- Production: cũng giữ `/api/v1` và để reverse proxy / hosting rewrite `/api/v1` về backend.

Chỉ đổi `HERO_CHESS_API_URL` khi backend của bạn chạy ở cổng/host khác.

## 4. Chạy dev server

```bash
npm run dev
```

- URL: <http://127.0.0.1:5175/>
- `--host 127.0.0.1` → chỉ bind loopback (không expose ra LAN).
- HashRouter → mở <http://127.0.0.1:5175/#/login>.

Nếu port 5175 đã bận, Vite sẽ báo và hỏi dùng cổng khác — hãy dừng process cũ thay vì
đổi cổng trong `package.json` (cổng 5175 là quy ước của dự án).

## 5. Kiểm tra chất lượng

| Lệnh | Script | Kiểm tra gì |
|---|---|---|
| `npm run typecheck` | `tsc -b --pretty` | TypeScript (app + node config). |
| `npm run build` | `tsc -b && vite build` | Typecheck **rồi** build production → `dist/`. |
| `npm test` | `node --test "src/lib/*.test.ts"` | Unit test cho `format.ts` + `permissions.ts`. |

> **Không có script lint** trong `package.json` hiện tại (không có ESLint/Prettier được
> cài đặt). Nếu muốn thêm lint, cần cài thêm dependency — chưa làm trong giai đoạn này.

Test file được **loại khỏi** `tsconfig.app.json` (`exclude: ["src/**/*.test.ts"]`) vì
`node --test` tự strip type (Node ≥ 22.6) và không đi qua build của app.

## 6. Cấu trúc mã nguồn — quy ước

| Quy ước | Thực hiện |
|---|---|
| Mỗi file bắt đầu bằng comment `// Vai trò file: …` | Mô tả 1 dòng vai trò của file. |
| Comment giải thích **vì sao**, không **cái gì** | Ví dụ: "tiền là chuỗi vì backend trả bigint". |
| Tiền (`bigint`) luôn là chuỗi | `formatCoin()` nhóm nghìn bằng regex, không ép `Number`. |
| Phân trang bằng keyset cursor | `usePagedResource`, không offset. |
| Lỗi 403 khác lỗi hệ thống | `ApiError.isPermission` → `ErrorState` hiện "không có quyền". |
| Mọi thao tác ghi nhạy cảm cần `reason` | Validate ở client trước khi gửi. |
| Không giả lập dữ liệu | Số liệu/biểu đồ lấy từ API; thiếu dữ liệu → `EmptyState`. |
| Không hardcode danh sách lớp quân | Nguồn: `GET /admin/chess-classes`. `FALLBACK_CLASSES` chỉ là fallback khi endpoint lỗi. |

## 7. Thêm một trang mới (quy trình)

1. Tạo component trong `src/pages/<Tên>Page.tsx`.
2. Đăng ký route trong `src/App.tsx`, bọc bằng `<ProtectedRoute domain={PERMISSION_...}>`.
3. Thêm mục menu trong `src/components/layout/Sidebar.tsx` (items có `domain`).
4. Thêm tiêu đề header trong `titleFor()` ở `src/components/layout/AppLayout.tsx`.
5. Thêm type DTO vào `src/lib/types.ts` nếu cần (bám 1-1 với `HeroChess.Contracts/AdminContracts.cs`).
6. Thêm endpoint vào `docs/API_INTEGRATION.md`.

## 8. Kiểm thử thủ công

- Mở <http://127.0.0.1:5175/#/login> và đăng nhập bằng tài khoản admin.
- Kiểm tra từng route trong bảng ở [`ARCHITECTURE.md`](./ARCHITECTURE.md) § 4.
- Kiểm tra trạng thái lỗi: dừng backend → các trang phải hiện `ErrorState` + nút "Thử lại",
  không phải màn trắng.
- Ảnh tham chiếu UI: [`../ui-audit/screenshots`](../ui-audit/screenshots).

## 9. Vấn đề thường gặp

| Triệu chứng | Nguyên nhân | Xử lý |
|---|---|---|
| Trang trắng, console báo lỗi import | Chạy `npm ci` lại (thiếu dependency) | `npm ci` |
| API trả 404 / ECONNREFUSED | Backend chưa chạy, hoặc `HERO_CHESS_API_URL` sai | Kiểm tra <http://127.0.0.1:5012/health/ready>; chạy lại `npm run dev` sau khi đổi env |
| Lỗi CORS trong console | Đang gọi URL tuyệt đối | Đặt lại `VITE_ADMIN_API_URL=/api/v1` |
| Lỗi 403 "Bạn không có quyền xem phần này" | Tài khoản thiếu quyền miền, hoặc endpoint cần policy khác | Kiểm tra quyền trong trang Tài khoản Admin |
| Đăng nhập xong bị văng ra `/login` | Token hết hạn ngay / refresh hỏng | Xem log backend; kiểm tra `sessionStorage` trong DevTools |
| `npm test` báo `MODULE_NOT_FOUND` | Chạy sai mẫu file | Script phải là `node --test "src/lib/*.test.ts"` |
| Build ra `/admin/assets/...` mà không load | Deploy sai `base` path | Xem [`DEPLOYMENT.md`](./DEPLOYMENT.md) |
| `npm warn Unknown env config "devdir"` | Cấu hình `devdir` trong `.npmrc` của máy, không liên quan project | Bỏ qua hoặc xoá khỏi `.npmrc` máy |

## 10. Liên kết

- [`ARCHITECTURE.md`](./ARCHITECTURE.md) — routes, API client, auth, component.
- [`API_INTEGRATION.md`](./API_INTEGRATION.md) — endpoint thực gọi.
- [`DEPLOYMENT.md`](./DEPLOYMENT.md) — build & deploy production.
