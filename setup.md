# 環境建置清單

> 新機器要裝的東西。上半部是每台都會裝的，分隔線以下是視需要再裝。勾選狀態是目前這台的實際情形。

## 目錄

- [✅ 常用](#common)
- [🔧 其他](#others)
- [🔐 授權金鑰](#license)

---

<a id="common"></a>

## ✅ 常用

- [x] **Git**　→ [Git 使用筆記](git.md)、[換行字元 EOL / CRLF 規範](git-eol.md)
  Windows 裝 Git for Windows，會一併帶 Git Bash
- [x] **WSL + Ubuntu**　→ [WSL + Ubuntu](wsl.md)
  `wsl --install -d Ubuntu`，含 zsh 與終端機環境設定
- [x] **Docker**　開發機用 Docker Desktop，記得開 WSL integration
- [x] **Node.js**　→ [Node.js 安裝（nvm）](nodejs.md)
  用 nvm 管版本，不要用 apt 直接裝
- [x] **yarn**　`npm i -g yarn`
- [x] **VS Code**　→ [VS Code 設定與擴充套件](vscode.md)
  擴充套件清單與設定都在那篇
- [x] **Visual Studio 2022**　.NET 開發
- [x] **C / C++ 工具鏈**　→ [C++ 環境配置](cpp.md)
- [x] **PostgreSQL client**　`psql` 連線與備份用，不需要裝伺服器

```shell
# 驗證
git --version
node -v && yarn -v
docker --version
psql --version
code --version
```

---

<a id="others"></a>

## 🔧 其他

視需要再裝。

- [x] **7-Zip**　壓縮解壓
- [ ] **Bandizip**　與 7-Zip 擇一
- [ ] **Everything**　全碟檔名即時搜尋，比內建快很多
- [ ] **Chocolatey**　Windows 套件管理，可批次安裝上面這些
- [ ] **LINQPad**　C# 片段測試與 LINQ 查詢　→ [LINQ](linq.md)
- [ ] **QGIS**　處理道路圖資　→ [QGIS 道路資料處理](qgis.md)
- [ ] **iTop Easy Desktop**　桌面圖示整理
- [ ] **GitHub Copilot**　需訂閱，替代方案見 [VS Code 的 AI 輔助](vscode.md#extensions)

Chocolatey 裝好後其餘可以一行帶過：

```powershell
choco install 7zip everything git docker-desktop vscode -y
```

---

<a id="license"></a>

## 🔐 授權金鑰

**金鑰不要寫進這個倉庫。** 這是公開的 GitHub repo，貼上去的序號會被爬蟲收走而失效，也可能牴觸授權條款。

放在密碼管理器、公司的授權管理系統，或加密的本機檔案。筆記裡只記錄去哪裡找：

```text
Visual Studio 2022：公司授權，向 IT 申請
LINQPad：個人購買，金鑰存在密碼管理器
```

> Visual Studio 有免費的 **Community** 版，個人開發與小型團隊適用，符合授權條件就不需要金鑰。
