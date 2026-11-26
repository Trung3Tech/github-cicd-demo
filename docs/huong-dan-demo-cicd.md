# Hướng dẫn demo CI/CD với GitHub Actions

Tài liệu đi kèm bộ slide **“CI/CD với GitHub Actions – Demo từng bước”**. Giảng viên dùng để chuẩn bị và trình diễn; sinh viên dùng để tự làm lại ở nhà. Toàn bộ demo chỉ dùng GitHub: GitHub Actions chạy CI, GitHub Pages chạy website, GitHub Releases lưu bản phát hành.

---

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Chuẩn bị](#2-chuẩn-bị)
3. [Phân bổ thời gian](#3-phân-bổ-thời-gian)
4. [Các bước demo](#4-các-bước-demo)
5. [Kịch bản demo lỗi](#5-kịch-bản-demo-lỗi)
6. [Reset để demo lại](#6-reset-để-demo-lại)
7. [Xử lý lỗi thường gặp](#7-xử-lý-lỗi-thường-gặp)
8. [Câu hỏi thường gặp](#8-câu-hỏi-thường-gặp)
9. [Tài liệu tham khảo](#9-tài-liệu-tham-khảo)

---

## 1. Tổng quan

### 1.1 Ứng dụng demo

Ứng dụng **Việc cần làm** viết bằng HTML, CSS và JavaScript thuần, chạy bằng Vite.

- Thêm công việc; báo lỗi khi ô nhập trống hoặc dài hơn 100 ký tự.
- Đánh dấu hoàn thành, xoá từng việc, xoá mọi việc đã xong.
- Lọc theo Tất cả, Đang làm, Đã xong; đếm số việc chưa xong.
- Lưu vào `localStorage`, tải lại trang vẫn còn dữ liệu.

### 1.2 Pipeline

```text
Push / Pull request
   ├── Lint       (ESLint + Prettier)   ┐
   ├── Unit test  (Vitest + coverage)   ├─ chạy song song
   └── E2E test   (Playwright)          ┘
          │  cả 3 xanh
          ▼
        Build (vite build → dist/)
          │  chỉ khi code vào nhánh main
          ▼
        Deploy → GitHub Pages

Push tag v*.*.*  →  Release (lint, test, build, zip) → GitHub Releases
```

### 1.3 Cấu trúc mã nguồn

| Đường dẫn                              | Vai trò                                                                 |
| -------------------------------------- | ----------------------------------------------------------------------- |
| `index.html`                           | Khung giao diện, có `label` và `role` để dễ test                        |
| `src/todo.js`                          | Logic thuần (không DOM): thêm, xoá, đổi trạng thái, lọc, đếm            |
| `src/main.js`                          | Giao diện: bắt sự kiện, gọi logic, lưu `localStorage`, vẽ lại danh sách |
| `src/style.css`                        | Giao diện                                                               |
| `tests/unit/todo.test.js`              | 13 unit test cho `todo.js`                                              |
| `tests/e2e/todo.spec.js`               | 4 kịch bản test giao diện bằng trình duyệt thật                         |
| `eslint.config.js`, `.prettierrc.json` | Quy tắc lint và định dạng                                               |
| `vite.config.js`                       | `base: './'` cho GitHub Pages; cấu hình test và ngưỡng coverage 90%     |
| `playwright.config.js`                 | Tự build, chạy `vite preview` cổng 4173 rồi chạy test                   |
| `.github/workflows/ci.yml`             | Workflow CI và deploy Pages                                             |
| `.github/workflows/release.yml`        | Workflow phát hành khi gắn tag                                          |
| `.github/dependabot.yml`               | Tự mở PR nâng phiên bản hằng tuần                                       |

### 1.4 Các lệnh npm

| Lệnh                    | Tác dụng                                                      |
| ----------------------- | ------------------------------------------------------------- |
| `npm run dev`           | Chạy thử tại `http://localhost:5173`, tự tải lại khi sửa code |
| `npm run lint`          | Kiểm tra lỗi code bằng ESLint                                 |
| `npm run format:check`  | Kiểm tra định dạng bằng Prettier                              |
| `npm run format`        | Tự sửa định dạng                                              |
| `npm test`              | Chạy unit test                                                |
| `npm run test:coverage` | Chạy unit test và đo coverage                                 |
| `npm run test:e2e`      | Chạy test giao diện bằng Playwright                           |
| `npm run build`         | Đóng gói vào thư mục `dist/`                                  |
| `npm run preview`       | Xem bản build ở máy                                           |

> **Nguyên tắc:** mọi bước trong CI đều là các lệnh npm này. Khi CI báo đỏ, chạy lại đúng lệnh đó ở máy để tái hiện lỗi.

---

## 2. Chuẩn bị

### 2.1 Sinh viên

| Cần có                                               | Kiểm tra                     | Link                              |
| ---------------------------------------------------- | ---------------------------- | --------------------------------- |
| Tài khoản GitHub                                     | Đăng nhập được github.com    | https://github.com/signup         |
| GitHub Student Developer Pack (khuyến nghị)          | Trang Education báo đã duyệt | https://education.github.com/pack |
| Git                                                  | `git --version`              | https://git-scm.com/downloads     |
| Node.js 22 LTS trở lên                               | `node -v`                    | https://nodejs.org                |
| VS Code                                              | Mở được thư mục dự án        | https://code.visualstudio.com     |
| Extension ESLint, Prettier cho VS Code (khuyến nghị) | Thấy lỗi gạch đỏ khi gõ      | Marketplace trong VS Code         |

Xác thực Git với GitHub, chọn một trong hai cách:

- Dùng GitHub CLI: cài từ https://cli.github.com rồi chạy `gh auth login`.
- Dùng Git Credential Manager (có sẵn trong Git for Windows; macOS cài kèm Git qua Homebrew) và đăng nhập khi `git push` hỏi.

### 2.2 Giảng viên (làm trước buổi học ít nhất 1 ngày)

1. Chạy thử toàn bộ 12 bước trên một repo thật, đến khi web chạy trên GitHub Pages và có release `v1.0.0`.
2. **Giữ repo chạy thử đó** làm phương án dự phòng. Nếu mạng chậm khi demo, mở các lần chạy đã thành công để giải thích.
3. Tải sẵn trình duyệt cho Playwright trên máy trình chiếu: `npx playwright install chromium` (khoảng 100 MB).
4. Chuẩn bị sẵn hai nhánh lỗi cho phần 5 (tuỳ chọn) để không phải gõ trong lúc demo.
5. Mở sẵn các tab trình duyệt: repo trên GitHub, tab **Actions**, **Settings**, trang GitHub Pages.
6. Phóng to font VS Code và terminal (khoảng 18–20) để lớp đọc được.

### 2.3 Chi phí

Không tốn phí khi dùng **repo public**: GitHub Actions và GitHub Pages miễn phí cho repo public. Repo private vẫn dùng được Actions (giới hạn phút chạy theo gói), còn GitHub Pages cho repo private cần gói trả phí hoặc Student Developer Pack.

---

## 3. Phân bổ thời gian

Gợi ý cho buổi 3 giờ.

| Thời gian   | Nội dung                                                      | Slide |
| ----------- | ------------------------------------------------------------- | ----- |
| 0:00 – 0:15 | Mở đầu, giới thiệu ứng dụng, pipeline tổng thể, cấu trúc repo | 1–5   |
| 0:15 – 0:45 | Bước 1–3: chuẩn bị, chạy ở máy, đọc code                      | 6–9   |
| 0:45 – 1:25 | Bước 4–6: lint, unit test, E2E test ở máy                     | 10–15 |
| 1:25 – 1:35 | Nghỉ giải lao                                                 | –     |
| 1:35 – 2:10 | Bước 7–8: viết workflow, push, đọc tab Actions                | 16–19 |
| 2:10 – 2:35 | Bước 9–10: Rulesets, PR lỗi bị chặn                           | 20–22 |
| 2:35 – 2:50 | Bước 11–12: GitHub Pages, release bằng tag                    | 23–25 |
| 2:50 – 3:00 | Bảo trì, bảo mật, checklist, bài tập                          | 26–32 |

---

## 4. Các bước demo

Mỗi bước có: **Làm gì**, **Kết quả mong đợi**, **Điểm nhấn khi giảng**.

### Bước 1 – Tạo repo trên GitHub

**Làm gì**

1. Trên github.com bấm **New repository**.
2. Repository name: `github-cicd-demo`; chọn **Public**.
3. **Không** tích _Add a README_, _.gitignore_, _license_ (repo phải trống để push lần đầu không bị xung đột).
4. Bấm **Create repository**.

**Kết quả mong đợi:** trang repo trống, có hướng dẫn “…or push an existing repository from the command line”.

### Bước 2 – Chạy ứng dụng ở máy

**Làm gì**

```bash
cd github-cicd-demo
npm install
npm run dev
```

Mở `http://localhost:5173`, thêm vài công việc, thử bộ lọc, tải lại trang.

**Kết quả mong đợi**

```text
  VITE ready in ... ms
  ➜  Local:   http://localhost:5173/
```

Dữ liệu vẫn còn sau khi tải lại trang. Dừng server bằng `Ctrl + C`.

**Điểm nhấn:** `npm install` tạo `package-lock.json` ghi chính xác phiên bản từng thư viện. CI dùng `npm ci` để cài đúng theo file này, nên **phải commit `package-lock.json`**.

### Bước 3 – Đọc code

**Làm gì:** mở lần lượt `src/todo.js`, `src/main.js`, `index.html`.

**Điểm nhấn**

- `todo.js` gồm các **hàm thuần**: cùng đầu vào cho cùng kết quả, không đụng DOM, không ghi `localStorage`, không sửa mảng gốc. Nhờ vậy unit test chỉ cần gọi hàm và so sánh kết quả.
- `main.js` chỉ làm một vòng: bắt sự kiện → gọi hàm thuần → lưu → vẽ lại (`update()` → `save()` → `render()`).
- `index.html` dùng `<label for="todo-input">Công việc mới</label>` và `role="alert"`. Đây vừa là thực hành tốt cho người dùng trình đọc màn hình, vừa là cách Playwright tìm phần tử (`getByLabel`, `getByRole`).
- ID công việc được tạo bằng `crypto.randomUUID()`. Nếu dùng `Date.now()`, hai việc thêm trong cùng một mili giây sẽ trùng ID; lỗi này đã được test E2E phát hiện khi viết demo.

### Bước 4 – Lint

**Làm gì**

```bash
npm run lint
npm run format:check
```

**Kết quả mong đợi:** `npm run lint` không in lỗi nào; `format:check` in `All matched files use Prettier code style!`.

**Điểm nhấn**

- **ESLint** bắt lỗi logic: biến khai báo mà không dùng (`no-unused-vars`), so sánh `==` thay vì `===` (`eqeqeq`), `let` không đổi giá trị (`prefer-const`).
- **Prettier** thống nhất định dạng: dấu nháy đơn, dòng tối đa 100 ký tự.
- ESLint 9 trở lên dùng cấu hình dạng phẳng trong `eslint.config.js`.
- Demo lỗi lint: xem [mục 5.1](#51-lỗi-lint).

### Bước 5 – Unit test

**Làm gì**

```bash
npm test
npm run test:coverage
```

**Kết quả mong đợi**

```text
 Test Files  1 passed (1)
      Tests  13 passed (13)

File      | % Stmts | % Branch | % Funcs | % Lines
----------|---------|----------|---------|--------
 todo.js  |     100 |    93.33 |     100 |     100
```

Báo cáo HTML nằm ở `coverage/index.html`.

**Điểm nhấn**

- `describe` nhóm test, `it` là một trường hợp, `expect(...).toEqual(...)` so sánh kết quả.
- `it.each` chạy một test với nhiều bộ dữ liệu (ba bộ lọc).
- Ngưỡng coverage 90% đặt trong `vite.config.js` (`test.coverage.thresholds`). Nếu xoá bớt test làm coverage dưới 90%, lệnh `test:coverage` báo lỗi và CI đỏ.
- Demo test fail: xem [mục 5.2](#52-lỗi-unit-test).

### Bước 6 – E2E test với Playwright

**Làm gì**

```bash
npx playwright install chromium   # chỉ cần lần đầu
npm run test:e2e
```

Xem trực quan từng bước:

```bash
npx playwright test --ui
```

**Kết quả mong đợi**

```text
  4 passed (2s)
```

**Điểm nhấn**

- `playwright.config.js` có `webServer`: Playwright tự `npm run build`, chạy `vite preview` cổng 4173, chạy test rồi tự tắt server.
- Tìm phần tử theo vai trò và nhãn: `getByRole('button', { name: 'Thêm' })`, `getByLabel('Công việc mới')`.
- `expect(...)` tự chờ đến khi điều kiện đúng, không cần `sleep`.
- **Strict mode:** mỗi locator phải khớp đúng một phần tử. `getByRole('button', { name: 'Đã xong' })` khớp cả nút lọc “Đã xong” và nút “Xoá việc đã xong”, nên test dùng `exact: true`. Đây là lỗi thật đã gặp khi viết demo, dùng làm ví dụ.
- Khi test fail ở máy: `npx playwright show-report` mở báo cáo HTML.

### Bước 7 – Viết workflow CI

**Làm gì:** mở `.github/workflows/ci.yml`, giải thích theo thứ tự:

| Khối                                                   | Ý nghĩa                                                                                       |
| ------------------------------------------------------ | --------------------------------------------------------------------------------------------- |
| `on: push / pull_request / workflow_dispatch`          | Chạy khi push lên main, mở PR vào main, hoặc bấm tay                                          |
| `permissions: contents: read`                          | Quyền tối thiểu cho mọi job                                                                   |
| `concurrency`                                          | Push mới vào cùng nhánh sẽ huỷ lần chạy cũ đang dở                                            |
| Job `lint`, `unit-test`, `e2e-test`                    | Không có `needs` nên chạy song song trên 3 máy khác nhau                                      |
| `actions/setup-node` với `cache: npm`                  | Lưu cache thư viện, lần sau cài nhanh hơn                                                     |
| `npx playwright install --with-deps chromium`          | Cài trình duyệt và thư viện hệ thống trên Ubuntu                                              |
| `upload-artifact` với `if: always()`                   | Luôn lưu báo cáo coverage                                                                     |
| `upload-artifact` với `if: failure()`                  | Chỉ lưu báo cáo Playwright khi E2E fail                                                       |
| Job `build` với `needs: [lint, unit-test, e2e-test]`   | Chỉ build khi cả 3 check xanh                                                                 |
| Job `deploy` với `if: github.ref == 'refs/heads/main'` | Chỉ deploy từ main; trên PR, `github.ref` có dạng `refs/pull/<số>/merge` nên job tự bị bỏ qua |

**Điểm nhấn:** tên hiển thị của job (`name: Lint`, `name: Unit test`, `name: E2E test`) chính là tên check dùng ở bước 9.

### Bước 8 – Push và đọc tab Actions

**Làm gì**

```bash
git init -b main
git add .
git commit -m "feat: todo app with CI"
git remote add origin https://github.com/<tài-khoản>/github-cicd-demo.git
git push -u origin main
```

Mở tab **Actions** → workflow **CI** → lần chạy vừa tạo.

**Kết quả mong đợi**

| Job                       | Trạng thái lần đầu                                 |
| ------------------------- | -------------------------------------------------- |
| Lint, Unit test, E2E test | Xanh (chạy song song)                              |
| Build                     | Xanh                                               |
| Deploy to GitHub Pages    | **Đỏ**, vì chưa bật GitHub Pages; sẽ sửa ở bước 11 |

**Điểm nhấn**

- Kiểm tra `git status` trước khi commit: không được có `node_modules/`, `dist/`, `coverage/` (đã có trong `.gitignore`).
- Bấm vào từng job để xem log từng step; ô tìm kiếm trong log giúp tìm nhanh.
- Cuối trang Summary có mục **Artifacts**: tải `coverage-report` về, mở `index.html`.

### Bước 9 – Bảo vệ nhánh main bằng Rulesets

**Làm gì**

1. Repo → **Settings** → **Rules** → **Rulesets** → **New ruleset** → **New branch ruleset**.
2. Ruleset name: `protect-main`; Enforcement status: **Active**.
3. Target branches → **Add target** → **Include default branch**.
4. Tích **Restrict deletions** và **Block force pushes**.
5. Tích **Require a pull request before merging** (Required approvals để 0 nếu demo một mình).
6. Tích **Require status checks to pass** → **Add checks** → chọn **Lint**, **Unit test**, **E2E test**.
7. Bấm **Create**.

**Kết quả mong đợi:** không push thẳng lên main được nữa; mọi thay đổi phải qua PR và 3 check phải xanh.

**Điểm nhấn**

- Tên check chỉ xuất hiện trong danh sách **sau khi workflow đã chạy ít nhất một lần**, nên phải làm bước 8 trước.
- Không cần chọn Build và Deploy vì chúng chỉ chạy sau khi 3 check pass.
- Rulesets miễn phí cho repo public.

### Bước 10 – PR lỗi bị chặn

Làm theo [mục 5.1](#51-lỗi-lint): mở một PR có lỗi lint.

**Kết quả mong đợi:** check **CI / Lint** đỏ, hai check còn lại xanh, nút **Merge pull request** bị khoá với thông báo _required status check “Lint” has not succeeded_. Sửa lỗi, push tiếp vào cùng nhánh, check chạy lại và xanh, merge được.

**Điểm nhấn:** đây là ý quan trọng nhất của CI: **lỗi bị chặn trước khi vào main**.

### Bước 11 – Deploy lên GitHub Pages

**Làm gì**

1. Repo → **Settings** → **Pages**.
2. Mục **Build and deployment** → **Source**: chọn **GitHub Actions**.
3. Tab **Actions** → lần chạy CI gần nhất trên main → **Re-run jobs** → **Re-run all jobs** (hoặc merge một PR bất kỳ).

**Kết quả mong đợi**

- Job **Deploy to GitHub Pages** xanh, hiện link `https://<tài-khoản>.github.io/github-cicd-demo/`.
- Trang chủ repo có mục **Deployments** → `github-pages` ghi lại lịch sử deploy.

**Điểm nhấn**

- Job `build` dùng `actions/upload-pages-artifact` đóng gói `dist/`; job `deploy` dùng `actions/deploy-pages`.
- Quyền `pages: write` và `id-token: write` chỉ cấp cho riêng job deploy.
- `base: './'` trong `vite.config.js` là bắt buộc. Thiếu nó, web hiện trang trắng vì file JS, CSS bị tìm ở `/assets/` thay vì `/github-cicd-demo/assets/`.
- Demo trọn vòng: sửa dòng `Demo CI/CD với GitHub Actions` trong `index.html` → mở PR → 3 check xanh → merge → khoảng 2 phút sau tải lại trang web để thấy thay đổi.

### Bước 12 – Release bằng tag

**Làm gì**

```bash
git checkout main
git pull
git tag v1.0.0
git push origin v1.0.0
```

**Kết quả mong đợi**

- Tab **Actions**: workflow **Release** chạy (lint, test, build, zip, publish).
- Tab **Releases** (cột phải trang repo): có `v1.0.0`, ghi chú tự sinh từ các PR đã merge, đính kèm `todo-app-v1.0.0.zip`.

**Điểm nhấn**

- `GH_TOKEN: ${{ github.token }}` là token tự động của mỗi lần chạy, không cần tạo secret.
- `permissions: contents: write` bắt buộc để tạo release.
- Đặt tên tag theo semantic versioning: `v1.0.1` sửa lỗi, `v1.1.0` thêm tính năng, `v2.0.0` thay đổi lớn.
- Workflow CI không chạy khi push tag (chỉ lắng nghe nhánh main), nên release tự lint và test lại.

---

## 5. Kịch bản demo lỗi

Làm trên nhánh riêng, mở PR vào main để thấy check bị chặn. Đầu ra dưới đây là kết quả thật khi chạy ở máy.

### 5.1 Lỗi lint

```bash
git checkout -b feature/debug-flag
```

Trong `src/main.js`, ngay dưới dòng `let currentFilter = 'all';` thêm:

```js
const debugMode = true;
```

Chạy thử ở máy trước:

```text
$ npm run lint

src/main.js
  23:7  error  'debugMode' is assigned a value but never used  no-unused-vars

✖ 1 problem (1 error, 0 warnings)
```

Đẩy lên và mở PR:

```bash
git commit -am "feat: add debug flag"
git push -u origin feature/debug-flag
```

Trên GitHub bấm **Compare & pull request** → **Create pull request**. Sửa: xoá dòng vừa thêm, `git commit -am "fix: remove unused flag"`, `git push`.

### 5.2 Lỗi unit test

```bash
git checkout main && git pull
git checkout -b feature/broken-filter
```

Trong `src/todo.js`, sửa dòng bộ lọc `active` (bỏ dấu `!`):

```js
if (filter === 'active') return todos.filter((todo) => todo.done);
```

Kết quả ở máy:

```text
 FAIL  tests/unit/todo.test.js > filterTodos và countActive > bộ lọc "active"
AssertionError: expected [ 2 ] to deeply equal [ 1 ]
 ❯ tests/unit/todo.test.js:67:58

      Tests  1 failed | 12 passed (13)
```

Khi mở PR, cả **Unit test** và **E2E test** cùng đỏ, vì kịch bản E2E bấm nút lọc cũng dùng hàm này.

**Điểm nhấn:** unit test chỉ ra **chính xác hàm sai**; E2E test chỉ ra **tính năng nào hỏng** với người dùng. Tải artifact `playwright-report` ở cuối trang run, giải nén, chạy `npx playwright show-report playwright-report` để xem ảnh chụp và trace.

### 5.3 Lỗi định dạng

Thêm dấu cách thừa hoặc đổi `'` thành `"` ở một dòng bất kỳ trong `src/main.js`:

```text
$ npm run format:check
[warn] src/main.js
[warn] Code style issues found in the above file. Run Prettier with --write to fix.
```

Sửa bằng `npm run format`.

---

## 6. Reset để demo lại

| Muốn làm lại                      | Cách làm                                                                                                                 |
| --------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Xoá nhánh lỗi đã demo             | Trên PR bấm **Close pull request** → **Delete branch**; ở máy `git branch -D <tên-nhánh>`                                |
| Xoá release và tag                | Tab **Releases** → `v1.0.0` → **Delete**; rồi `git push --delete origin v1.0.0` và `git tag -d v1.0.0`                   |
| Gỡ ruleset để push thẳng          | **Settings → Rules → Rulesets → protect-main** → đổi Enforcement thành **Disabled**                                      |
| Làm lại từ đầu                    | Xoá repo (**Settings → General → Danger Zone → Delete this repository**); ở máy xoá thư mục `.git` rồi làm lại từ bước 1 |
| Xoá dữ liệu thử trong trình duyệt | DevTools → Application → Local Storage → xoá khoá `cicd-demo-todos`                                                      |

---

## 7. Xử lý lỗi thường gặp

| Hiện tượng                                           | Nguyên nhân                                         | Cách sửa                                                                                                      |
| ---------------------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `npm ci` báo lock file không khớp `package.json`     | Thêm thư viện nhưng chưa commit `package-lock.json` | Chạy `npm install` ở máy, commit `package-lock.json`                                                          |
| CI đỏ ở step `format:check`                          | Code chưa chạy Prettier                             | `npm run format`, commit lại                                                                                  |
| `Executable doesn't exist` khi chạy Playwright       | Chưa cài trình duyệt                                | Ở máy: `npx playwright install chromium`; trong CI phải có step `npx playwright install --with-deps chromium` |
| E2E báo `strict mode violation`                      | Locator khớp nhiều phần tử                          | Dùng tên đầy đủ, `exact: true`, hoặc `getByRole` với vai trò cụ thể hơn                                       |
| E2E báo cổng 4173 đang bận                           | Còn `vite preview` chạy từ lần trước                | macOS/Linux: `lsof -i :4173` rồi tắt tiến trình đó; Windows: `netstat -ano \| findstr 4173`                   |
| Không thấy check khi tạo ruleset                     | Workflow chưa chạy lần nào                          | Push để workflow chạy ít nhất một lần                                                                         |
| Job deploy báo lỗi lấy thông tin Pages               | Chưa bật Pages                                      | **Settings → Pages → Source: GitHub Actions**, rồi re-run                                                     |
| Web trên Pages trắng, Console báo 404 file `.js`     | Thiếu `base: './'` trong `vite.config.js`           | Thêm lại, commit, deploy lại                                                                                  |
| Deploy bị từ chối do quy tắc nhánh của environment   | Đang deploy từ nhánh khác main                      | Chỉ deploy từ main; xem **Settings → Environments → github-pages**                                            |
| Release báo `Resource not accessible by integration` | Thiếu quyền ghi                                     | Thêm `permissions: contents: write`                                                                           |
| `git push` lên main bị từ chối                       | Ruleset đang bật                                    | Làm việc trên nhánh riêng và mở PR                                                                            |
| `git push` hỏi mật khẩu rồi báo lỗi                  | GitHub không nhận mật khẩu tài khoản cho Git        | Dùng `gh auth login` hoặc Git Credential Manager                                                              |

**Cách đọc lỗi chung:** mở step đỏ → tìm dòng lỗi **đầu tiên** (không phải dòng cuối) → chạy lại đúng lệnh của step đó ở máy. Nếu lỗi chỉ xảy ra trên CI: **Re-run jobs** → tích **Enable debug logging**.

---

## 8. Câu hỏi thường gặp

**Vì sao chạy lint, unit test và E2E song song thay vì nối tiếp?**
Ba job độc lập nhau, chạy song song trên ba máy nên tổng thời gian chỉ bằng job lâu nhất (thường là E2E). Kết quả hiện riêng từng check nên biết ngay phần nào hỏng.

**Vì sao build lại ở job `build` thay vì dùng bản build trong job E2E?**
Mỗi job chạy trên một máy mới, không chia sẻ file. Job `build` tạo một bản duy nhất, đóng gói thành artifact, và job `deploy` đưa đúng bản đó lên Pages.

**Có cần tạo secret nào không?**
Không. Deploy Pages dùng quyền `pages: write` và `id-token: write`; release dùng `github.token`. GitHub tự cấp token ngắn hạn cho mỗi lần chạy.

**PR từ repo fork có deploy hay tạo release được không?**
Không. Workflow chạy trên PR từ fork chỉ có quyền đọc, và job deploy chỉ chạy trên nhánh main.

**Dependabot làm gì?**
Mỗi tuần tự mở PR nâng phiên bản thư viện npm và các action. CI chạy trên các PR đó để chứng minh bản mới không làm hỏng gì; xanh thì merge.

**Chạy E2E trên nhiều trình duyệt thế nào?**
Thêm project `firefox` và `webkit` vào `playwright.config.js` (dùng `devices['Desktop Firefox']`, `devices['Desktop Safari']`), và đổi lệnh cài trong workflow thành `npx playwright install --with-deps`.

---

## 9. Tài liệu tham khảo

| Chủ đề                              | Link                                                                                                                        |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| GitHub Actions                      | https://docs.github.com/actions                                                                                             |
| Cú pháp workflow                    | https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions                                     |
| Rulesets                            | https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets |
| GitHub Pages với workflow tuỳ chỉnh | https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages                 |
| Deploy Vite lên GitHub Pages        | https://vite.dev/guide/static-deploy.html#github-pages                                                                      |
| `gh release create`                 | https://cli.github.com/manual/gh_release_create                                                                             |
| Dependabot                          | https://docs.github.com/en/code-security/dependabot                                                                         |
| ESLint                              | https://eslint.org/docs/latest/use/getting-started                                                                          |
| Prettier                            | https://prettier.io/docs/                                                                                                   |
| Vitest                              | https://vitest.dev/guide/                                                                                                   |
| Playwright                          | https://playwright.dev/docs/intro                                                                                           |
| Playwright trên CI                  | https://playwright.dev/docs/ci-intro                                                                                        |
| GitHub Student Developer Pack       | https://education.github.com/pack                                                                                           |
