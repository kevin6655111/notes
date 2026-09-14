# Playwright + VS Code UI 測試

> Playwright 是微軟的端對端（E2E）瀏覽器自動化測試框架，支援 Chromium、Firefox、WebKit。搭配 VS Code 擴充套件可直接在編輯器點擊執行、錄製與除錯測試。

## 目錄

- [📥 建立專案](#init)
- [🧩 VS Code 擴充套件](#vscode)
- [▶️ 執行測試](#run)
- [🗄️ 測試中連 MSSQL](#mssql)
- [🚨 設定與錯誤排除](#trouble)
- [📚 參考資料](#ref)

---

<a id="init"></a>

## 📥 建立專案

前置：已安裝 Node.js（見 [Node.js 安裝（nvm）](nodejs.md)）。

### 全新專案

在專案資料夾下執行互動式初始化，會建立 `playwright.config.ts`、`tests/` 範例與 `package.json`：

```shell
npm init playwright@latest
# 或用 yarn
yarn create playwright
```

問答項目：TypeScript / JavaScript、測試資料夾名稱、是否加 GitHub Actions、是否立刻下載瀏覽器。

> 原筆記寫的 `yarn add init playwright@latest` 是錯的，`yarn add` 會把名為 `init` 的套件裝進 dependencies。建立專案請用 `yarn create playwright`，既有專案加 Playwright 用 `yarn add -D @playwright/test`。

### 既有專案（clone 下來）

```shell
npm i -g yarn                 # 尚未安裝 yarn 時，全域安裝一次
yarn                          # 依 package.json 安裝相依套件
npx playwright install        # 下載 Chromium / Firefox / WebKit
```

Linux / WSL 第一次跑可能缺系統函式庫，加 `--with-deps` 一併安裝：

```shell
npx playwright install --with-deps
```

---

<a id="vscode"></a>

## 🧩 VS Code 擴充套件

在 VS Code 擴充套件市集搜尋 **Playwright Test for VSCode**（發行者 Microsoft，ID `ms-playwright.playwright`）安裝。

> 原筆記的 `git clone https://github.com/microsoft/playwright-vscode.git` 是擴充套件的原始碼倉庫，只有要修改或除錯擴充套件本身才需要，一般使用直接從市集安裝即可。

安裝後在側邊欄「測試」圖示會列出所有測試，並提供：

| 功能 | 說明 |
|------|------|
| 綠色箭頭 | 在測試函式旁點擊即可執行單一測試，右鍵可 Debug |
| Show browser | 勾選後執行時顯示瀏覽器，不勾則 headless |
| Record new | 開啟瀏覽器錄製操作，自動產生測試程式碼 |
| Pick locator | 在瀏覽器上點元素，取得建議的 locator |
| Show trace viewer | 檢視每一步的截圖、DOM、網路請求 |

---

<a id="run"></a>

## ▶️ 執行測試

```shell
npx playwright test                       # 執行全部（headless）
npx playwright test tests/login.spec.ts   # 執行單一檔案
npx playwright test -g "登入"              # 依測試名稱篩選
npx playwright test --headed              # 顯示瀏覽器
npx playwright test --project=chromium    # 只跑指定瀏覽器
npx playwright test --debug               # 開 Inspector 逐步除錯
npx playwright test --ui                  # UI 模式，可監看、篩選、看 trace

npx playwright show-report                # 開啟上次執行的 HTML 報告
npx playwright codegen https://example.com   # 命令列錄製
```

`package.json` 常用 scripts：

```json
"scripts": {
  "test": "playwright test",
  "test:headed": "playwright test --headed",
  "test:ui": "playwright test --ui",
  "report": "playwright show-report"
}
```

> 原筆記的 `yarn serve` 是該專案自訂的 script，不是 Playwright 內建指令，內容需看該專案的 `package.json`。

---

<a id="mssql"></a>

## 🗄️ 測試中連 MSSQL

UI 測試前後需要準備或驗證資料庫資料時，安裝 `mssql` 套件：

```shell
yarn add -D mssql
yarn add -D @types/mssql     # TypeScript 型別
```

```ts
import sql from "mssql";

const config: sql.config = {
  server: "localhost",
  database: "TestDb",
  user: "sa",
  password: process.env.DB_PASSWORD!,
  options: { encrypt: false, trustServerCertificate: true },
};

test("登入後資料寫入正確", async ({ page }) => {
  // ...UI 操作...

  const pool = await sql.connect(config);
  const result = await pool.request().query("SELECT COUNT(*) AS n FROM Logs");
  expect(result.recordset[0].n).toBeGreaterThan(0);
  await pool.close();
});
```

密碼不要寫死在程式裡，放 `.env` 並加進 `.gitignore`，用 `dotenv` 讀取。

---

<a id="trouble"></a>

## 🚨 設定與錯誤排除

### 測試旁沒有綠色箭頭

- 測試面板右上 `...` → **Clear All Results**，或重新載入視窗（`Ctrl + Shift + P` → Reload Window）。
- 確認檔名符合 `playwright.config.ts` 的 `testMatch`（預設 `*.spec.ts` / `*.test.ts`）。
- 多根目錄工作區時，擴充套件只認第一個找到的 config，可在測試面板選擇要用的 config。

### test not found / No tests found

`testDir` 沒涵蓋到測試檔。修改 `playwright.config.ts`：

```ts
import { defineConfig } from "@playwright/test";

export default defineConfig({
  testDir: "./tests",     // 依實際測試資料夾調整；原筆記用 "../." 是把整個上層都掃進去，範圍過大會變慢
});
```

### 瀏覽器啟動失敗（Linux / WSL）

```shell
npx playwright install --with-deps
```

### 測試不穩定（flaky）

- 避免 `page.waitForTimeout()`，改用 `await expect(locator).toBeVisible()` 等自動等待的斷言。
- 設定 `retries` 與 `trace: "on-first-retry"` 保留失敗現場：

  ```ts
  export default defineConfig({
    retries: 2,
    use: { trace: "on-first-retry", screenshot: "only-on-failure" },
  });
  ```

---

<a id="ref"></a>

## 📚 參考資料

- [Playwright 官方文件](https://playwright.dev/docs/intro)
- [Playwright Test for VSCode](https://marketplace.visualstudio.com/items?itemName=ms-playwright.playwright)
- [playwright-vscode 原始碼](https://github.com/microsoft/playwright-vscode)
- [Test configuration](https://playwright.dev/docs/test-configuration)
- [node-mssql](https://github.com/tediousjs/node-mssql)
