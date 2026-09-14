# Node.js 安裝（nvm）

> 用 nvm（Node Version Manager）管理 Node.js 版本：可切換版本、指定預設版本、在不同版本上安裝各自的套件。適用 Ubuntu / WSL。

## 目錄

- [📦 安裝 nvm](#nvm)
- [🟢 安裝 Node.js + npm](#node)
- [💡 npm 與 npx](#npm-npx)
- [❗ 錯誤排除](#errors)
- [📚 參考資料](#ref)

---

<a id="nvm"></a>

## 📦 安裝 nvm

主要步驟：先裝 nvm，再用 nvm 安裝各版本的 Node（含 npm）。

1. 更新相依套件

   ```shell
   sudo apt-get update
   sudo apt-get install build-essential libssl-dev
   ```

2. 到 <https://github.com/nvm-sh/nvm/releases> 確認 nvm 最新版本號。

3. 依版本號用 curl 抓取安裝腳本執行（`v0.40.7` 請換成上一步看到的最新版本）

   ```shell
   curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.7/install.sh | bash
   ```

4. 重新載入 shell 設定並驗證

   ```shell
   source ~/.profile     # bash；zsh 用 source ~/.zshrc，或直接重開終端機
   nvm -v
   ```

---

<a id="node"></a>

## 🟢 安裝 Node.js + npm

```shell
nvm ls-remote                 # 列出遠端所有可安裝的 Node 版本
nvm ls                        # 列出本機已安裝的版本

nvm install v20.0.0           # 安裝指定版本（含 npm）
nvm install --lts             # 安裝最新 LTS

nvm use v20.0.0               # 目前終端機切換到該版本
nvm alias default v20.0.0     # 設為預設，避免重開終端機後又恢復原本版本

node -v
npm -v
```

---

<a id="npm-npx"></a>

## 💡 npm 與 npx

| 指令 | 說明 |
|------|------|
| `npm install <pkg>` | 安裝到目前資料夾的 `node_modules`，永久保留 |
| `npm install -g <pkg>` | 安裝到全域（nvm 下是該 Node 版本專屬的全域） |
| `npx <pkg>` | 臨時下載最新版執行一次，用完不保留 |

```shell
npm install http-server       # 裝在專案資料夾
npm install -g http-server    # 裝在全域
npx http-server -h            # 不安裝，直接執行一次
```

---

<a id="errors"></a>

## ❗ 錯誤排除

### `sudo: node: command not found`

nvm 把 node 裝在使用者家目錄，`sudo` 的 PATH 找不到。建一個符號連結到系統路徑：

```shell
which node                                  # 例如 /home/kevin/.nvm/versions/node/v20.0.0/bin/node
sudo ln -s "$(which node)" /usr/bin/node
sudo ln -s "$(which npm)"  /usr/bin/npm
```

> 連結指向特定版本，之後用 `nvm use` 換版本時 `sudo node` 仍是舊版，需重建連結。

---

<a id="ref"></a>

## 📚 參考資料

- [nvm GitHub](https://github.com/nvm-sh/nvm)
- [Node.js 官方下載](https://nodejs.org/)