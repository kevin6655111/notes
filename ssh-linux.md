# Linux 安裝 OpenSSH Server

> 讓其他電腦能透過 SSH 連進這台 Linux 主機。適用 Ubuntu / Debian / CentOS / RHEL 等主流發行版。
>
> Windows 主機的做法見 [Windows 安裝 OpenSSH Server](ssh-windows.md)。

## 目錄

- [🔍 確認服務是否已安裝](#check)
- [📥 安裝 OpenSSH Server](#install)
- [🔥 開放防火牆 22 port](#firewall)
- [🌐 查詢主機 IP](#ip)
- [⚙️ 調整 SSH 設定](#config)
- [🔑 設定金鑰登入](#key)
- [🖥️ 從客戶端測試連線](#connect)
- [📄 簡化連線指令](#ssh-config)
- [🚨 常見問題排查](#trouble)
- [📋 快速指令總覽](#quick)
- [📚 參考資料](#ref)

---

<a id="check"></a>

## 🔍 確認服務是否已安裝

大多數雲端 Linux 映像檔（尤其是 Ubuntu Server）預設就已安裝並啟動。

```shell
sudo systemctl status ssh      # Ubuntu / Debian
sudo systemctl status sshd     # CentOS / RHEL
```

| 看到什麼 | 代表 |
|----------|------|
| `Active: active (running)` | 已在執行，直接跳到[開放防火牆](#firewall) |
| `Unit ssh.service could not be found` | 尚未安裝，往下看 |

---

<a id="install"></a>

## 📥 安裝 OpenSSH Server

```shell
# Ubuntu / Debian
sudo apt update
sudo apt install openssh-server -y

# CentOS / RHEL / Rocky / AlmaLinux
sudo dnf install openssh-server -y     # 較舊的版本用 yum

# Arch Linux
sudo pacman -S openssh
```

啟動並設定開機自動啟動：

```shell
sudo systemctl enable ssh --now      # Ubuntu / Debian
sudo systemctl enable sshd --now     # CentOS / RHEL
```

確認狀態：

```shell
sudo systemctl status ssh
```

---

<a id="firewall"></a>

## 🔥 開放防火牆 22 port

```shell
# Ubuntu（ufw）
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status

# CentOS / RHEL（firewalld）
sudo firewall-cmd --permanent --add-port=22/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

> ⚠️ **遠端操作時先開規則再 `ufw enable`。** 順序顛倒會立刻把自己的連線切斷。

**雲端主機還有第二層防火牆。** 除了主機內部的 ufw / firewalld，還要到平台後台的「安全群組 / Security Group / 防火牆規則」開放 inbound 22 port。這一步最常被忽略，症狀是 `Connection timed out` 而不是 `refused`。

---

<a id="ip"></a>

## 🌐 查詢主機 IP

```shell
hostname -I        # 只列 IP
ip a               # 完整網路介面資訊
```

雲端主機直接看後台顯示的公網 IP。`hostname -I` 列出的是內網位址。

---

<a id="config"></a>

## ⚙️ 調整 SSH 設定

設定檔在 `/etc/ssh/sshd_config`：

```shell
sudo nano /etc/ssh/sshd_config
```

常見調整項目：

```text
Port 22                      # 改成非標準 port 可降低被掃描機率
PermitRootLogin no           # 關閉 root 直接登入
PasswordAuthentication no    # 關閉密碼登入，強制只用金鑰
PubkeyAuthentication yes
```

改完**先驗證語法再重啟**，這一步能擋掉大部分把自己鎖在外面的意外：

```shell
sudo sshd -t                 # 沒有輸出就是設定檔沒問題
sudo systemctl restart ssh   # 或 sshd
```

> ⚠️ **改設定時保留一個已登入的 session 不要關。** 重啟 SSH 不會踢掉既有連線。另外開一個新視窗測試能不能連得進來，確認成功再關掉舊的。設定寫錯時這是唯一的救命繩。

> ⚠️ **關閉密碼登入之前，務必先確認金鑰登入測試成功。**

### 改 port 的兩個額外步驟

**Ubuntu 22.10 以後改用 socket activation**，只改 `sshd_config` 的 `Port` 不會生效，要另外改 socket 設定：

```shell
sudo systemctl edit ssh.socket
```

```ini
[Socket]
ListenStream=
ListenStream=2222
```

```shell
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

**RHEL 系列有 SELinux**，非標準 port 要先登記，否則服務起不來：

```shell
sudo semanage port -a -t ssh_port_t -p tcp 2222
```

改 port 後記得防火牆也要開新的 port，並確認舊的連線還能用再關掉 22。

---

<a id="key"></a>

## 🔑 設定金鑰登入

### 1. 在客戶端產生金鑰

```shell
ssh-keygen -t ed25519 -C "your_email@example.com"
```

一路 Enter 使用預設路徑 `~/.ssh/id_ed25519`。

> `ed25519` 比 `rsa` 短、快且安全，現在是預設建議。只有連非常舊的伺服器才需要 `-t rsa -b 4096`。

### 2. 上傳公鑰到伺服器

```shell
ssh-copy-id 使用者名稱@伺服器IP
ssh-copy-id -i ~/.ssh/id_ed25519.pub 使用者名稱@伺服器IP    # 指定金鑰
```

會要求輸入一次密碼，之後公鑰就寫進伺服器的 `~/.ssh/authorized_keys`。

### 3. Windows 客戶端沒有 ssh-copy-id

PowerShell 一行解決：

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh 使用者名稱@伺服器IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

或手動把 `id_ed25519.pub` 的內容貼進伺服器的 `~/.ssh/authorized_keys`。

### 4. 權限必須正確

SSH 對權限要求嚴格，過寬會直接拒絕金鑰登入且不說原因：

```shell
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
chown -R $USER:$USER ~/.ssh
```

家目錄本身也不能是群組可寫（`chmod 755 ~` 或更嚴）。

---

<a id="connect"></a>

## 🖥️ 從客戶端測試連線

```shell
ssh 使用者名稱@伺服器IP
ssh ubuntu@203.0.113.10
```

- 第一次連線會問是否信任主機金鑰指紋，輸入 `yes`
- 設定了金鑰登入就不會再問密碼
- 自訂 port 或指定金鑰：

```shell
ssh -p 2222 -i ~/.ssh/id_ed25519 使用者名稱@伺服器IP
```

連不上時加 `-v` 看詳細過程，`-vvv` 更詳細：

```shell
ssh -v 使用者名稱@伺服器IP
```

---

<a id="ssh-config"></a>

## 📄 簡化連線指令

在客戶端編輯 `~/.ssh/config`：

```text
Host myserver
    HostName 203.0.113.10
    User ubuntu
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host prod
    HostName 203.0.113.20
    User deploy
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60      # 每 60 秒送一次封包，避免閒置被斷線
```

之後直接：

```shell
ssh myserver
```

`scp`、`rsync`、`git` 也會套用這份設定，所以 `scp file.txt myserver:~/` 一樣可用。

---

<a id="trouble"></a>

## 🚨 常見問題排查

| 狀況 | 檢查方式 |
|------|----------|
| `Connection refused` | 服務沒跑或沒監聽。`sudo systemctl status ssh`、`sudo ss -tlnp \| grep :22` |
| `Connection timed out` | 封包被擋。檢查本機防火牆與**雲端安全群組** |
| `Permission denied (publickey)` | `authorized_keys` 內容或權限不對（700 / 600），或 `PubkeyAuthentication` 沒開 |
| root 登入被拒 | `sshd_config` 的 `PermitRootLogin no`，改用一般帳號登入再 `sudo` |
| 改了設定沒生效 | 沒重啟服務，或 Ubuntu 22.10+ 改 port 要一併改 `ssh.socket` |
| `Host key verification failed` | 伺服器重灌過，指紋變了。`ssh-keygen -R 伺服器IP` 清掉舊紀錄 |
| 閒置一陣子就斷線 | 客戶端設 `ServerAliveInterval 60` |

排查順序：**先確認服務有在監聽，再確認防火牆，最後才查金鑰。** 從 `refused` 或 `timed out` 就能判斷卡在哪一層。

---

<a id="quick"></a>

## 📋 快速指令總覽

以 Ubuntu 為例，可直接複製貼上：

```shell
# 1. 安裝
sudo apt update
sudo apt install openssh-server -y

# 2. 啟動並設定開機自動啟動
sudo systemctl enable ssh --now

# 3. 確認狀態
sudo systemctl status ssh

# 4. 開防火牆（先開規則再啟用）
sudo ufw allow 22/tcp
sudo ufw enable

# 5. 查 IP
hostname -I

# 6. 客戶端產生金鑰並上傳
ssh-keygen -t ed25519
ssh-copy-id 使用者名稱@伺服器IP

# 7. 測試連線
ssh 使用者名稱@伺服器IP
```

---

<a id="ref"></a>

## 📚 參考資料

- [OpenSSH 官方](https://www.openssh.com/)
- [sshd_config 手冊](https://man.openbsd.org/sshd_config)
- [ssh_config 手冊](https://man.openbsd.org/ssh_config)
- [Ubuntu Server 說明：OpenSSH](https://ubuntu.com/server/docs/service-openssh)
