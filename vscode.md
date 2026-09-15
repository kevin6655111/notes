# VS Code 設定與擴充套件

> 常用設定、擴充套件清單與識別碼，以及在 WSL 下使用時的注意事項。

## 目錄

- [🧩 擴充套件](#extensions)
- [📦 批次安裝與備份](#batch)
- [👥 專案推薦清單](#recommend)
- [⚙️ 設定](#settings)
- [🐧 WSL remote 的注意事項](#wsl)
- [⌨️ 常用快捷鍵](#keys)
- [📚 參考資料](#ref)

---

<a id="extensions"></a>

## 🧩 擴充套件

安裝方式：側邊欄的 Extensions 搜尋名稱，或用命令列 `code --install-extension <識別碼>`。

### AI 輔助

- [x] **Windsurf Plugin**（前身為 Codeium）— `codeium.codeium`　行內補全與 AI 對話，個人使用免費
- [x] **Claude Code** — `anthropic.claude-code`　在編輯器內操作 Claude Code，可讀寫檔案與執行指令

### Git

- [x] **Git Graph** — `mhutchie.git-graph`　圖形化的分支與 commit 樹，可直接操作
- [x] **Git History** — `donjayamanne.githistory`　檔案或行的歷史紀錄，快捷鍵 `Alt + H`
- [x] **GitLens** — `eamodio.gitlens`　行內顯示最後修改者與 commit，blame 強化

> GitLens 預設會在每一行行尾顯示 blame 註解，覺得吵可以關掉：`"gitlens.currentLine.enabled": false`。

### 前端

- [x] **ES7+ React/Redux/React-Native snippets** — `dsznajder.es7-react-js-snippets`　輸入縮寫按 Enter 展開樣板，例如 `rafce`
- [x] **Live Server** — `ritwickdey.liveserver`　靜態頁面起一個有熱重載的本機伺服器

### 測試

VS Code 內建**測試面板**（側邊欄的燒杯圖示），下面這些套件都是把測試結果接進那個面板，裝了就能在程式碼行號旁直接按執行與除錯。

- [x] **Playwright Test for VSCode** — `ms-playwright.playwright`　端對端測試，可錄製操作產生程式碼
- [ ] **Vitest** — `vitest.explorer`　Vite 生態的單元測試，官方出品
- [ ] **Jest** — `orta.vscode-jest`　專案用 Jest 時裝這個
- [ ] **REST Client** — `humao.rest-client`　用 `.http` 純文字檔發 API 請求，檔案可進版控

| 語言 / 框架 | 怎麼跑測試 |
|-------------|------------|
| JavaScript / TypeScript | 裝 Vitest 或 Jest 套件，看專案用哪個 |
| 端對端 | Playwright |
| Python | 不用另外裝，`ms-python.python` 內建 pytest 與 unittest 探索 |
| C# | 不用另外裝，C# Dev Kit 內建測試總管 |
| C / C++ | CMake Tools 已整合 CTest |

> **API 測試建議用 REST Client 而不是 Postman 這類 GUI 工具。** 請求寫在 `.http` 檔裡，可以跟程式碼一起進版控、一起做 code review，換機器也不用重新匯入。

Playwright 的使用見 [Playwright + VS Code UI 測試](playwright.md)。

### 程式規範檢查

- [x] **ESLint** — `dbaeumer.vscode-eslint`　JavaScript / TypeScript 語法檢查
- [x] **Prettier** — `esbenp.prettier-vscode`　程式碼排版
- [x] **Code Spell Checker** — `streetsidesoftware.code-spell-checker`　英文拼字檢查
- [x] **Error Lens** — `usernamehw.errorlens`　把錯誤與警告直接顯示在該行行尾

ESLint 與 Prettier 的專案設定見 [程式碼統一格式化](format.md)。

### 語言支援

- [x] **C/C++** — `ms-vscode.cpptools`　IntelliSense、除錯
- [x] **C/C++ Extension Pack** — `ms-vscode.cpptools-extension-pack`　含 CMake Tools 等一整組
- [x] **C# Dev Kit** — `ms-dotnettools.csharp`　C# 語言服務與除錯
- [x] **CMake** — `twxs.cmake`　CMakeLists 語法高亮
- [x] **CMake Tools** — `ms-vscode.cmake-tools`　設定、建置、除錯整合
- [x] **Python** — `ms-python.python`　直譯器選擇、除錯、測試
- [x] **Pylance** — `ms-python.vscode-pylance`　Python 語言服務與型別檢查
- [x] **PHP Intelephense** — `bmewburn.vscode-intelephense-client`　PHP 語言服務
- [x] **PHP Debug** — `xdebug.php-debug`　搭配 Xdebug 除錯
- [x] **Dart / Flutter** — `dart-code.dart-code`、`dart-code.flutter`　Dart 與 Flutter 開發

C/C++ 的專案設定見 [C++ 環境配置](cpp.md)。

> **擴充套件包會自動帶進相依套件**，清單裡會多出沒印象裝過的項目，是正常的：
>
> | 你裝的 | 自動帶進來的 |
> |--------|--------------|
> | C/C++ Extension Pack | `ms-vscode.cpp-devtools`、`ms-vscode.cpptools-themes` |
> | Python | `ms-python.debugpy`、`ms-python.vscode-python-envs` |
> | C# Dev Kit | `ms-dotnettools.vscode-dotnet-runtime` |
>
> 這些不需要也不該手動移除，砍掉主套件會一起帶走。

### 其他

- [x] **Remote - WSL** — `ms-vscode-remote.remote-wsl`　連進 WSL 開發
- [x] **Dev Containers** — `ms-vscode-remote.remote-containers`　在容器內開發
- [x] **Partial Diff** — `ryu1kn.partial-diff`　比對選取的兩段文字
- [x] **繁體中文語言包** — `ms-ceintl.vscode-language-pack-zh-hant`　介面中文化

---

<a id="batch"></a>

## 📦 批次安裝與備份

匯出目前已安裝的清單：

```shell
code --list-extensions > extensions.txt
```

在另一台機器一次裝回來：

```shell
# Linux / macOS
cat extensions.txt | xargs -n1 code --install-extension

# Windows PowerShell
Get-Content extensions.txt | ForEach-Object { code --install-extension $_ }
```

其他常用指令：

```shell
code --list-extensions --show-versions    # 連版本一起列
code --uninstall-extension <識別碼>
code --disable-extensions                 # 停用所有套件開啟，用來排查衝突
```

> VS Code 內建的 **Settings Sync** 也能同步設定、快捷鍵與套件清單，用 Microsoft 或 GitHub 帳號登入即可。手動匯出適合不想登入帳號的情況。

---

<a id="recommend"></a>

## 👥 專案推薦清單

在專案裡放 `.vscode/extensions.json`，其他人開啟這個資料夾時 VS Code 會提示安裝：

```json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "usernamehw.errorlens",
    "ms-playwright.playwright"
  ],
  "unwantedRecommendations": [
    "hookyqr.beautify"
  ]
}
```

搭配 `.vscode/settings.json` 一起進版控，新人 clone 下來就有一致的環境。

---

<a id="settings"></a>

## ⚙️ 設定

`Ctrl + Shift + P` → `Preferences: Open User Settings (JSON)` 直接編輯 `settings.json`。

| 位置 | 路徑 |
|------|------|
| Windows | `%APPDATA%\Code\User\settings.json` |
| Linux / WSL | `~/.config/Code/User/settings.json` |
| 專案層級 | `<專案>/.vscode/settings.json`，覆寫使用者設定 |

### 標題列顯示上一頁 / 下一頁

```json
{
  "window.commandCenter": true
}
```

開啟後標題列中央出現命令中心，左側就有返回與前進的箭頭按鈕。

### 自動儲存

`Ctrl + Shift + P` → `File: Toggle Auto Save` 可以快速切換，對應的設定是：

```json
{
  "files.autoSave": "afterDelay",
  "files.autoSaveDelay": 1000
}
```

| 值 | 時機 |
|----|------|
| `off` | 不自動儲存 |
| `afterDelay` | 停止輸入後經過 `files.autoSaveDelay` 毫秒 |
| `onFocusChange` | 離開該檔案時 |
| `onWindowChange` | 切換到其他程式時 |

> 專案有 format on save 時建議用 `onFocusChange`，`afterDelay` 會在打字打到一半觸發格式化。

### 其他常用設定

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "files.eol": "\n",
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,
  "editor.renderWhitespace": "boundary",
  "editor.rulers": [115]
}
```

`files.eol` 與換行字元的關係見 [Git 換行字元 EOL / CRLF 規範](git-eol.md)，格式化工具的完整設定見 [程式碼統一格式化](format.md)。

---

<a id="wsl"></a>

## 🐧 WSL remote 的注意事項

用 Remote - WSL 開發時，**擴充套件分成兩邊各自安裝**：

| 類型 | 裝在哪 | 例子 |
|------|--------|------|
| UI 擴充套件 | Windows 端 | 佈景主題、圖示、語言包 |
| Workspace 擴充套件 | WSL 端 | 語言服務、Linter、除錯器 |

所以同一個套件可能在 Windows 裝過了，進到 WSL 專案卻不會生效，Extensions 面板會把它列在 **Local - Installed** 並顯示一顆 `Install in WSL` 按鈕，要再按一次。

```shell
code --list-extensions     # 在 WSL 終端機執行，列的是 WSL 端的清單
```

> 症狀通常是「ESLint 在某些專案沒作用」或「Python 找不到直譯器」。先確認該套件是不是只裝在 Windows 端。

WSL 的圖形介面相關設定見 [OpenCV / WSL GUI 顯示](opencv.md)。

---

<a id="keys"></a>

## ⌨️ 常用快捷鍵

| 快捷鍵 | 功能 |
|--------|------|
| `Ctrl + Shift + P` | 命令選擇區，幾乎所有功能的入口 |
| `Ctrl + P` | 依檔名快速開檔 |
| `Ctrl + Shift + F` | 全專案搜尋 |
| `Ctrl + B` | 切換側邊欄 |
| `` Ctrl + ` `` | 切換終端機 |
| `F12` | 跳到定義 |
| `Alt + ←` / `Alt + →` | 上一個 / 下一個游標位置 |
| `Shift + Alt + F` | 格式化整份檔案 |
| `Ctrl + D` | 選取下一個相同的字 |
| `Alt + ↑` / `Alt + ↓` | 整行上下移動 |
| `Ctrl + K Ctrl + S` | 開啟快捷鍵設定 |

---

<a id="ref"></a>

## 📚 參考資料

- [VS Code 官方文件](https://code.visualstudio.com/docs)
- [settings.json 設定一覽](https://code.visualstudio.com/docs/getstarted/settings)
- [命令列介面](https://code.visualstudio.com/docs/editor/command-line)
- [Remote - WSL](https://code.visualstudio.com/docs/remote/wsl)
- [Extension Marketplace](https://marketplace.visualstudio.com/vscode)
