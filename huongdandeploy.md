# VERT Fork — verttools

Fork riêng của [VERT](https://github.com/VERT-sh/VERT), deploy trên Cloudflare Pages. README này ghi lại toàn bộ thay đổi so với bản gốc và cách build lại từ đầu, để không phải mò lại từ đầu lần sau.

**Site đang chạy:** `https://verttools.pages.dev`

---

## Thay đổi so với bản gốc VERT-sh/VERT

### 1. Đã tắt tính năng convert video

> Chỉ đổi 1 biến môi trường, **không sửa dòng code nào** trong `src/lib/converters/index.ts`. Đoạn code dưới đây chỉ trích ra để minh họa cơ chế có sẵn từ bản gốc, cho dễ hiểu vì sao đổi env lại tắt được video.

Video convert cần server `vertd` riêng (server gốc: `vertd.vert.sh`). Không dùng server đó để tránh phụ thuộc bên ngoài, và cũng không tự host `vertd` (cần Rust/Docker, không có VPS).

**Cách tắt:** set biến môi trường `PUB_DISABLE_ALL_EXTERNAL_REQUESTS=true`. Biến này được check trong `src/lib/converters/index.ts`:

```ts
if (!DISABLE_ALL_EXTERNAL_REQUESTS) {
    converters.push(new VertdConverter());
}
```

Set `true` → `VertdConverter` không được đăng ký → video convert biến mất khỏi UI.

Tác dụng phụ (chấp nhận được): biến này cũng tắt luôn donate/Stripe button và Plausible analytics, vì tên biến là "disable ALL external requests", không riêng gì video.

### 2. `pandoc.wasm` chuyển sang load từ Cloudflare R2, không bundle vào build

**Vấn đề gốc:** file `pandoc.wasm` nặng 50.4MB. Cloudflare Pages (và Workers Static Assets) giới hạn cứng **25MB/file** — build luôn fail ở bước deploy assets nếu để nguyên trong `static/`.

**Đã thử và không work:**
- Cloudflare Workers — giới hạn 25MB/file y hệt Pages, không giải quyết được gì.
- jsDelivr CDN (`cdn.jsdelivr.net/gh/...`) — giới hạn file nguồn từ GitHub tối đa 20MB, còn thấp hơn cả Cloudflare, không dùng được cho file 50.4MB này.

**Giải pháp cuối cùng:** upload `pandoc.wasm` lên Cloudflare R2, serve qua Public Development URL, sửa code fetch từ đó thay vì file local.

Trong `src/lib/converters/pandoc.svelte.ts`, đổi:

```ts
// Cũ — load file local, bundle vào build, gây lỗi 25MB
this.wasm = await fetch("/pandoc.wasm").then((r) => r.arrayBuffer());
```

thành:

```ts
// Mới — load từ R2 lúc runtime, không bundle vào build
this.wasm = await fetch(
    "https://pub-96f1f57023f14b549baaa0cd62042149.r2.dev/pandoc.wasm",
).then((r) => r.arrayBuffer());
```

Và xóa hẳn file `static/pandoc.wasm` khỏi repo sau khi đã xác nhận link R2 hoạt động — quy trình đầy đủ theo đúng thứ tự xem ở Bước 3 phần checklist bên dưới.

**Bắt buộc phải bật CORS trên bucket R2**, nếu không sẽ lỗi `TypeError: Failed to fetch` khi trình duyệt gọi `fetch()` tới domain khác (dù link vẫn mở được trực tiếp bằng cách bấm vào — CORS chỉ chặn `fetch()` qua JS, không chặn truy cập trực tiếp qua trình duyệt).

Cấu hình CORS đã thêm ở R2 bucket → Settings → CORS Policy:

```json
[
  {
    "AllowedOrigins": ["https://verttools.pages.dev"],
    "AllowedMethods": ["GET"],
    "AllowedHeaders": ["*"],
    "MaxAgeSeconds": 3600
  }
]
```

### 3. Build output directory phải set đúng `build`, không phải mặc định

SvelteKit dùng `@sveltejs/adapter-static` — output thực tế nằm ở thư mục `build` (log build có dòng `Wrote site to "build"`), **không phải** `.svelte-kit/output/client` như Cloudflare Pages tự set mặc định khi tạo project. Nếu để mặc định, build báo Success nhưng domain trả về 404 vì Cloudflare tìm `index.html` sai chỗ.

**Đã sửa ở:** Cloudflare Pages → Settings → Build → Build configuration → Build output directory = `build`.

### 4. Svelte 5 runes — không destructure `$props()` ra biến trung gian

Đã sửa 2 file sau, vì bản gốc destructure props qua biến trung gian:

- `src/lib/components/functional/Dialog.svelte` — **sửa triệt để**, destructure trực tiếp ngay tại `$props()`, hết warning hoàn toàn.
- `src/lib/components/visual/Toast.svelte` — **chỉ sửa một phần**. Đổi biến trung gian từ `props.toast` sang `toast`, nhưng vẫn destructure `{ id, type, message, durations }` từ `toast` ở dòng sau, nên bản chất vẫn là destructure qua biến trung gian. Warning `state_referenced_locally` **vẫn còn nguyên** sau khi sửa, thấy rõ trong log build cuối cùng. Không sửa tiếp vì đây chỉ là warning, không chặn build — để dành sửa sau nếu rảnh.

Kiểu sai gốc (cả 2 file):

```ts
let props = $props();
const { id, title } = props; // ❌ lỗi state_referenced_locally
```

Svelte 5 chỉ capture giá trị tại thời điểm khởi tạo nếu làm vậy. Cách đúng — destructure trực tiếp ngay tại lệnh gọi `$props()`:

```ts
const { id, title } = $props(); // ✅
```

**Cách sửa triệt để cho `Toast.svelte`** (chưa áp dụng, để dành nếu sau này muốn dọn sạch warning): dùng `$derived` cho từng field thay vì destructure thường, vì `toast` là 1 object prop có thể đổi theo thời gian sống của component:

```ts
const { toast }: { toast: ToastType<unknown> } = $props();
const id = $derived(toast.id);
const type = $derived(toast.type);
const message = $derived(toast.message);
const durations = $derived(toast.durations);
const additional = $derived("additional" in toast ? (toast as any).additional : {});
```

Lưu ý: đây chỉ là **warning**, không phải lỗi cứng — build vẫn pass dù còn cảnh báo này. Không bắt buộc phải sửa, nhưng nên sửa cho sạch nếu có thời gian.

### 5. `package.json` — downgrade `@stripe/stripe-js` từ `^8.5.2` xuống `^4.0.0`

Code gốc VERT khai `@stripe/stripe-js@^8.5.2`, nhưng `svelte-stripe@1.4.0` (thư viện wrap Stripe cho Svelte) chỉ chấp nhận `@stripe/stripe-js@^3 || ^4` làm peer dependency. Bản 8.x không nằm trong range đó.

`npm install` mặc định trên Cloudflare Pages chạy chế độ strict — thấy peer dependency không khớp là chặn cài luôn (`ERESOLVE unable to resolve dependency tree`), không tự động cài liều rồi hy vọng chạy được như một số package manager khác.

**Đã sửa:** trong `package.json`, đổi:

```json
"@stripe/stripe-js": "^8.5.2"
```

thành:

```json
"@stripe/stripe-js": "^4.0.0"
```

Bản 4.x vẫn nằm trong range `svelte-stripe` chấp nhận, cài được bình thường. Vì donate/Stripe đã tắt hẳn qua `PUB_DISABLE_ALL_EXTERNAL_REQUESTS=true` (mục 1), thư viện này thực ra không còn được gọi tới lúc runtime — downgrade chỉ để **cài package qua được**, không ảnh hưởng gì tới tính năng vì tính năng đó đã tắt từ đầu.

---

## Build từ đầu — checklist đầy đủ

Thứ tự đúng: **fork repo → sửa code → xử lý file pandoc.wasm → mới deploy Cloudflare Pages.** Làm đúng thứ tự này để tránh vướng như lần đầu (deploy trước, dò lỗi từng vòng sau).

### Bước 1: Fork repo

Fork `VERT-sh/VERT` về tài khoản GitHub riêng.

> Lưu ý: nếu sau này build log báo Cloudflare tự detect `bun` thay vì `npm`, và dính lỗi `lockfile had changes, but lockfile is frozen` — xóa file `bun.lock` khỏi repo (nếu có) để Cloudflare chuyển hẳn sang dùng `npm`. Xem thêm ở bảng tra lỗi cuối README.

### Bước 2: Sửa code (làm trước khi deploy, không phải sau)

Sửa trực tiếp trên GitHub web: mở file cần sửa trong repo đã fork → bấm biểu tượng **bút chì** (Edit this file) ở góc trên phải → sửa → cuộn xuống **Commit changes** → chọn **Commit directly to the main branch** → **Commit changes**.

#### 2.1. `package.json`

Mở file `package.json`, tìm dòng:

```json
"@stripe/stripe-js": "^8.5.2",
```

Sửa thành:

```json
"@stripe/stripe-js": "^4.0.0",
```

Chỉ đổi đúng số version, giữ nguyên dấu `^`, dấu phẩy, và vị trí dòng trong file (nằm trong khối `"dependencies"`, không phải `"devDependencies"`). Lý do đổi — xem mục 5 phần "Thay đổi so với bản gốc" nếu cần hiểu sâu, nhưng bước làm chỉ cần đúng 1 dòng này.

#### 2.2. `src/lib/components/functional/Dialog.svelte`

Mở file, tìm 2 dòng này trong khối `<script>`:

```ts
let props: Props = $props();
const { id, title, message, buttons, type } = props;
const additional = "additional" in props ? props.additional : undefined;
```

Xóa cả 3 dòng trên, thay bằng đúng 2 dòng sau:

```ts
const { id, title, message, buttons, type, ...rest }: Props = $props();
const additional = rest && "additional" in rest ? (rest as any).additional : undefined;
```

Phần còn lại của file (`colors`, `Icons`, `$derived`, và toàn bộ HTML bên dưới `</script>`) giữ nguyên, không đụng vào.

#### 2.3. `src/lib/components/visual/Toast.svelte`

Mở file, tìm 3 dòng này trong khối `<script>`:

```ts
const props: {
    toast: ToastType<unknown>;
} = $props();
const { id, type, message, durations } = props.toast;
const additional =
    "additional" in props.toast ? props.toast.additional : {};
```

Xóa cả khối trên, thay bằng:

```ts
const { toast }: { toast: ToastType<unknown> } = $props();
const { id, type, message, durations } = toast;
const additional = toast && "additional" in toast ? (toast as any).additional : {};
```

> Lưu ý: cách sửa này chỉ hết lỗi hiển thị (không còn nhắc tới `props.toast` lòng vòng), nhưng **chưa hết warning `state_referenced_locally`** — build vẫn pass, chỉ là cảnh báo còn treo. Không sao, không cần sửa thêm trừ khi muốn dọn sạch (cách sửa triệt để hơn xem mục 4 phần "Thay đổi so với bản gốc").

#### 2.4. `src/lib/converters/pandoc.svelte.ts` — chưa động vào lúc này

File này cần link R2 mới sửa được — **bỏ qua bước này**, quay lại sửa cùng lúc với Bước 3 ngay dưới đây (mục 7 trong Bước 3).

#### File không sửa gì

`src/lib/converters/index.ts` — **giữ nguyên 100% bản gốc**, không mở file này ra sửa. Việc tắt video hoàn toàn qua 1 biến môi trường ở Bước 4, không đụng gì tới code (xem mục 1 phần "Thay đổi so với bản gốc" để hiểu vì sao).

### Bước 3: Xử lý file `pandoc.wasm` — tải về máy, đưa lên R2, gắn link vào code

File `pandoc.wasm` nặng 50.4MB, vượt giới hạn 25MB/file của Cloudflare Pages — phải tách ra khỏi repo, serve qua R2 thay vì bundle vào build. Xem đầy đủ lý do và các cách đã thử mà không work ở mục 2 phía trên.

Làm đúng thứ tự sau, **không đảo bước**:

1. **Tải file về máy** — file còn nguyên ở `static/pandoc.wasm` trong repo vừa fork, tải trực tiếp về.
2. **Tạo R2 bucket** trên Cloudflare Dashboard, đặt tên tùy ý.
3. **Upload file `pandoc.wasm`** vừa tải vào bucket đó.
4. **Bật Public Development URL** trong bucket → Settings, lấy được link dạng `https://pub-xxxx.r2.dev/pandoc.wasm`.
5. **Thêm CORS policy** cho bucket (bắt buộc, không có sẽ lỗi `Failed to fetch` khi web gọi tới):

```json
[
  {
    "AllowedOrigins": ["https://ten-domain-cua-ban.pages.dev"],
    "AllowedMethods": ["GET"],
    "AllowedHeaders": ["*"],
    "MaxAgeSeconds": 3600
  }
]
```

6. **Test thử link R2** — bấm trực tiếp vào URL vừa lấy, xác nhận trình duyệt tải được file (khoảng 50MB) trước khi đi tiếp.
7. **Gắn link R2 vào code** — mở file `src/lib/converters/pandoc.svelte.ts` trên GitHub web, tìm dòng:

```ts
this.wasm = await fetch("/pandoc.wasm").then((r) =>
    r.arrayBuffer(),
);
```

Xóa cả 3 dòng trên, thay bằng:

```ts
this.wasm = await fetch(
    "URL_R2_CUA_BAN/pandoc.wasm",
).then((r) => r.arrayBuffer());
```

Đổi `URL_R2_CUA_BAN` thành đúng link lấy được ở bước 4 (dạng `https://pub-xxxx.r2.dev`). Commit thẳng lên `main` như cách làm ở Bước 2.

8. **Xóa file `static/pandoc.wasm` khỏi repo** — chỉ làm bước này sau khi bước 7 đã commit xong. Vào thư mục `static` trên GitHub → bấm vào file `pandoc.wasm` → bấm biểu tượng **thùng rác** (Delete this file) ở góc trên phải → cuộn xuống **Commit changes** → **Commit directly to the main branch**. Xóa sớm hơn bước 7 = mất file, phải tìm lại qua lịch sử git (tab **Commits** → tìm commit trước lúc xóa → mở file → **Download**).

### Bước 4: Deploy Cloudflare Pages

Bây giờ mới tới phần deploy, vì code đã chuẩn bị xong hết ở Bước 2-3.

1. Vào Cloudflare Dashboard → **Workers & Pages** → **Create** → connect với repo đã fork.

2. **Environment variables** — Settings → Variables and secrets, thêm đủ các biến sau (copy từ `.env.example` gốc, sửa 2 giá trị):

| Name | Value | Ghi chú |
|---|---|---|
| `PUB_HOSTNAME` | `localhost:5173` | Chỉ dùng cho analytics, không ảnh hưởng build |
| `PUB_PLAUSIBLE_URL` | `https://plausible.example.com` | Placeholder, không dùng analytics thật |
| `PUB_ENV` | `production` | |
| `PUB_VERTD_URL` | `https://vertd.vert.sh` | Không dùng tới vì video đã tắt, nhưng vẫn cần set để tránh lỗi biến không tồn tại lúc build |
| `PUB_DISABLE_ALL_EXTERNAL_REQUESTS` | `true` | **Quan trọng** — tắt video + donate + analytics |
| `PUB_DISABLE_FAILURE_BLOCKS` | `false` | |
| `PUB_DONATION_URL` | `https://donations.vert.sh` | Không hiện ra vì đã tắt ở trên |
| `PUB_STRIPE_KEY` | *(key gốc từ .env.example)* | Không dùng tới vì đã tắt donate |

3. **Build configuration** — Settings → Build → Build configuration:

```
Build command: npm run build
Build output directory: build
```

⚠️ Cloudflare mặc định set `Build output directory` là `.svelte-kit/output/client` — **sai**, phải đổi thành `build` (lý do xem mục 3 phía trên), nếu không domain sẽ 404 dù build báo Success.

4. **Deploy** — save toàn bộ setting, trigger build (push commit hoặc Retry deployment).

### Bước 5: Test sau khi deploy

Mở domain `.pages.dev`, kiểm tra:

- [ ] Trang chủ load được, không 404
- [ ] Convert ảnh (png → ico chẳng hạn) chạy được
- [ ] Convert audio chạy được
- [ ] Convert document (docx → md chẳng hạn) chạy được, không lỗi CORS trong Console
- [ ] Không thấy nút/tính năng convert video trong UI
- [ ] Không thấy nút donate

---

## Lỗi từng gặp — tra cứu nhanh khi build lại

| Lỗi | Nguyên nhân | Cách sửa |
|---|---|---|
| `npm error ERESOLVE ... @stripe/stripe-js` | Version `@stripe/stripe-js` không khớp peer dependency của `svelte-stripe` | Xem mục 5 ở trên |
| `"PUB_DISABLE_ALL_EXTERNAL_REQUESTS" is not exported` | Thiếu biến env trên Cloudflare | Thêm đủ biến env như bảng ở Bước 4 |
| `"PUB_VERTD_URL" is not exported` | Thiếu biến env | Như trên |
| `Error: Pages only supports files up to 25 MiB in size` — `pandoc.wasm is 50.4 MiB` | File quá lớn so với giới hạn Cloudflare (Pages lẫn Workers đều 25MB/file) | Chuyển sang load từ R2 (mục 2) |
| `npm error lockfile had changes, but lockfile is frozen` | `bun.lock` không khớp `package.json` sau khi sửa dependency | Xóa `bun.lock`, để Cloudflare tự cài lại bằng npm, hoặc chạy `bun install` lại ở máy local rồi commit lockfile mới |
| Domain `.pages.dev` trả về 404 dù build Success | Build output directory sai — Cloudflare mặc định set `.svelte-kit/output/client`, nhưng `adapter-static` xuất ra `build` | Sửa Build output directory thành `build` trong Settings |
| Console: `Failed to load Pandoc worker: TypeError: Failed to fetch` | CORS chưa bật trên R2 bucket, `fetch()` từ JS bị trình duyệt chặn dù link vẫn mở được trực tiếp | Thêm CORS policy ở R2 bucket Settings |
| Vite warning `state_referenced_locally` ở `Dialog.svelte` | Destructure `$props()` qua biến trung gian, sai cú pháp Svelte 5 runes | Đã sửa triệt để — xem mục 4. Hết warning. |
| Vite warning `state_referenced_locally` ở `Toast.svelte` | Như trên | Mới sửa 1 phần, warning vẫn còn — xem mục 4 để biết cách sửa triệt để nếu muốn dọn nốt |

---

## Ghi chú thêm

- Repo gốc: [VERT-sh/VERT](https://github.com/VERT-sh/VERT)
- Server `vertd` (nếu sau này muốn tự host để bật lại video): [VERT-sh/vertd](https://github.com/VERT-sh/vertd)
- Tài liệu chính thức về video conversion: xem `docs/VIDEO_CONVERSION.md` trong repo gốc
