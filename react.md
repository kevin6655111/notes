# React

> 建立專案，以及現行專案實際使用中的套件清單與用途。
>
> 套件表格取自現行專案的 `package.json`，已移除不再使用的項目。

## 目錄

- [🚀 建立專案](#create)
- [🧱 核心與建置](#core)
- [🎨 UI 與樣式](#ui)
- [🗺️ 地圖與地理空間](#geo)
- [📊 圖表](#chart)
- [🖱️ 表格、拖放與互動](#table)
- [📁 檔案處理](#file)
- [📡 即時通訊](#rtc)
- [🔧 前端工具](#util)
- [⚙️ 後端與共用](#backend)
- [🧪 開發與測試](#dev)
- [📚 參考資料](#ref)

---

<a id="create"></a>

## 🚀 建立專案

```shell
npx create-react-app my-app
cd my-app
yarn start          # 開發伺服器，預設 http://localhost:3000
yarn build          # 編譯與壓縮，產出 build/
```

`build/` 裡是編譯過的 `index.html` 與 JS，傳到伺服器就能執行。

> `src` 或 `public` 底下沒用到的檔案編譯前先移出專案，否則會被一起打包進去。

> **create-react-app 已經停止維護**，官方現在建議用 Vite 或框架級方案。新專案直接用 Vite：
>
> ```shell
> npm create vite@latest my-app -- --template react-ts
> ```

---

<a id="core"></a>

## 🧱 核心與建置

| 套件 | 用途 |
|------|------|
| `react`、`react-dom` | 核心 |
| `react-router-dom` | 路由 |
| `react-scripts` | create-react-app 的建置腳本 |
| `@craco/craco` | 不 eject 就覆寫 CRA 的 webpack 設定 |
| `vite`、`@vitejs/plugin-react` | Vite 建置與 React 支援 |
| `web-vitals` | 前端效能指標蒐集 |

---

<a id="ui"></a>

## 🎨 UI 與樣式

| 套件 | 用途 |
|------|------|
| `@mui/material`、`@mui/icons-material` | Material UI 元件與圖示 |
| `@emotion/react`、`@emotion/styled` | MUI 的樣式引擎，安裝 MUI 時必裝 |
| `@mui/x-date-pickers` | 日期與時間選擇器 |
| `@mui/styles` | MUI v4 的舊版樣式 API，僅供舊元件相容 |
| `@mui/styled-engine-sc` | 把 MUI 的樣式引擎換成 styled-components |
| `styled-components` | CSS-in-JS |
| `bootstrap`、`react-bootstrap` | Bootstrap 元件 |
| `react-bootstrap-range-slider` | 範圍滑桿 |
| `react-icons` | 集合多套圖示庫 |
| `framer-motion` | 動畫 |
| `react-toastify` | 浮動訊息提示 |

> `@mui/styles` 在 MUI v6 已經不建議使用，新元件請直接用 `sx` 或 `styled`。

---

<a id="geo"></a>

## 🗺️ 地圖與地理空間

| 套件 | 用途 |
|------|------|
| `leaflet`、`react-leaflet` | 地圖底圖與 React 綁定 |
| `react-leaflet-cluster` | 大量標記點的聚合顯示 |
| `leaflet.heat` | 熱力圖圖層 |
| `deck.gl` 及 `@deck.gl/core`、`layers`、`react` | WebGL 大量資料圖層 |
| `@deck.gl/aggregation-layers` | 網格、六角格等聚合圖層 |
| `supercluster` | 點位聚合演算法 |
| `@turf/turf` | 幾何運算，緩衝區、交集、距離 |
| `proj4` | 座標系統轉換，WGS84 與 TWD97 互轉 |
| `geojson`、`geojson-vt`、`vt-pbf` | GeoJSON 處理與向量圖磚產生 |
| `@ngageoint/geopackage` | 讀寫 GeoPackage 格式 |
| `osmbuildings` | OSM 3D 建物圖層 |
| `mapillary-js` | 街景服務 |
| `wgsl_reflect` | WGSL 著色器解析，deck.gl 的相依 |

座標系統的說明見 [QGIS 道路資料處理](qgis.md)。

---

<a id="chart"></a>

## 📊 圖表

| 套件 | 用途 |
|------|------|
| `chart.js`、`react-chartjs-2` | Chart.js 與 React 綁定 |
| `recharts` | 宣告式圖表元件 |
| `d3` | 底層資料視覺化，自訂圖形時使用 |

---

<a id="table"></a>

## 🖱️ 表格、拖放與互動

| 套件 | 用途 |
|------|------|
| `mui-datatables` | 帶分頁、排序、篩選的表格 |
| `@dnd-kit/core`、`sortable`、`utilities` | 拖放排序，現行維護中的方案 |
| `react-beautiful-dnd` | 舊的拖放套件，**已停止維護** |
| `react-draggable` | 可拖移元素 |
| `react-rnd` | 可拖移且可縮放的視窗 |
| `react-to-print` | 列印指定區塊 |
| `react-image-magnifiers` | 圖片放大鏡 |

> `react-beautiful-dnd` 官方已停止維護，新功能改用 `@dnd-kit`。兩者目前並存是遷移中的狀態。

---

<a id="file"></a>

## 📁 檔案處理

| 套件 | 用途 |
|------|------|
| `xlsx` | 讀寫 Excel |
| `exceljs` | 產生帶樣式的 Excel |
| `file-saver` | 瀏覽器端觸發下載 |
| `jszip` | 前端打包 ZIP |
| `fflate` | 輕量快速的壓縮解壓 |
| `docx` | 程式產生 Word 文件 |
| `docxtemplater`、`pizzip` | 用範本填值產生 Word，`pizzip` 負責解壓 docx |
| `docxtemplater-image-module-free` | 讓範本能塞圖片 |
| `xml2js` | 解析 XML，處理 docx 內部結構時使用 |
| `archiver`、`yauzl` | 後端壓縮與解壓 |

---

<a id="rtc"></a>

## 📡 即時通訊

| 套件 | 用途 |
|------|------|
| `ws`、`bufferutil`、`utf-8-validate` | WebSocket，後兩者是效能加速的原生模組 |
| `agora-rtc-sdk-ng` | 聲網即時音視訊 |
| `@volcengine/rtc` | 火山引擎即時音視訊 |

---

<a id="util"></a>

## 🔧 前端工具

| 套件 | 用途 |
|------|------|
| `axios` | HTTP 請求 |
| `lodash` | 通用工具函式 |
| `moment` | 日期格式化，**官方已進入維護模式** |
| `date-fns` | 日期處理，模組化、體積小 |
| `crypto-js` | 前端雜湊與加解密 |
| `nanoid` | 產生短的唯一 ID |
| `worker-loader` | 把檔案打包成 Web Worker |
| `es6-promise-pool` | 限制並行任務數量 |

> `moment` 官方已宣告進入維護模式，新程式建議用 `date-fns`。兩者並存是遷移中的狀態。

---

<a id="backend"></a>

## ⚙️ 後端與共用

後端是 NestJS，前端專案會用到的共用套件也列在這裡。

| 套件 | 用途 |
|------|------|
| `@nestjs/core`、`common`、`platform-express` | NestJS 核心 |
| `@nestjs/typeorm`、`typeorm` | ORM，見 [PostgreSQL 筆記](postgresql.md) |
| `pg`、`mssql` | PostgreSQL 與 MSSQL driver |
| `@nestjs/swagger`、`rapidoc`、`redoc` | API 文件產生與呈現 |
| `@nestjs/bullmq`、`bullmq`、`ioredis` | 佇列與 Redis |
| `@nestjs/schedule`、`cron` | 排程 |
| `@nestjs/websockets`、`platform-ws` | WebSocket 閘道 |
| `@nestjs/throttler`、`helmet` | 流量限制與安全標頭 |
| `class-validator`、`class-transformer` | DTO 驗證與轉型 |
| `jsonwebtoken`、`bcryptjs` | JWT 與密碼雜湊 |
| `cookie`、`cookie-parser`、`express-session` | Cookie 與 session |
| `multer` | 接收上傳檔案 |
| `@aws-sdk/client-s3`、`lib-storage`、`s3-request-presigner` | S3 相容儲存，實際連的是 MinIO |
| `sharp` | 影像縮放與轉檔 |
| `nodemailer` | 寄信 |
| `firebase-admin` | 推播 |
| `svg-captcha` | 登入驗證碼 |
| `winston`、`winston-daily-rotate-file` | 日誌與輪替 |
| `prom-client`、`swagger-stats` | Prometheus 指標與 API 統計 |
| `dotenv`、`js-yaml`、`jsonc-parser` | 設定檔讀取 |
| `fs-extra`、`stream-buffers`、`ftp` | 檔案系統、串流與 FTP |
| `cors`、`compression-webpack-plugin` | 跨域與壓縮 |
| `@nestjs/config` | 設定模組，包裝環境變數讀取 |
| `@nestjs/axios` | 把 axios 包成 Nest 的 HttpService |
| `@nestjs/mapped-types` | 由既有 DTO 衍生出 Partial、Pick 等型別 |
| `@nestjs/microservices` | 微服務傳輸層 |
| `express` | web-demo 的 API 直接使用 |
| `reflect-metadata` | 裝飾器的中繼資料，NestJS 與 TypeORM 的前提 |
| `rxjs` | NestJS 的資料流基礎 |

---

<a id="dev"></a>

## 🧪 開發與測試

| 套件 | 用途 |
|------|------|
| `vitest` | 單元與整合測試 |
| `@testing-library/react`、`jest-dom`、`user-event`、`dom` | React 元件測試 |
| `jsdom` | 測試用的瀏覽器環境模擬 |
| `supertest` | HTTP API 測試 |
| `prettier` | 排版，見 [程式碼統一格式化](format.md) |
| `typescript` | 型別檢查 |
| `concurrently` | 同時跑多個指令 |
| `nodemon` | 檔案變更自動重啟 |
| `@faker-js/faker` | 產生假資料 |
| `wait-on` | 等待服務起來再繼續 |
| `@redocly/cli` | 產生 API 文件 HTML |
| `javascript-obfuscator` | 程式碼混淆 |
| `@babel/plugin-proposal-private-property-in-object` | 解決 CRA 的 babel 警告 |
| `@nestjs/testing` | Nest 模組的測試工具 |
| `ts-node` | 直接執行 TypeScript |
| `tsconfig-paths`、`vite-tsconfig-paths` | 讓路徑別名在執行期與建置期都能解析 |

web-demo 是 monorepo，`@road-patrol/shared` 是 `apps/web` 與 `apps/api` 共用型別與常數的內部套件，不會發佈到 npm。

端對端測試用 Playwright，見 [Playwright + VS Code UI 測試](playwright.md)。

---

<a id="ref"></a>

## 📚 參考資料

- [React 官方文件](https://react.dev/)
- [React Router](https://reactrouter.com/)
- [Material UI](https://mui.com/)
- [Vite](https://vite.dev/)
- [deck.gl](https://deck.gl/)
- [Leaflet](https://leafletjs.com/)
- [Turf.js](https://turfjs.org/)
