# Windows 安裝 OpenSSH Server

> 讓其他電腦能透過 SSH 連進這台 Windows。適用 Windows 10 / 11、Windows Server 2019 以後。
>
> Linux 主機的做法見 [Linux 安裝 OpenSSH Server](ssh-linux.md)。

## 目錄

- [🔍 確認元件狀態](#check)
- [📥 安裝 OpenSSH Server](#install)
- [▶️ 啟動服務](#service)
- [🔥 開放防火牆 22 port](#firewall)
- [🐚 設定預設 Shell](#shell)
- [🔑 設定金鑰登入](#key)
- [🌐 查詢 IP](#ip)
- [🖥️ 從另一台電腦測試連線](#connect)
- [⚙️ 修改設定](#config)
- [🚨 常見問題排查](#trouble)
- [📋 快速指令總覽](#quick)
- [📚 參考資料](#ref)

> 以下指令都要用**系統管理員身分**開啟 PowerShell。開始選單搜尋 PowerShell，右鍵選「以系統管理員身分執行」。

---

<a id="check"></a>

## 🔍 確認元件狀態

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'
```

會看到兩個項目：

```text
Name  : OpenSSH.Client~~~~0.0.1.0
State : Installed

Name  : OpenSSH.Server~~~~0.0.1.0
State : NotPresent
```

| 元件 | 作用 |
|------|------|
| `OpenSSH.Client` | 這台電腦可以**連出去**，Windows 10 1809 以後預設就有 |
| `OpenSSH.Server` | 這台電腦可以**被連進來**，預設沒有安裝 |

---

<a id="install"></a>

## 📥 安裝 OpenSSH Server

`OpenSSH.Server` 顯示 `NotPresent` 時執行：

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

裝完再跑一次上一步的指令，確認變成 `Installed`。

> 沒有網際網路連線的機器會失敗，因為這是從 Windows Update 取得的隨選功能。離線環境要改用 FoD ISO 或從 [PowerShell/Win32-OpenSSH](https://github.com/PowerShell/Win32-OpenSSH/releases) 下載安裝。

---

<a id="service"></a>

## ▶️ 啟動服務

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'
Get-Service sshd
```

正常顯示：

```text
Status   Name    DisplayName
------   ----    -----------
Running  sshd    OpenSSH SSH Server
```

首次啟動會在 `C:\ProgramData\ssh\` 產生主機金鑰與預設的 `sshd_config`。

---

<a id="firewall"></a>

## 🔥 開放防火牆 22 port

安裝時通常會自動建立規則，先確認有沒有：

```powershell
Get-NetFirewallRule -Name *ssh*
```

沒有的話手動建立：

```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' `
  -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22
```

**雲端主機還有第二層防火牆。** Azure VM、AWS EC2 等還要到後台的網路安全群組開放 inbound 22 port，本機規則設得再對也連不進來。

---

<a id="shell"></a>

## 🐚 設定預設 Shell

連進來預設是 `cmd.exe`。改成 PowerShell：

```powershell
# Windows PowerShell 5.1（系統內建）
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell `
  -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force

# PowerShell 7（需另外安裝）
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell `
  -Value "C:\Program Files\PowerShell\7\pwsh.exe" -PropertyType String -Force
```

> ⚠️ **改了 DefaultShell 之後 `scp` 與 `sftp` 可能失效。** 那兩個協定預期拿到乾淨的輸出，PowerShell 的啟動訊息與提示字元會干擾。有傳檔需求就維持 `cmd.exe`，或在 `sshd_config` 用 `Subsystem sftp` 指定內建的 sftp-server。

---

<a id="key"></a>

## 🔑 設定金鑰登入

Windows 這裡有一個**最常見的踩雷點**：一般帳號與系統管理員帳號的公鑰放在不同位置。

| 帳號類型 | 公鑰放哪 |
|----------|----------|
| 一般使用者 | `C:\Users\<帳號>\.ssh\authorized_keys` |
| Administrators 群組成員 | `C:\ProgramData\ssh\administrators_authorized_keys` |

這個行為來自 `sshd_config` 結尾的這段：

```text
Match Group administrators
       AuthorizedKeysFile __PROGRAMDATA__/ssh/administrators_authorized_keys
```

把這兩行註解掉，管理員帳號就會改用各自家目錄下的 `authorized_keys`。

### 權限必須收緊

`administrators_authorized_keys` 只能讓 `SYSTEM` 與 `Administrators` 存取，否則 SSH 會直接忽略這個檔案而退回問密碼，且不會說原因：

```powershell
icacls.exe "C:\ProgramData\ssh\administrators_authorized_keys" /inheritance:r
icacls.exe "C:\ProgramData\ssh\administrators_authorized_keys" /grant "Administrators:F" "SYSTEM:F"
```

### 從客戶端上傳公鑰

Windows 沒有 `ssh-copy-id`，用這行代替：

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh 帳號@IP "powershell -c \"Add-Content C:\ProgramData\ssh\administrators_authorized_keys\""
```

金鑰的產生方式見 [Linux 安裝 OpenSSH Server 的金鑰章節](ssh-linux.md#key)，客戶端的做法兩邊相同。

---

<a id="ip"></a>

## 🌐 查詢 IP

```powershell
ipconfig                                    # 找 IPv4 位址
(Get-NetIPAddress -AddressFamily IPv4).IPAddress    # 只列 IP
```

例如 `192.168.1.50` 是區網位址。雲端主機用後台顯示的公網 IP。

---

<a id="connect"></a>

## 🖥️ 從另一台電腦測試連線

```shell
ssh 使用者帳號@IP位址
ssh User@192.168.1.50
```

- 第一次連線會問是否信任主機金鑰指紋，輸入 `yes`
- 接著輸入該 **Windows 帳號的登入密碼**
- 成功後會看到 Windows 的命令提示字元或 PowerShell 提示字元

網域帳號要寫成 `網域\帳號` 的形式，在多數 shell 裡反斜線需要跳脫：

```shell
ssh "DOMAIN\User@192.168.1.50"
```

---

<a id="config"></a>

## ⚙️ 修改設定

設定檔在 `C:\ProgramData\ssh\sshd_config`，語法與 Linux 相同。

改完先驗證再重啟：

```powershell
& 'C:\Windows\System32\OpenSSH\sshd.exe' -t     # 沒有輸出就是沒問題
Restart-Service sshd
```

改 port 時記得一併調整防火牆規則：

```powershell
Set-NetFirewallRule -Name sshd -LocalPort 2222
```

> ⚠️ **改設定時保留一個已登入的 session。** 重啟服務不會斷開既有連線，另開一個視窗確認連得進來再關舊的。

---

<a id="trouble"></a>

## 🚨 常見問題排查

| 狀況 | 檢查方式 |
|------|----------|
| `Connection refused` | `Get-Service sshd` 確認是 Running；`netstat -ano \| findstr :22` 確認有在監聽 |
| `Connection timed out` | 本機防火牆規則或雲端安全群組沒開 |
| 金鑰登入失敗退回問密碼 | 管理員帳號的公鑰是否放在 `administrators_authorized_keys`，以及 `icacls` 權限是否收緊 |
| 改了設定沒生效 | 沒執行 `Restart-Service sshd` |
| `scp` / `sftp` 傳檔失敗 | `DefaultShell` 被改成 PowerShell，改回 `cmd.exe` 試試 |
| 22 port 被佔用 | 這台同時跑 WSL 且 WSL 內也啟了 sshd。把其中一邊改到別的 port |

看詳細日誌：

```powershell
Get-WinEvent -LogName 'OpenSSH/Operational' -MaxEvents 30
```

或在 `sshd_config` 開啟檔案日誌後重啟服務，日誌會寫到 `C:\ProgramData\ssh\logs\sshd.log`：

```text
SyslogFacility LOCAL0
LogLevel DEBUG3
```

---

<a id="quick"></a>

## 📋 快速指令總覽

系統管理員 PowerShell，可直接複製貼上：

```powershell
# 1. 檢查狀態
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'

# 2. 安裝 Server
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# 3. 啟動並設定開機自動啟動
Start-Service sshd
Set-Service -Name sshd -StartupType 'Automatic'

# 4. 開防火牆
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' `
  -Enabled True -Direction Inbound -Protocol TCP -Action Allow -LocalPort 22

# 5. 確認服務狀態
Get-Service sshd

# 6. 查 IP
ipconfig
```

---

<a id="ref"></a>

## 📚 參考資料

- [Microsoft Learn：OpenSSH for Windows](https://learn.microsoft.com/windows-server/administration/openssh/openssh_install_firstuse)
- [OpenSSH Server 設定](https://learn.microsoft.com/windows-server/administration/openssh/openssh_server_configuration)
- [金鑰管理](https://learn.microsoft.com/windows-server/administration/openssh/openssh_keymanagement)
- [PowerShell/Win32-OpenSSH](https://github.com/PowerShell/Win32-OpenSSH)
