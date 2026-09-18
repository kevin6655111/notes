# 環境建置清單

> 新機器要裝的東西。上半部是每台都會裝的，分隔線以下是視需要再裝。勾選狀態是目前這台的實際情形。

## 目錄

- [✅ 常用](#common)
- [🔧 其他](#others)

---

<a id="common"></a>

## ✅ 常用

- [x] **Git**　→ [Git 使用筆記](git.md)、[換行字元 EOL / CRLF 規範](git-eol.md)
  Windows 裝 Git for Windows，會一併帶 Git Bash

**WSL 開發環境**

- [x] **WSL + Ubuntu**　→ [WSL + Ubuntu](wsl.md)
  `wsl --install -d Ubuntu`，含 zsh 與終端機環境設定
- [x] **Docker**　開發機用 Docker Desktop，記得開 WSL integration
- [x] **Node.js**　→ [Node.js 安裝（nvm）](nodejs.md)
  用 nvm 管版本，不要用 apt 直接裝
- [x] **yarn**　`npm i -g yarn`
- [x] **VS Code**　→ [VS Code 設定與擴充套件](vscode.md)
  擴充套件清單與設定都在那篇

**資料庫工具**

- [x] **PostgreSQL client**　`psql` 連線與備份用，不需要裝伺服器
- [ ] **[pgAdmin](https://www.pgadmin.org/)**　圖形介面管理工具，需要視覺化操作時再裝，日常查詢用 `psql` 就夠

---

<a id="others"></a>

## 🔧 其他

視需要再裝。

**壓縮**（擇一即可）

- [x] **[7-Zip](https://www.7-zip.org/)**　開源免費，壓縮解壓
- [ ] **[Bandizip](https://www.bandisoft.com/bandizip/)**　介面較新，個人使用免費

**檔案搜尋**（擇一即可）

- [ ] **[Everything](https://www.voidtools.com/)**　以檔名索引 NTFS，全碟即時搜尋最快
- [ ] **[Quick Search](https://www.glarysoft.com/quick-search/)**　Glarysoft 出品，介面更直覺，內建檔案類型篩選

**.NET / C#**

- [x] **Visual Studio 2022**　.NET 開發
- [ ] **[LINQPad](https://www.linqpad.net/)**　C# 片段測試與 LINQ 查詢　→ [LINQ](linq.md)

**C / C++**

- [x] **C / C++ 工具鏈**　→ [C++ 環境配置](cpp.md)

**Python**

- [ ] **[Anaconda3](https://www.anaconda.com/download)**　Python 環境與套件管理，需要跑資料科學/AI 相關腳本時裝

**圖資**

- [ ] **QGIS**　處理道路圖資　→ [QGIS 道路資料處理](qgis.md)

**繪圖 / 圖表**

- [ ] **[draw.io](https://www.drawio.com/)**　流程圖、架構圖，免費且可存成本機檔案

**桌面工具**

- [ ] **iTop Easy Desktop**　桌面圖示整理

**遠端軟體**（擇一即可）

- [ ] **[AnyDesk](https://anydesk.com/)**　連線快、介面單純，免費版可商用但有連線次數限制
- [ ] **[RustDesk](https://rustdesk.com/)**　開源，可自架 server，注重隱私或要免費商用選這個

**其它**
- [ ] **Revo Uninstaller**
- [ ] **Tailscale**