# WSL + Ubuntu

> 在 Windows 上裝 WSL2 與 Ubuntu、終端機環境設定（zsh / oh-my-zsh / Powerlevel10k），以及帳號權限、網路與記憶體調整。

## 目錄

- [📥 安裝 WSL 與 Ubuntu](#install)
- [⬆️ 升級到 WSL 2](#wsl2)
- [📂 檔案互通路徑](#path)
- [💾 記憶體與資源設定](#resource)
- [🖥️ Windows Terminal](#terminal)
- [🐚 zsh 與 oh-my-zsh](#zsh)
- [🎨 Powerlevel10k](#p10k)
- [🧩 好用的 zsh 外掛](#plugins)
- [👤 使用者與權限](#user)
- [🌐 網路與 port 轉發](#network)
- [🐍 Python 環境](#python)
- [📦 apt 常用指令](#apt)
- [🔧 其他常用指令](#misc)
- [🔀 功能重疊的替代方案](#alt)
- [📚 參考資料](#ref)

---

<a id="install"></a>

## 📥 安裝 WSL 與 Ubuntu

### 現在的做法：一行指令

Windows 10 2004 以後，以**系統管理員**開啟 PowerShell：

```powershell
wsl --install                      # 裝 WSL2 + 預設的 Ubuntu
wsl --install -d Ubuntu-22.04      # 指定發行版
wsl --list --online                # 看有哪些發行版可裝
```

這一行會一併啟用所需的 Windows 功能、下載核心、設定 WSL2 為預設版本，然後要求重開機。重開後第一次啟動會要你建立 Linux 帳號與密碼。

### 舊的手動流程

較舊的 Windows 版本沒有 `wsl --install`，要自己開功能：

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

或從**控制台 → 程式與功能 → 開啟或關閉 Windows 功能**勾選：

- 適用於 Linux 的 Windows 子系統
- 虛擬機器平台（WSL 2 需要）

![Windows 功能中要勾選的兩個項目](https://raw.githubusercontent.com/kevin6655111/notes/main/images/wsl-windows-features.png)

重開機後確認：

```powershell
wsl --list
```

看到「沒有任何已安裝的發行版本」就是裝好了，接著從 Microsoft Store 安裝 Ubuntu。

### 啟動與基本確認

```powershell
wsl                      # 進入預設發行版
```

```shell
lsb_release -a           # 查看 Ubuntu 版本
sudo apt update          # 同步套件資料庫
sudo apt upgrade         # 更新已安裝的套件
```

---

<a id="wsl2"></a>

## ⬆️ 升級到 WSL 2

```powershell
wsl -l -v                              # 查看目前每個發行版是 1 還是 2
wsl --set-version Ubuntu-22.04 2       # 單一發行版轉成 WSL2
wsl --set-default-version 2            # 之後新裝的都用 WSL2
```

轉換需要的前置條件：

1. **啟用虛擬機器平台**

   ```powershell
   dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
   ```

2. **BIOS 的虛擬化要開啟**。確認方式是工作管理員 → 效能 → CPU，看「虛擬化」是否顯示已啟用。沒有的話要進 BIOS 打開，Intel 叫 VT-x、AMD 叫 SVM。

   ![工作管理員顯示虛擬化已啟用](https://raw.githubusercontent.com/kevin6655111/notes/main/images/wsl-cpu-virtualization.png)

3. **舊版 Windows 需要手動裝核心更新包**，從 [Linux kernel update package](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi) 下載。現在用 `wsl --update` 就好。

> **WSL2 不需要完整的 Hyper-V。** 只要「虛擬機器平台」這個功能，Windows 家庭版也能跑。網路上流傳的 Hyper-V.cmd 腳本是給 Docker Desktop 舊版用的，裝 WSL2 不需要。
>
> Hyper-V 在 Windows 功能裡是另一個獨立項目，位置如下。專業版以上才有，跑其他虛擬機時才需要開。
>
> ![Windows 功能中的 Hyper-V 項目](https://raw.githubusercontent.com/kevin6655111/notes/main/images/wsl-hyper-v.png)

### 磁碟空間自動回收

WSL2 的虛擬磁碟只會長大不會縮小。開啟 sparse 模式讓刪掉的空間能還給 Windows：

```powershell
wsl --manage Ubuntu-22.04 --set-sparse true
```

> WSL2 是虛擬機，會有一個 `vmmem` 行程常駐佔用記憶體。不用的時候 `wsl --shutdown` 可以完全停掉。

---

<a id="path"></a>

## 📂 檔案互通路徑

| 方向 | 路徑 |
|------|------|
| Linux 讀 Windows | `/mnt/c`、`/mnt/d`，對應各個磁碟機 |
| Windows 讀 Linux | 檔案總管輸入 `\\wsl.localhost\Ubuntu-22.04` |

![在檔案總管輸入路徑存取 WSL 檔案系統](https://raw.githubusercontent.com/kevin6655111/notes/main/images/wsl-explorer-path.png)

> 舊的 `\\wsl$` 寫法仍可用（上圖即是），但官方現在建議 `\\wsl.localhost`。

**跨系統存取很慢。** 專案檔案放在 Linux 家目錄（`~/`）裡，效能比放在 `/mnt/c` 好非常多，差距可以到數倍。VS Code 用 Remote-WSL 開啟 Linux 端的資料夾就是為了這個。

> WSL2 的檔案實際存在一個 `ext4.vhdx` 虛擬磁碟裡，**不要從 Windows 直接去改那個檔案或裡面的內容**，只能透過 `\\wsl.localhost` 存取。

---

<a id="resource"></a>

## 💾 記憶體與資源設定

WSL2 預設會吃掉相當比例的實體記憶體。建立 `.wslconfig` 限制：

```ini
[wsl2]
memory=16GB
processors=8
swap=8GB
```

存在 `C:\Users\<你的帳號>\.wslconfig`，用 `Win + R` 輸入 `%UserProfile%` 可以直接開到那個資料夾。

改完要重啟 WSL 才生效：

```powershell
wsl --shutdown
```

---

<a id="terminal"></a>

## 🖥️ Windows Terminal

從 Microsoft Store 安裝。Windows 11 已內建。

值得換掉內建主控台的理由：支援捲動、分頁、多重窗格，輸入中文時游標不會亂跳，也能自訂字體與配色。後面 Powerlevel10k 的圖示需要特殊字體，也要靠它設定。

---

<a id="zsh"></a>

## 🐚 zsh 與 oh-my-zsh

### 安裝 zsh

```shell
echo $SHELL                # 看目前用的是哪個 shell
sudo apt install zsh
cat /etc/shells            # 確認 zsh 已登記
chsh -s $(which zsh)       # 切換預設 shell
```

改完要重新登入才生效。

### 安裝 oh-my-zsh

```shell
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

設定檔是 `~/.zshrc`，主題與外掛都在那裡設定。

---

<a id="p10k"></a>

## 🎨 Powerlevel10k

提示字元主題，會顯示 git 分支狀態、指令執行時間、錯誤碼等資訊。

### 安裝

```shell
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

在 `~/.zshrc` 設定主題：

```shell
ZSH_THEME="powerlevel10k/powerlevel10k"
```

`ZSH_THEME` 就在 `~/.zshrc` 上方，預設值是 `robbyrussell`，把引號裡的值換掉即可。

![zshrc 中的 ZSH_THEME 設定位置](https://raw.githubusercontent.com/kevin6655111/notes/main/images/zsh-theme.png)

### 安裝字型

Powerlevel10k 用到大量圖示，需要 Nerd Font 才顯示得出來。

1. 從 [Nerd Fonts](https://www.nerdfonts.com/font-downloads) 下載，建議 **FiraCode Nerd Font**
2. 解壓後選 `FiraCodeNerdFontPropo-Regular.ttf`，右鍵安裝
3. Windows Terminal → 設定 → 設定檔 → 預設值 → 外觀 → 字體，選 `FiraCode NFM`

字型沒裝好的症狀是提示字元出現一堆問號或方框。

### 設定外觀

```shell
exec $SHELL      # 重啟 shell，第一次會自動跳出設定精靈
p10k configure   # 之後想重新設定就下這個
```

設定結果存在 `~/.p10k.zsh`，可以再手動微調。

---

<a id="plugins"></a>

## 🧩 好用的 zsh 外掛

在 `~/.zshrc` 的 `plugins` 陣列裡啟用：

```shell
plugins=(
    git
    z
    zsh-autosuggestions
    zsh-syntax-highlighting
)
```

| 外掛 | 功能 | 觸發方式 |
|------|------|----------|
| `z` | 跳到之前去過的目錄 | `z 部分目錄名` |
| `zsh-autosuggestions` | 依歷史紀錄灰字提示 | 按 `→` 接受補全 |
| `zsh-syntax-highlighting` | 指令有效無效顯示不同顏色 | 輸入時即時上色 |

`z` 是 oh-my-zsh 內建，另外兩個要自己 clone：

```shell
git clone https://github.com/zsh-users/zsh-autosuggestions \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions

git clone https://github.com/zsh-users/zsh-syntax-highlighting \
  ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

> **不要用 `sudo git clone`。** 那會讓外掛檔案變成 root 所有，之後更新會失敗。

`zsh-syntax-highlighting` 要放在 plugins 陣列的**最後一個**，否則會影響其他外掛。

### FZF 模糊搜尋

```shell
sudo apt install fzf
```

`~/.zshrc` 加上，位置要在 `source $ZSH/oh-my-zsh.sh` **之後**：

```shell
source /usr/share/doc/fzf/examples/key-bindings.zsh
source /usr/share/doc/fzf/examples/completion.zsh
```

![zshrc 中 plugins 陣列與 fzf 的兩行 source](https://raw.githubusercontent.com/kevin6655111/notes/main/images/zsh-plugins-fzf.png)

| 快捷鍵 | 功能 |
|--------|------|
| `Ctrl + T` | 模糊搜尋檔案，選中後插入到目前指令 |
| `Ctrl + R` | 模糊搜尋指令歷史 |
| `Alt + C` | 模糊搜尋目錄並直接 cd 過去 |
| `**` + `Tab` | 對目前指令做情境補全 |

### 資源監視器

```shell
git clone https://github.com/aristocratos/bashtop
cd bashtop && bash bashtop
```

---

<a id="user"></a>

## 👤 使用者與權限

### 切換到 root

WSL 的 Ubuntu 預設沒有設定 root 密碼，要先設定才能 `su`：

```shell
sudo passwd root      # 設定 root 密碼
su root               # 切換到 root
exit                  # 離開
```

> WSL 另有一條捷徑，不需要 root 密碼也能直接以 root 進入，救援時很有用：
>
> ```powershell
> wsl -u root
> ```

### 新增與刪除帳號

```shell
sudo adduser <帳號>                  # 新增，會互動式問密碼與資訊
sudo usermod -aG sudo <帳號>         # 加入 sudo 群組
sudo deluser <帳號> sudo             # 移出 sudo 群組

sudo deluser --remove-home <帳號>    # 刪除帳號並移除家目錄
sudo usermod -l <新帳號> <舊帳號>     # 更名
```

測試新帳號有沒有 sudo 權限：

```shell
sudo ls -la /root
```

> `userdel <帳號>` 不會刪掉家目錄，用 `deluser --remove-home` 比較乾淨。

### 讓特定指令免密碼

```shell
sudo visudo
```

在檔案最後加上：

```text
<帳號>   ALL=(ALL) NOPASSWD: /bin/mount
```

要給予完整的 sudo 權限則是加在 `User privilege specification` 區塊，格式與 `root` 那行相同：

![sudoers 的 User privilege specification 區塊](https://raw.githubusercontent.com/kevin6655111/notes/main/images/sudoers-user.png)

> ⚠️ **修改 sudoers 一律用 `visudo`，不要直接編輯 `/etc/sudoers`。** `visudo` 會在存檔前檢查語法，寫錯會擋下來讓你修正。直接編輯寫壞的話，sudo 會完全無法使用，而你也沒有權限改回來。
>
> ⚠️ **絕對不要 `chmod 777 /etc/sudoers`。** sudo 偵測到 sudoers 可被任何人寫入時會拒絕執行，直接把自己鎖在門外。正確權限是 `440`，要改內容就用 `visudo`。
>
> 真的鎖住了，WSL 可以從 Windows 用 root 進去救：
>
> ```powershell
> wsl -u root
> chmod 440 /etc/sudoers
> ```

---

<a id="network"></a>

## 🌐 網路與 port 轉發

### Ubuntu 防火牆

```shell
sudo ufw status
sudo ufw allow 8080
sudo ufw delete allow 8080
```

### 讓區網其他電腦連得到 WSL 裡的服務

WSL2 有自己的虛擬網段，Windows 以外的機器連不進來。要在 Windows 上做 port 轉發：

```powershell
netsh interface portproxy add v4tov4 `
  listenport=3008 listenaddress=0.0.0.0 `
  connectport=3008 connectaddress=<WSL 的 IP>

netsh interface portproxy show all
netsh interface portproxy delete v4tov4 listenport=3008 listenaddress=0.0.0.0
```

WSL 的 IP 用 `hostname -I` 查。**這個 IP 每次重開機都會變**，重開後要重設，或寫成開機腳本自動更新。

Windows 防火牆也要開對應的 port。

### 查佔用 port 的行程

```powershell
netstat -ano | findstr :3008
taskkill /F /PID <PID>
```

### 掛載 Windows 共享資料夾

```shell
sudo apt install -y cifs-utils

sudo mount -t cifs -o credentials=/etc/cifs-credentials,rw,file_mode=0777,dir_mode=0777,nounix,sec=ntlmssp,vers=3.0,iocharset=utf8 \
  //<伺服器IP>/<分享名稱> /mnt/<掛載點>

sudo umount /mnt/<掛載點>
```

帳密寫在指令裡會留在 shell 歷史，改用 credentials 檔：

```shell
sudo tee /etc/cifs-credentials > /dev/null <<'EOF'
username=<帳號>
password=<密碼>
EOF
sudo chmod 600 /etc/cifs-credentials
```

> `wsl --shutdown` 或關閉 WSL 都會斷開掛載，重開後要重新 mount。要免密碼執行 mount 的設定見[使用者與權限](#user)。

---

<a id="python"></a>

## 🐍 Python 環境

Ubuntu 內建的 Python 版本較舊時，用 deadsnakes PPA 裝指定版本：

```shell
sudo add-apt-repository ppa:deadsnakes/ppa -y
sudo apt update
sudo apt install python3.11
python3.11 --version
```

需要的話一併裝開發相關模組：

```shell
sudo apt install python3.11-dev python3.11-venv python3.11-distutils python3.11-tk
```

### 虛擬環境

現在直接用內建的 `venv` 就好，不需要另外裝 virtualenv：

```shell
python3.11 -m venv myenv      # 建立
source myenv/bin/activate     # 啟用
deactivate                    # 離開
```

> 不要 `sudo pip install`。套件裝進系統目錄會跟 apt 管理的套件打架，新版的 Ubuntu 也會直接擋下來。一律在虛擬環境裡裝。

---

<a id="apt"></a>

## 📦 apt 常用指令

```shell
sudo apt update                    # 同步套件資料庫
sudo apt upgrade                   # 更新已安裝的套件
sudo apt full-upgrade              # 更新並允許移除衝突套件

sudo apt install <套件>
sudo apt install --reinstall <套件>
sudo apt remove <套件>             # 移除程式
sudo apt purge <套件>              # 移除程式與設定檔

apt search <關鍵字>                # 搜尋
apt show <套件>                    # 看說明、大小、版本
apt depends <套件>                 # 看相依哪些套件
apt rdepends <套件>                # 看被哪些套件相依

sudo apt autoremove                # 清掉沒人需要的相依套件
sudo apt clean                     # 清掉下載的 deb 快取
sudo apt autoclean                 # 只清掉已無法下載的舊版快取
sudo apt --fix-broken install      # 修復中斷的安裝
sudo apt-get check                 # 檢查相依關係是否損壞

sudo apt-get build-dep <套件>      # 安裝編譯該套件所需的開發環境
apt-get source <套件>              # 下載原始碼（需先啟用 deb-src 來源）
```

套件來源清單在 `/etc/apt/sources.list` 與 `/etc/apt/sources.list.d/`。

要用 `add-apt-repository` 加 PPA 前，得先有這個套件：

```shell
sudo apt install software-properties-common -y
```

> `apt` 與 `apt-get` 大多可互換。互動使用建議用 `apt`，輸出比較友善；寫在腳本裡則建議 `apt-get`，行為在版本間比較穩定。

---

<a id="misc"></a>

## 🔧 其他常用指令

```shell
grep -rl "<關鍵字>" <路徑>      # 遞迴搜尋含關鍵字的檔案，只列檔名
which c++ | tr ':' '\n'        # 列出執行檔位置，多個路徑時換行顯示
chmod +x <檔案>                 # 給予執行權限
```

SSH 服務的啟停（完整說明見 [Linux 安裝 OpenSSH Server](ssh-linux.md)）：

```shell
sudo service ssh start|status|stop|restart    # WSL 常用，systemd 未啟用時也能用
sudo systemctl start|status|stop|restart ssh  # 有 systemd 的環境
```

> WSL 舊版預設沒有啟用 systemd，所以 `service` 指令比 `systemctl` 可靠。新版可在 `/etc/wsl.conf` 加上 `[boot]` 與 `systemd=true` 開啟。

---

<a id="alt"></a>

## 🔀 功能重疊的替代方案

以下工具與目前採用的方案功能重疊，記錄下來備查，不需要重複安裝。

| 工具 | 與誰重疊 | 說明 |
|------|----------|------|
| [Starship](https://starship.rs/) | Powerlevel10k | 跨 shell 的提示字元主題，Bash、Fish 都能用。單用 zsh 的話兩者擇一即可 |
| [zsh-completions](https://github.com/zsh-users/zsh-completions) | FZF | 強化 Tab 補全與模糊搜尋。FZF 的 `Ctrl + T`、`**` + `Tab` 已涵蓋主要情境 |

```shell
# Starship
curl -sS https://starship.rs/install.sh | sh
echo 'eval "$(starship init zsh)"' >> ~/.zshrc

# zsh-completions
git clone https://github.com/zsh-users/zsh-completions \
  ${ZSH_CUSTOM:-${ZSH:-~/.oh-my-zsh}/custom}/plugins/zsh-completions
# 在 ~/.zshrc 的 source $ZSH/oh-my-zsh.sh 之前加上
# fpath+=${ZSH_CUSTOM:-${ZSH:-~/.oh-my-zsh}/custom}/plugins/zsh-completions/src
```

---

<a id="ref"></a>

## 📚 參考資料

- [WSL 官方文件](https://learn.microsoft.com/windows/wsl/)
- [WSL 進階設定 .wslconfig](https://learn.microsoft.com/windows/wsl/wsl-config)
- [Oh My Zsh](https://ohmyz.sh/)
- [Powerlevel10k](https://github.com/romkatv/powerlevel10k)
- [Nerd Fonts](https://www.nerdfonts.com/)
- [fzf](https://github.com/junegunn/fzf)
- [Oh My Zsh 主題一覽](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes)
- [fzf 模糊搜尋教學 - Red Hat](https://www.redhat.com/sysadmin/fzf-linux-fuzzy-finder)
- [Powerlevel10k 風格設定參考](https://www.onejar99.com/terminal-iterm2-zsh-powerlevel10k/)
