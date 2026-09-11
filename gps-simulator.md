# GPS Simulator

> 模擬 GPS 裝置透過 Serial Port 輸出 NMEA 資料，程式接收後解析出經緯度並顯示在地圖上。

## 目錄

- [📚 參考資料](#ref)
- [🪟 Windows：GPSinfo](#gpsinfo)
- [🔁 流程](#flow)
- [🐧 Linux 安裝模組](#linux-modules)
- [⌨️ Linux 常用指令](#linux-cmd)
- [🚨 常見問題](#trouble)

---

<a id="ref"></a>

## 📚 參考資料

- [NMEA 0183 格式說明 - NCU msplab](https://hackmd.io/@NCUmsplab/Bk6BsFP3D)
- [GPS NMEA 解析 - CSDN](https://blog.csdn.net/qq_32478489/article/details/107149673)
- [架設 OpenStreetMap Tile Server（Ubuntu 20.04）](https://www.linuxbabe.com/ubuntu/openstreetmap-tile-server-ubuntu-20-04-osm)

---

<a id="gpsinfo"></a>

## 🪟 Windows：GPSinfo

1. [下載 Virtual Serial Port Driver](https://freevirtualserialports.com/)
2. 用 **Virtual Serial Port Driver Pro** 建立一對虛擬 Serial Port（例如 COM1 ↔ COM2），模擬實體裝置。
3. 執行 `main.py`

![](https://hackmd.io/_uploads/BJ6vluCW6.png)

![](https://hackmd.io/_uploads/ByHPrIyGT.png)

---

<a id="flow"></a>

## 🔁 流程

**手動輸入**

```
輸入經緯度 → 顯示地圖並標記位置
```

**Serial Port 模擬**

```
COM 輸入端送出 NMEA 字串 → COM 輸出端接收 → 程式解析出經緯度 → 顯示地圖並標記位置
```

---

<a id="linux-modules"></a>

## 🐧 Linux 安裝模組

### 1. socat：建立虛擬 Serial Port

一對 pty 互相連通，寫入一端就能從另一端讀到，用來取代 Windows 的 Virtual Serial Port Driver。

```bash
# 建立一對互通的虛擬 port，並建立符號連結方便存取
sudo socat -d -d pty,link=/dev/ttyV0,raw,echo=0 pty,link=/dev/ttyV1,raw,echo=0

# 監聽其中一端
cat /dev/ttyV1

# 從另一端寫入測試
echo "\$GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M,46.9,M,,*47" > /dev/ttyV0

# socat 實際建立的是 /dev/pts/N，權限不足時開放讀寫
sudo chmod a+rw /dev/pts/9
```

> - `-d -d` 顯示除錯訊息，會印出實際配置到的 `/dev/pts/N` 編號。
> - 原筆記用 `/dev/ttyS2`、`/dev/ttyS3` 當連結名稱，但 `ttyS*` 是系統保留給實體 UART 的名稱，可能與真實裝置衝突，建議改用 `ttyV0`、`ttyV1` 之類的自訂名稱。
> - `raw,echo=0`：關閉終端機的行緩衝與回顯，讓資料原封不動地通過。

### 2. minicom：Serial Port 終端機

- 用於 Serial Port 通訊的終端模擬程式，可在 Linux 與 Unix 系統使用。
- 提供終端使用者介面，透過 Serial Port 與其他設備或系統通訊，例如嵌入式裝置、路由器、交換器。

```bash
sudo apt install minicom
sudo minicom -s                  # 進入設定畫面（選 port、baud rate）
sudo minicom -D /dev/ttyV1 -b 9600  # 直接指定裝置與 baud rate
```

> 離開 minicom：`Ctrl+A` 再按 `X`。

### 3. libcurl：HTTP 連線

```bash
sudo apt install libcurl4-openssl-dev
```

### 4. ImageMagick：顯示圖片

```bash
sudo apt-get install imagemagick
display -geometry 1000x600 ~/Downloads/bird.jpg   # 指定視窗大小
display -resize 600x400 ~/Downloads/bird.jpg      # 縮放圖片後顯示
```

### 5. OpenCV：影像處理

```bash
sudo apt install libopencv-dev
pkg-config --modversion opencv4   # 確認版本
```

### 6. nlohmann/json：解析 JSON

```bash
sudo apt install nlohmann-json3-dev
```

```cpp
#include <nlohmann/json.hpp>
using json = nlohmann::json;
```

### 7. Qt5：寫介面

```bash
sudo apt-get install qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools qtcreator
```

### 8. 中文語言包與字型

```bash
sudo apt-get install language-pack-zh-hant   # 繁體中文（原筆記為 zh-hans 簡體，依需求擇一）
sudo apt-get install fonts-droid-fallback fonts-wqy-zenhei fonts-wqy-microhei fonts-arphic-ukai fonts-arphic-uming
```

---

<a id="linux-cmd"></a>

## ⌨️ Linux 常用指令

```bash
# 列出 serial port 資訊（含 USB 轉 serial 的裝置偵測紀錄）
dmesg | grep tty

# 列出目前存在的 serial 裝置
ls -l /dev/ttyS* /dev/ttyUSB* /dev/ttyACM* 2>/dev/null

# 查看哪個程式正在佔用 serial port
sudo lsof /dev/ttyS0

# 直接讀取 serial port 原始輸出
cat /dev/ttyUSB0

# 設定 baud rate 後讀取
stty -F /dev/ttyUSB0 9600 raw
cat /dev/ttyUSB0
```

---

<a id="trouble"></a>

## 🚨 常見問題

### VS Code 開不了資料夾（Permission denied）

系統目錄的擁有者是 root，一般使用者沒有寫入權限。把目錄擁有者改成自己即可：

```bash
sudo chown -R kevin:kevin /usr/local/include/opencv4/opencv2/
```

> - `-R` 遞迴套用到所有子目錄與檔案。
> - `kevin:kevin` 是「使用者:群組」，換成自己的帳號。
> - 只對需要編輯的目錄做，不要對整個 `/usr` 或 `/usr/local` 執行，會破壞系統套件的權限。

### Serial Port 沒有權限（/dev/ttyUSB0: Permission denied）

```bash
sudo usermod -aG dialout $USER   # 把自己加入 dialout 群組，登出再登入生效
```

比每次 `chmod a+rw` 好，重開機後依然有效。
