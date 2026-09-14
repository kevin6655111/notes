# Windows 設定筆記

## 目錄

- [🛑 關閉 Windows 自動更新](#disable-update)
  - [方法 1：登錄編輯程式（Home / Pro 皆可）](#regedit)
  - [方法 2：群組原則（Pro / Enterprise）](#gpedit)
  - [方法 3：停用 Windows Update 服務](#service)
  - [方法 4：Windows 設定 暫停更新](#pause)
  - [方法 5：計量付費連線](#metered)
  - [方法 6：PowerShell 一鍵設定](#powershell)
  - [還原自動更新](#restore)
  - [各方法比較](#compare)

---

<a id="disable-update"></a>

## 🛑 關閉 Windows 自動更新

> 關閉自動更新會讓系統少掉安全性修補，只建議用在展示機、測試機或無法隨意重開的設備，並定期手動更新。

<a id="regedit"></a>

### 方法 1：登錄編輯程式（Home / Pro 皆可）

等同於方法 2 的群組原則，Home 版沒有 gpedit 時用這個。

1. `Win + R` 輸入 `regedit`，開啟登錄編輯程式。
2. 前往（路徑不存在就逐層新增機碼）：

   ```text
   HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU
   ```

3. 右側空白處右鍵 → 新增 → DWORD (32 位元) 值，建立以下兩個值：

   | 名稱 | 值 | 說明 |
   |------|:--:|------|
   | `NoAutoUpdate` | `1` | 完全關閉自動更新 |
   | `AUOptions` | `1` | 不檢查更新（僅在 NoAutoUpdate = 0 時才會參考此值） |

   `AUOptions` 的其他選項，不想完全關閉時可改用：

   | 值 | 行為 |
   |:--:|------|
   | `2` | 有更新時通知，下載與安裝都要手動 |
   | `3` | 自動下載，安裝前通知 |
   | `4` | 自動下載並依排程安裝 |
   | `5` | 允許本機管理員自行選擇 |

4. 重新開機。

以 `.reg` 檔一次匯入：

```reg
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU]
"NoAutoUpdate"=dword:00000001
"AUOptions"=dword:00000001
```

<a id="gpedit"></a>

### 方法 2：群組原則（Pro / Enterprise）

1. `Win + R` 輸入 `gpedit.msc`。
2. 前往：

   ```text
   電腦設定 → 系統管理範本 → Windows 元件 → Windows Update → 管理使用者體驗
   ```

   （Windows 10 舊版路徑：`... → Windows Update → 設定自動更新`）

3. 開啟「設定自動更新」，選「已停用」→ 確定。
   若只想改成通知不自動裝，選「已啟用」並在下方選 `2 - 通知下載和自動安裝`。
4. 命令提示字元執行 `gpupdate /force`，或重新開機。

實際上就是寫入方法 1 的登錄機碼。

<a id="service"></a>

### 方法 3：停用 Windows Update 服務

最直接，但 Windows 10/11 有時會由「Windows Update Medic Service」自動把服務改回啟用，需一併處理。

1. `Win + R` 輸入 `services.msc`。
2. 找到 **Windows Update**（服務名稱 `wuauserv`），雙擊。
3. 啟動類型改為「已停用」，並按「停止」。
4. 「登入」分頁可改用不存在的帳號登入（如 `Guest`、亂打密碼），使服務即使被改回也啟動失敗。

命令列版本：

```bat
sc stop wuauserv
sc config wuauserv start= disabled
```

<a id="pause"></a>

### 方法 4：Windows 設定 暫停更新

不需管理權限，但只能暫停最多 5 週（Windows 11 為 1 至 5 週），到期後會自動恢復。

```text
設定 → Windows Update → 暫停更新 → 選擇週數
```

延長暫停天數（最多 35 天）可改登錄：

```text
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\WindowsUpdate\UX\Settings
FlightSettingsMaxPauseDays (DWORD) = 想要的天數
```

<a id="metered"></a>

### 方法 5：計量付費連線

把目前網路設為計量付費，Windows 預設不會透過計量連線下載更新。適合筆電、臨時場合，但部分「優先更新」仍會下載。

```text
設定 → 網路和網際網路 → Wi-Fi / 乙太網路 → 目前的網路 → 計量付費連線：開啟
```

<a id="powershell"></a>

### 方法 6：PowerShell 一鍵設定

以系統管理員身分執行，等同方法 1 + 方法 3：

```powershell
# 登錄：關閉自動更新
New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Force | Out-Null
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name NoAutoUpdate -Type DWord -Value 1
Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" -Name AUOptions   -Type DWord -Value 1

# 服務：停止並停用
Stop-Service wuauserv -Force
Set-Service  wuauserv -StartupType Disabled

# 驗證
Get-Service wuauserv | Select-Object Status, StartType
Get-ItemProperty "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU"
```

<a id="restore"></a>

### 還原自動更新

```powershell
Remove-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate" -Recurse -Force
Set-Service  wuauserv -StartupType Manual
Start-Service wuauserv
```

群組原則則把「設定自動更新」改回「尚未設定」。

<a id="compare"></a>

### 各方法比較

| 方法 | 需管理員 | Home 版 | 永久 | 被系統自動還原 | 備註 |
|------|:--:|:--:|:--:|:--:|------|
| 1 登錄編輯程式 | ✅ | ✅ | ✅ | 少見 | 最常用，建議搭配方法 3 |
| 2 群組原則 | ✅ | ❌ | ✅ | 少見 | 與方法 1 效果相同，介面較友善 |
| 3 停用服務 | ✅ | ✅ | ✅ | 常見 | 可能被 Medic Service 改回 |
| 4 暫停更新 | ❌ | ✅ | ❌ | 到期恢復 | 最多 5 週，最安全 |
| 5 計量付費連線 | ❌ | ✅ | ✅ | ❌ | 只擋下載，不擋檢查 |
| 6 PowerShell | ✅ | ✅ | ✅ | 少見 | 方法 1 + 3 的腳本版 |

---

## 📚 參考資料

- [Microsoft Learn：Configure Automatic Updates by using Group Policy / Registry](https://learn.microsoft.com/windows/deployment/update/waas-wu-settings)
- [Microsoft Learn：Windows Update 登錄機碼一覽](https://learn.microsoft.com/windows/deployment/update/waas-wu-settings#registry-keys-used-to-manage-restart)
