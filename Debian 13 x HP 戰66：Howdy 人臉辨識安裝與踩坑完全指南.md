---
title: Debian 13 x HP 戰66：Howdy 人臉辨識安裝與踩坑完全指南
labels: [Linux, Debian, Howdy, 人臉辨識, HP, 技術筆記]
published: false
post_id: 8061740908218559780
---

# Debian 13 x HP 戰66：Howdy 人臉辨識安裝與踩坑完全指南

在建構與維護 HomeLab（PVE Node、Synology NAS、Mac Mini）的過程中，作為主要管理終端的 HP 戰 66 (Debian 13) 頻繁需要使用 `sudo` 權限。由於閉源指紋晶片 (Elan) 的 Linux 支援度極差，我們改採用 **Howdy** 搭配筆電內建的 HP HD Camera (`/dev/videoX`) 來實現無密碼的臉部解鎖，大幅提升維運效率。

> [!WARNING]
> **核心挑戰：**
> Debian 13 嚴格實施了 Python PEP 668 規範，加上 Howdy 2.6.1 安裝腳本過於老舊，會導致安裝過程中斷、缺少執行檔捷徑，以及 dlib 編譯失敗。本指南記錄了完整的強制通關 SOP。

---

## 階段一：前置準備與依賴補齊

### 1. 確認攝影機裝置路徑

首先安裝工具並列出所有影片裝置：

```bash
sudo apt update && sudo apt install v4l-utils
v4l2-ctl --list-devices
```

> [!NOTE]
> 請記下 HP HD Camera 的路徑，通常為 `/dev/video0` 或 `/dev/video2`。

### 2. 安裝系統層級的基礎編譯與視覺套件

預先補齊 `numpy`、`opencv` 與 C++ 編譯工具，減輕後續的依賴地獄：

```bash
sudo apt install python3-numpy python3-opencv cmake python3-dev build-essential
```

### 3. 下載 Howdy 安裝包

```bash
wget https://github.com/boltgolt/howdy/releases/download/v2.6.1/howdy_2.6.1.deb
```

---

## 階段二：外科手術級的強制安裝 (破解 PEP 668)

### 1. 執行初始安裝（預期會報錯中斷）

Debian 會因為套件腳本試圖升級全域 pip 而強行中斷安裝。此時套件會卡在 `half-configured` 狀態：

```bash
sudo apt install ./howdy_2.6.1.deb
```

### 2. 修改殘留的安裝腳本 (閹割報錯點)

直接進入系統底層修改 Howdy 的安裝後腳本：

```bash
sudo vi /var/lib/dpkg/info/howdy.postinst
```

**操作重點：** 搜尋 `Upgrading pip`，將呼叫 pip 升級的那一行註解掉：

```python
# subprocess.call([sys.executable, "-m", "pip", "install", "--upgrade", "pip"])
```

### 3. 接續未完成的安裝 (下載權重模型)

帶上突破 Python 限制的環境變數，讓系統跑完剩餘的設定。此步驟會從網路下載三個 `.dat` 機器學習模型檔：

```bash
sudo PIP_BREAK_SYSTEM_PACKAGES=1 dpkg --configure howdy
```

### 4. 手動編譯核心引擎 (dlib / face_recognition)

由於腳本被我們修改過，必須手動補上最重要的兩個機器學習函式庫。

> [!IMPORTANT]
> **注意：** 此步驟會讓 CPU 滿載編譯長達 5~15 分鐘，請耐心等待，絕對不要中斷！

```bash
sudo PIP_BREAK_SYSTEM_PACKAGES=1 pip3 install dlib face_recognition
```

---

## 階段三：修復幽靈指令與系統設定

### 1. 建立遺失的全域指令捷徑

Howdy 腳本忘記把執行檔放進 PATH，我們手動建立軟連結：

```bash
sudo chmod +x /lib/security/howdy/cli.py
sudo ln -s /lib/security/howdy/cli.py /usr/bin/howdy
```

### 2. 設定鏡頭路徑

開啟主設定檔，找到 `[video]` 區塊，將 `device_path` 改為階段一查到的路徑：

```bash
sudo EDITOR=vi howdy config
```

> [!TIP]
> 建議同時確認 `certainty = 3.5` 的安全閾值，修改後儲存離開。

### 3. 錄製與測試臉部模型

```bash
# 新增你的臉部特徵 (可多次執行以適應不同光源)
sudo howdy add

# 啟動即時辨識測試迴圈 (按 Ctrl+C 結束)
sudo howdy test
```

---

## 階段四：生死交關的 PAM 認證串接

最後一步是讓 `sudo` 認識 Howdy。

> [!CAUTION]
> **警告：** 修改時請保持目前終端機開啟，另開新視窗測試，以免改錯導致無法使用 `sudo` 救回系統。

1. **編輯 sudo 認證設定：**
   ```bash
   sudo vi /etc/pam.d/sudo
   ```

2. **注入 Howdy 規則：**
   在檔案的**最上方**（`@include common-auth` 的前面），加入以下這一行：
   ```text
   auth sufficient pam_howdy.so
   ```

3. **終極驗證：**
   在**另一個全新的終端機視窗**中測試：
   ```bash
   sudo -k
   sudo ls /root
   ```

**如果鏡頭燈亮起，且無須密碼即印出內容，恭喜你，臉部解鎖系統正式上線！**

---
*文件產生日期：2026-03-02*