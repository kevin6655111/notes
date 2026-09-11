# C++ 環境配置

> 在 Ubuntu（含 WSL）與 Windows 上建立 VS Code 的 C++ 開發環境。

## 目錄

- [🐧 Ubuntu / WSL](#ubuntu)
- [🪟 Windows](#windows)
- [🚨 常見錯誤](#error)

---

<a id="ubuntu"></a>

## 🐧 Ubuntu / WSL

1. [官方設定教學](https://code.visualstudio.com/docs/cpp/config-linux)
2. VS Code 要安裝 **WSL** 擴充套件（Remote Development），才能在 WSL 環境裡開專案。

### 安裝編譯工具

```bash
sudo apt-get update
sudo apt-get install gcc      # C 編譯器
sudo apt-get install g++      # C++ 編譯器
sudo apt-get install gdb      # 除錯器
sudo apt-get install clang    # 另一套編譯器，與 gcc 功能相同，可擇一或並存
```

一行裝齊（`build-essential` 已包含 gcc、g++、make）：

```bash
sudo apt-get update && sudo apt-get install -y build-essential gdb
```

### 確認版本

```bash
gcc --version
g++ --version
gdb --version
```

---

<a id="windows"></a>

## 🪟 Windows

### 1. 下載 [MinGW](https://nuwen.net/mingw.html)

### 2. [安裝 VS Code 並設定 C++](https://hackmd.io/@liaojason2/vscodecppwindows)

### 3. 設定支援 C++17

- `c_cpp_properties.json`

  ![c_cpp_properties.json 設定](https://raw.githubusercontent.com/kevin6655111/notes/main/images/cpp-c_cpp_properties.png)
  <!-- 原圖：https://hackmd.io/_uploads/SkHxnT3Z6.png -->

- `tasks.json`

  ![tasks.json 設定](https://raw.githubusercontent.com/kevin6655111/notes/main/images/cpp-tasks.png)
  <!-- 原圖：https://hackmd.io/_uploads/S1j-m0nbT.png -->

---

<a id="error"></a>

## 🚨 常見錯誤

### 1. include 標頭檔出現紅色波浪符

- 可以正常編譯，但編輯器顯示紅色波浪符，通常是 IntelliSense 找不到 include 路徑。
- [解決辦法](https://blog.csdn.net/m0_38055352/article/details/105375367)

### 2. 直接關閉錯誤提示

- 若只是提示誤報，可以關閉 IntelliSense 的錯誤波浪符。
- [解決辦法](https://blog.csdn.net/pingxiaozhao/article/details/124251063)
