# Linux 常用指令整理

## 檔案與目錄操作

```bash
ls              # 列出檔案
ls -la          # 列出所有檔案(含隱藏檔)及詳細資訊
cd 路徑          # 切換目錄
cd ..           # 回上一層
pwd             # 顯示目前所在路徑
mkdir 名稱       # 建立資料夾
rm 檔名          # 刪除檔案
rm -rf 資料夾     # 強制刪除資料夾及內容(小心使用)
cp 來源 目的      # 複製檔案
mv 來源 目的      # 移動或重新命名
find . -name "檔名"  # 搜尋檔案
```

## 檢視與編輯檔案

```bash
cat 檔名         # 顯示檔案內容
less 檔名        # 分頁瀏覽檔案
head -n 20 檔名  # 顯示前20行
tail -n 20 檔名  # 顯示後20行
tail -f 檔名     # 即時追蹤檔案變化(常用於看 log)
nano 檔名        # 簡易文字編輯器
vim 檔名         # vim 編輯器
grep "字串" 檔名  # 搜尋檔案內容
```

## 權限與擁有者

```bash
chmod 755 檔名          # 修改權限
chmod +x 檔名           # 加上可執行權限
chown user:group 檔名   # 修改擁有者
```

## 系統與程序管理

```bash
ps aux          # 顯示所有執行中程序
top / htop      # 即時監控系統資源
kill PID        # 結束程序
kill -9 PID     # 強制結束程序
df -h           # 顯示磁碟使用量
du -sh 資料夾    # 顯示資料夾大小
free -h         # 顯示記憶體使用狀況
uptime          # 系統執行時間
```

## 網路相關

```bash
ping 網址        # 測試連線
curl 網址        # 抓取網頁內容/測試 API
wget 網址        # 下載檔案
ss -tulnp        # 查看開啟的連接埠(取代 netstat)
ifconfig / ip a  # 查看網路介面
```

## 套件管理(依發行版不同)

```bash
# Ubuntu/Debian
sudo apt update && sudo apt upgrade
sudo apt install 套件名

# CentOS/RHEL
sudo yum install 套件名
```

## 壓縮與解壓縮

```bash
tar -zcvf 檔名.tar.gz 資料夾   # 壓縮
tar -zxvf 檔名.tar.gz          # 解壓縮
zip -r 檔名.zip 資料夾
unzip 檔名.zip
```

## 其他常用

```bash
history         # 查看指令歷史
clear           # 清空畫面
sudo 指令        # 以管理員權限執行
man 指令名        # 查看指令說明
which 指令名      # 查看指令所在路徑
```
