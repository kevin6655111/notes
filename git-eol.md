# Git 換行字元 EOL / CRLF 規範

> 跨平台（Windows / Linux / macOS）專案中，換行字元的差異會造成「沒改過的檔案卻被標記異動」、diff 出現 `^M`、Visual Studio 警告行尾不一致等問題。本篇整理 EOL 的基礎知識、Git 的轉換機制，以及團隊的規範決議。

## 目錄

- [⚡ TL;DR](#tldr)
- [📖 什麼是換行](#what)
- [✍️ 寫入 EOL](#write)
- [👀 讀取 / 檢視 EOL](#read)
- [🌐 跨平台取得目前 OS 的 EOL](#os-eol)
- [⚠️ Visual Studio 的 Inconsistent line ending style 警告](#vs-warning)
- [🔁 Git 概念：正規化與轉換](#git-concept)
- [⚙️ Git 設定：core.autocrlf / core.safecrlf / core.eol](#git-config)
- [📄 .gitattributes](#gitattributes)
- [🧪 遇到的情境](#cases)
- [🔧 進行中的專案套用新規範](#renormalize)
- [✅ 團隊規範決議](#rules)
- [📚 參考資料](#ref)

---

<a id="tldr"></a>

## ⚡ TL;DR

```shell
# 先確認沒有 untracked 檔案（git status 乾淨）
echo "* text=auto eol=lf" > .gitattributes
git add --renormalize .
git commit -m "Introduce end-of-line normalization"

# 列出工作目錄與索引的 EOL 狀態
git ls-files --eol

# 重設工作目錄的 EOL（依 .gitattributes 重新 checkout）
git rm -r .
git reset --hard
```

---

<a id="what"></a>

## 📖 什麼是換行

- 英文：newline、line ending、end-of-line (EOL)、line feed (LF)、line break
- 加在一行文字最後的特殊字元，其後的字元會出現在下一行。實際編碼依硬體平台或作業系統而不同。

### 表示法

| OS | 縮寫 | 符號 |
|----|------|------|
| Windows | CR LF | `\r\n` |
| Unix / macOS | LF | `\n` |

| 全名 | 縮寫 | 16 進位 | 10 進位 (ASCII) | 符號 |
|------|------|:------:|:---------------:|------|
| Carriage Return | CR | `0x0D` | 13 | `\r` |
| Line Feed | LF | `0x0A` | 10 | `\n` |

> CR / LF 的名稱源自打字機：CR 是把滾筒（carriage）推回行首，LF 是把紙往上捲一行。影片說明：<https://www.youtube.com/watch?v=D_TjvwjnAdM>

---

<a id="write"></a>

## ✍️ 寫入 EOL

以 C# 產出各種換行的檔案，方便測試：

```csharp
File.WriteAllText(@"CRLF.txt", "__\r\n__\r\n");
File.WriteAllText(@"LF.txt", "__\n__\n");
File.WriteAllText(@"MIXED.txt", "__\r\n__\n");

File.WriteAllBytes(@"Byte-CRLF.txt", new byte[] { 95, 95, 13, 10, 95, 95, 13, 10 });
File.WriteAllBytes(@"Byte-LF.txt", new byte[] { 95, 95, 10, 95, 95, 10 });
File.WriteAllBytes(@"Byte-MIXED.txt", new byte[] { 95, 95, 13, 10, 95, 95, 10 });

File.WriteAllBytes(@"Hex-CRLF.txt", new byte[] { 0x5F, 0x5F, 0x0D, 0x0A, 0x5F, 0x5F, 0x0D, 0x0A });
File.WriteAllBytes(@"Hex-LF.txt", new byte[] { 0x5F, 0x5F, 0x0A, 0x5F, 0x5F, 0x0A });
File.WriteAllBytes(@"Hex-MIXED.txt", new byte[] { 0x5F, 0x5F, 0x0D, 0x0A, 0x5F, 0x5F, 0x0A });
```

---

<a id="read"></a>

## 👀 讀取 / 檢視 EOL

### 編輯器擴充套件

- VS Code：Hex Editor（以 16 進位檢視 `0D 0A` / `0A`）、Code-eol（在編輯器中直接標示每行的 CRLF / LF）
- Visual Studio：Show Line Endings 2022

### hexdump

```shell
hexdump -C MIXED.txt
```

[hexdump 說明](http://manpages.ubuntu.com/manpages/focal/man1/hexdump.1.html)

### Git

```shell
git ls-files --eol        # 顯示每個檔案在索引 (i/) 與工作目錄 (w/) 的 EOL，以及套用的屬性 (attr/)
```

---

<a id="os-eol"></a>

## 🌐 跨平台取得目前 OS 的 EOL

- .NET

  ```csharp
  $"Hello {Environment.NewLine} World";
  ```

- Node.js

  ```js
  import os from 'os';

  `Hello ${os.EOL} World`;
  ```

---

<a id="vs-warning"></a>

## ⚠️ Visual Studio 的 Inconsistent line ending style 警告

- 與 Git 的設定無關。
- 當一個文字檔同時含有 CRLF 與 LF，Visual Studio 會警示「行尾樣式不一致」，請選擇統一的換行樣式。
- 可在設定中關閉此警示：工具 → 選項 → 環境 → 文件 → 取消勾選「載入時檢查行尾一致性」。

---

<a id="git-concept"></a>

## 🔁 Git 概念：正規化與轉換

- **正規化 (normalize)**：commit 時，把文字檔中的 CRLF 轉成 LF 寫進 Git 資料庫。
- **轉換 (conversion)**：checkout 時，把文字檔中的 LF 轉成 CRLF 寫到工作目錄。

Git for Windows 安裝時「Configuring the line ending conversions」的三個選項：

| 選項 | core.autocrlf | commit | checkout | 適用 |
|------|:-------------:|--------|----------|------|
| Checkout Windows-style, commit Unix-style | `true` | CRLF → LF | LF → CRLF | Windows |
| Checkout as-is, commit Unix-style | `input` | CRLF → LF | 不轉換 | Linux / macOS |
| Checkout as-is, commit as-is (default) | `false` | 不轉換 | 不轉換 | 跨平台、搭配 .gitattributes |

---

<a id="git-config"></a>

## ⚙️ Git 設定：core.autocrlf / core.safecrlf / core.eol

### core.autocrlf

```shell
git config --global core.autocrlf true    # commit 轉 LF，checkout 轉 CRLF
git config --global core.autocrlf input   # commit 轉 LF，checkout 不轉換
git config --global core.autocrlf false   # commit、checkout 均不轉換
```

### core.safecrlf

只在 `core.autocrlf` 為 `true` 或 `input` 時才有作用，控制 `git add` 遇到混合換行檔案時的行為。

```shell
git config --global core.safecrlf true    # 拒絕：fatal: LF would be replaced by CRLF in Byte-MIXED.txt
git config --global core.safecrlf false   # 允許（預設）
git config --global core.safecrlf warn    # 警告但不阻止：warning: LF will be replaced by CRLF in Byte-MIXED.txt.
```

### core.eol

```shell
git config --global core.eol lf      # checkout 時使用 LF
git config --global core.eol crlf    # checkout 時使用 CRLF
```

> **Warning**
> `core.autocrlf` 設為 `true` 或 `input` 時，`core.eol` 會被忽略。

---

<a id="gitattributes"></a>

## 📄 .gitattributes

- `core.autocrlf` 依賴每位成員的電腦設定，很難確保大家都正確配置。
- `.gitattributes` 放在專案內隨 repo 一起版控，**優先順序高於 `core.autocrlf`**，讓整個專案只需維持一份設定。
- 未指定 `text` 屬性的檔案，Git 才會回頭看 `core.autocrlf`。
- 後面的規則會覆蓋前面的規則。

### 屬性說明

| 寫法 | commit | checkout | 適用 |
|------|--------|----------|------|
| `*.md text` | CRLF → LF | 轉成本機系統的換行（Windows → CRLF，Linux → LF） | 跨平台程式碼、文件（最常用） |
| `*.sh text eol=lf` | CRLF → LF | 不轉換，維持 LF | 僅限 Unix 的檔案，如 bash script |
| `*.bat text eol=crlf` | CRLF → LF | LF → CRLF | 僅限 Windows 的檔案，如 bat / cmd script |
| `*.exe binary` | 不動 | 不動 | exe、影音等不可更動任何 byte 的檔案 |
| `* text=auto` | 由 Git 判斷是文字檔（同 `text`）或二進位檔（同 `binary`） | 同左 | 放第一行當預設值 |
| `*.abc -text` | 不轉換 | 不轉換 | 不論 `core.autocrlf` 如何，一律 as-is |

### 範例檔

```gitattributes
*         text=auto eol=lf   # 放第一行，作為預設
*.cs      text
*.xaml    text
*.csproj  text
*.sln     text
*.tt      text
*.ps1     text
*.cmd     text eol=crlf
*.bat     text eol=crlf
*.msbuild text
*.md      text
*.png     binary
*.jpeg    binary
*.sdf     binary
```

- [Web 專案的 .gitattributes 樣版](https://github.com/alexkaratarakis/gitattributes/blob/master/Web.gitattributes)

---

<a id="cases"></a>

## 🧪 遇到的情境

### 情境 1：沒改過的檔案突然被標記異動，且必須 commit 才會消失

1. A 電腦設定 `core.autocrlf = true`，git clone repo。
2. Git 在 A 電腦自動把 LF 轉成 CRLF 寫入工作目錄。
3. B 電腦設定 `core.autocrlf = false`。
4. 把 A 電腦的 repo 資料夾整個複製到 B 電腦。
5. B 電腦上所有檔案都顯示已異動，卻看不出增刪任何字元。
6. `git diff` 可看到每行多了 `^M`。

   > **Note**
   > - `^M` 的 `^` 代表 Ctrl，`^M` 即 Ctrl + M（也就是 Enter）。
   > - `^M` 看似兩個字元，在終端機上實際只有一個字元（CR）。
   > - `0x0D` 是十進位的 13，而 M 正好是第 13 個英文字母。

7. B 電腦的解決方式（三選一）：
   - 把所有檔案的 CRLF 改回 LF；改完仍會顯示異動，要 stage 之後異動狀態才會消失。
   - 把 B 電腦也設成 `core.autocrlf = true`。
   - 重新 `git clone` 一份再開發。

### 情境 2：沒改過的檔案突然被標記異動，但 git add 後異動就消失

1. A 電腦設定 `core.autocrlf = true`，git clone repo。
2. 手動或不小心把其中 X 檔案的 CRLF 改成 LF。
3. X 檔案顯示已異動。
4. `git add` X 檔案後，異動狀態消失，Git 視為沒有變更（因為 commit 時本來就會正規化成 LF，與索引內容相同）。

---

<a id="renormalize"></a>

## 🔧 進行中的專案套用新規範

修改 `core.autocrlf` 或新增 / 修改 `.gitattributes` 後，需要重新正規化既有檔案。執行前先確認 `git status` 乾淨、沒有 untracked 檔案。

Git 2.16 以後（建議）：

```shell
echo "* text=auto eol=lf" > .gitattributes
git add --renormalize .
git commit -m "Introduce end-of-line normalization"

git ls-files --eol        # 確認索引與工作目錄的 EOL

# 若工作目錄的 EOL 仍不對，強制依新規則重新 checkout
git rm -r .
git reset --hard
```

舊版 Git 的做法：

```shell
echo "* text=auto" > .gitattributes
git read-tree --empty
git add .
git commit -m "Introduce end-of-line normalization"
```

- [`git add --renormalize` 官方說明](https://git-scm.com/docs/git-add#Documentation/git-add.txt---renormalize)

---

<a id="rules"></a>

## ✅ 團隊規範決議

### core.autocrlf

- [ ] `true`：Checkout Windows-style, commit Unix-style
- [ ] `input`：Checkout as-is, commit Unix-style
- [x] `false`：Checkout as-is, commit as-is

> 以 `.gitattributes` 為主、`core.autocrlf` 為輔。有 `.gitattributes` 的專案，個人的 `core.autocrlf` 設什麼都不影響結果；設成 `false` 只是確保沒有 `.gitattributes` 的專案不會被個人設定改動。

### core.safecrlf

- [ ] `true`
- [ ] `false`
- [ ] `warn`

（尚未決議）

### .gitattributes

每個專案根目錄第一行放：

```gitattributes
* text=auto eol=lf
```

### Visual Studio 遇到 Inconsistent line ending style

關閉該警示設定即可（見[上方](#vs-warning)），交由 Git 統一處理。

---

<a id="ref"></a>

## 📚 參考資料

- [Wikipedia 換行](https://zh.wikipedia.org/wiki/%E6%8F%9B%E8%A1%8C)
- [Git 在 Windows 平台處理斷行字元 (CRLF) 的注意事項](https://blog.miniasp.com/post/2013/09/15/Git-for-Windows-Line-Ending-Conversion-Notes)
- [Git gitattributes Document](https://git-scm.com/docs/gitattributes)
- [Git config Document](https://git-scm.com/docs/git-config)
- [理解 CRLF，LF](https://iter01.com/149469.html)
- [Useful .gitattributes Templates](https://github.com/alexkaratarakis/gitattributes)
- [處理 Git 斷行字元的問題](https://titangene.github.io/article/git-auto-crlf.html)
