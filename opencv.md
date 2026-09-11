# OpenCV

> WSL 沒有自己的螢幕，OpenCV 的 `imshow` 或 Qt 視窗要顯示出來，得把畫面透過 X11 送到 Windows 上的 X Server（VcXsrv）。

## 目錄

- [📚 參考資料](#ref)
- [📦 安裝 OpenCV](#install)
- [🪟 Windows 端：啟用 WSL](#windows-features)
- [🖥️ Windows 端：VcXsrv](#vcxsrv)
- [🐧 WSL 端：Qt / X11 環境變數](#env)
- [🖱️ 選用：跑整個 XFCE 桌面](#xfce)
- [🚨 常見問題](#trouble)

---

<a id="ref"></a>

## 📚 參考資料

- [WSL 使用 VcXsrv 顯示 GUI 教學 - JYU](https://hackmd.io/@JYU/B1zmv1MCU)（本篇截圖來源）
- [OpenCV 官方文件](https://docs.opencv.org/4.x/)
- 相關筆記：[GPS Simulator - Linux 安裝模組](gps-simulator.md#linux-modules)（OpenCV、Qt5 安裝指令）

---

<a id="install"></a>

## 📦 安裝 OpenCV

```bash
sudo apt update
sudo apt install libopencv-dev          # 含標頭檔與函式庫，C++ 開發用
pkg-config --modversion opencv4         # 確認版本
pkg-config --cflags --libs opencv4      # 編譯時需要的參數
```

編譯範例：

```bash
g++ main.cpp -o main $(pkg-config --cflags --libs opencv4)
```

---

<a id="windows-features"></a>

## 🪟 Windows 端：啟用 WSL

控制台 → 程式集 → 開啟或關閉 Windows 功能，勾選：

- **Windows 子系統 Linux 版**
- **虛擬機器平台**（WSL2 需要）

![Windows 功能中勾選 WSL 與虛擬機器平台](https://raw.githubusercontent.com/kevin6655111/notes/main/images/windows-features-wsl.png)

---

<a id="vcxsrv"></a>

## 🖥️ Windows 端：VcXsrv

VcXsrv 是跑在 Windows 上的 X Server，負責接收 WSL 送來的畫面並開視窗。

> **Windows 11 可以不用裝**。WSL2 內建 WSLg，直接支援 GUI，不需要 VcXsrv 也不用設 `DISPLAY`。先在 WSL 跑 `echo $DISPLAY`，若已經有值（例如 `:0`）就表示 WSLg 可用，以下步驤可以略過。

### 1. 安裝

下載安裝 [VcXsrv](https://sourceforge.net/projects/vcxsrv/)，安裝完執行 **XLaunch**。

### 2. XLaunch 設定精靈

**Step 1：Display settings**

- 只要跑單一程式視窗（例如 OpenCV `imshow`、Qt 程式）：選 **Multiple windows**，每個 Linux 視窗會變成獨立的 Windows 視窗。
- 想跑整個 Linux 桌面環境：選 **One large window**（教學截圖選的是這個）。
- **Display number** 填 `0`，對應之後 `DISPLAY` 環境變數結尾的 `:0`。

![XLaunch Display settings](https://raw.githubusercontent.com/kevin6655111/notes/main/images/vcxsrv-xlaunch-1-display.png)

**Step 2：Client startup**

選 **Start no client**，只啟動 X Server，程式之後由 WSL 端啟動。

![XLaunch Client startup](https://raw.githubusercontent.com/kevin6655111/notes/main/images/vcxsrv-xlaunch-2-client.png)

**Step 3：Extra settings**

- **Clipboard**：勾選，讓 WSL 與 Windows 共用剪貼簿。
- **Native opengl**：勾選的話 WSL 端要設 `LIBGL_ALWAYS_INDIRECT=1`（見下一節）。
- **Disable access control**：**一定要勾**，否則 WSL 連不上，會出現 `could not connect to display`。勾選後 Additional parameters 會自動帶入 `-ac`。

![XLaunch Extra settings](https://raw.githubusercontent.com/kevin6655111/notes/main/images/vcxsrv-xlaunch-3-extra.png)

**Step 4：Finish**

按 **Save configuration** 存成 `config.xlaunch`，之後直接雙擊這個檔案就能用同樣設定啟動，不用每次重跑精靈。放到「啟動」資料夾（`shell:startup`）可開機自動執行。

![XLaunch Finish configuration](https://raw.githubusercontent.com/kevin6655111/notes/main/images/vcxsrv-xlaunch-4-finish.png)

### 3. 啟動後的畫面

選 One large window 模式時會出現一個全黑視窗，這是正常的，代表 X Server 在等待連線。Multiple windows 模式則不會有視窗，只在系統列出現圖示。

![VcXsrv 啟動後的空白視窗](https://raw.githubusercontent.com/kevin6655111/notes/main/images/vcxsrv-empty-window.png)

### 4. 防火牆

WSL2 走的是 Hyper-V 虛擬網卡，Windows 防火牆會把它當成外部連線。第一次啟動 VcXsrv 若跳出詢問，**私人與公用網路都要允許**。若當時按了取消，到「控制台 → Windows Defender 防火牆 → 允許應用程式通過防火牆」找 **VcXsrv windows xserver**，兩欄都勾起來。

![防火牆允許 VcXsrv](https://raw.githubusercontent.com/kevin6655111/notes/main/images/vcxsrv-firewall.png)

---

<a id="env"></a>

## 🐧 WSL 端：Qt / X11 環境變數

把以下設定加進 shell 設定檔，讓每次開終端機都自動生效。

```bash
vim ~/.zshrc                              # 用 bash 的話是 ~/.bashrc 或 ~/.bash_login
```

```bash
# Qt 使用 X11 後端
export QT_QPA_PLATFORM=xcb

# DISPLAY 指向 Windows 主機的 X Server，兩種寫法擇一
export DISPLAY=192.168.182.78:0                                              # 寫法 A：固定 IP
export DISPLAY=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}'):0  # 寫法 B：自動抓（建議）

# XLaunch 有勾 Native opengl 時必須加，否則 OpenGL 程式會黑畫面或崩潰
export LIBGL_ALWAYS_INDIRECT=1

# 除錯用，找不到 Qt 平台外掛時打開，正常使用請關掉或註解
# export QT_DEBUG_PLUGINS=1
```

```bash
source ~/.zshrc                           # 重新載入設定
```

教學裡是寫在 `~/.bash_login`，內容只有 `DISPLAY` 與 `LIBGL_ALWAYS_INDIRECT` 兩行：

![.bash_login 中的 DISPLAY 與 LIBGL_ALWAYS_INDIRECT 設定](https://raw.githubusercontent.com/kevin6655111/notes/main/images/wsl-bash-login-display.png)

> **兩種 DISPLAY 寫法的差別**
> - WSL2 的 Windows 主機 IP 每次重開機都可能變，寫法 A 固定 IP 會失效，要一直手動改。
> - 寫法 B 從 `/etc/resolv.conf` 抓 nameserver，WSL2 預設會把它設成 Windows 主機的 IP，所以能自動跟上。
> - 兩行都留著的話後面那行會覆蓋前面，實際上只有寫法 B 生效。原筆記兩行都寫，這裡保留但註明擇一。
> - WSL1 的話主機就是 `localhost`，直接寫 `export DISPLAY=:0` 即可。

### 確認 IP 是否正確

`DISPLAY` 裡的 IP 應該等於 Windows 端 **vEthernet (WSL)** 網卡的 IPv4。可以在工作管理員 → 效能 → 乙太網路 vEthernet (WSL) 看到，也可以在 WSL 裡直接印出來比對：

```bash
cat /etc/resolv.conf | grep nameserver    # WSL 看到的主機 IP
echo $DISPLAY                             # 目前設定的 DISPLAY
```

![工作管理員中 vEthernet (WSL) 的 IPv4 位址](https://raw.githubusercontent.com/kevin6655111/notes/main/images/wsl-vethernet-ip.png)

### 測試

```bash
sudo apt install x11-apps
xeyes                                     # 出現一對眼睛就代表 X11 通了
xclock
```

---

<a id="xfce"></a>

## 🖱️ 選用：跑整個 XFCE 桌面

只跑 OpenCV 或 Qt 單一視窗不需要這步。若想在 VcXsrv 的 One large window 裡看到完整 Linux 桌面：

```bash
sudo apt install xfce4 xfce4-terminal     # 安裝 XFCE 桌面環境（約 1 GB）
startxfce4                                # 啟動桌面，畫面會出現在 VcXsrv 視窗裡
```

終端機會刷出大量 WARNING 與 CRITICAL，多半是 XFCE 在 WSL 裡找不到電源管理、GL 加速等硬體服務，不影響使用，可忽略。

![startxfce4 的終端機輸出](https://raw.githubusercontent.com/kevin6655111/notes/main/images/wsl-startxfce4.png)

![VcXsrv 視窗中顯示的 XFCE 桌面](https://raw.githubusercontent.com/kevin6655111/notes/main/images/vcxsrv-xfce-desktop.png)

---

<a id="trouble"></a>

## 🚨 常見問題

### `qt.qpa.xcb: could not connect to display`

- VcXsrv 沒開，或 XLaunch 沒勾 Disable access control。
- `DISPLAY` 的 IP 不對，用 `cat /etc/resolv.conf` 確認 nameserver 是否等於 `DISPLAY` 裡的 IP，並與工作管理員的 vEthernet (WSL) IPv4 比對。
- Windows 防火牆擋住 VcXsrv，到「允許應用程式通過防火牆」把 VcXsrv 的私人與公用網路都勾起來。

### `This application failed to start because no Qt platform plugin could be initialized`

- 打開 `export QT_DEBUG_PLUGINS=1` 再執行一次，會印出到底缺哪個外掛或哪個 `.so`。
- 常見缺件：

```bash
sudo apt install libxcb-xinerama0 libxcb-cursor0 libxkbcommon-x11-0
```

### OpenGL 程式黑畫面或 `Unsupported GL renderer`

- XLaunch 勾了 Native opengl 但 WSL 端沒設 `LIBGL_ALWAYS_INDIRECT=1`。
- 或反過來把 Native opengl 取消勾選，改用軟體渲染，會慢但穩定。

### OpenCV `imshow` 沒反應或直接結束

- `imshow` 後面要接 `waitKey()`，否則視窗還沒畫出來程式就結束了。
- 若 OpenCV 是用無 GUI 的版本編譯（headless），`imshow` 會直接報錯，改裝 `libopencv-dev` 或自行編譯時開 `WITH_QT` / `WITH_GTK`。

### 畫面很慢或字型很醜

- X11 轉送本來就慢，大量影像顯示會卡，考慮改存檔用 Windows 端看，或升級 Windows 11 用 WSLg。
- 字型醜是 WSL 缺字型，裝 `fonts-noto-cjk` 或參考 [GPS Simulator 的中文字型安裝](gps-simulator.md#linux-modules)。
