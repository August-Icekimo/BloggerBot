---
title: 在 Linux Mint 22.3 上切換至 MODA ODF 工具的技術實務與經驗
labels: [Linux, ODF, MODA, LibreOffice, Mint, 技術筆記]
published: false
post_id: 5423106593453830774
---

# 移除 LibreOffice 並安裝 MODA ODF 文件應用工具：經驗總結

在追求數位主權與開放文件格式 (ODF) 的過程中，將辦公軟體切換至數位發展部 (MODA) 提供的 ODF 文件應用工具是一項重要的技術實作。本篇筆記記錄了在 Linux 環境下移除舊有 LibreOffice 並精確安裝 MODA 工具的完整流程與遇到的坑。

---

## 原始需求 (Objective)

這是一個典型的辦公環境遷移任務：

{{sms-thread-start}}
{{sms-right: 1. 移除目前安裝的 LibreOffice。 | name=Icekimo | time=10:00}}
{{sms-right: 2. 依據官方 QA 說明，安裝新版數位發展部 (moda.gov.tw) 的 Office 工具。 | name=User | time=10:01}}
{{sms-left: 沒問題，我會先進行環境評估與現有套件的深度移除。 | name=Gemini-CLI | time=10:05}}
{{sms-thread-end}}

---

## 執行流程與步驟

### 1. 環境調查 (Research)

> [!NOTE]
> **測試環境：** Linux Mint 22.3 (基於 Ubuntu 24.04 Noble)
> 官方文件主要針對 Ubuntu 22.04 提供支援，但在 Noble 核心上同樣能穩定運作。

### 2. 移除現有的 LibreOffice (Deep Cleanup)

為了避免軟體衝突，必須徹底移除舊套件：

```bash
sudo apt-get purge -y "libreoffice*"
sudo apt-get autoremove
```

### 3. 配置 MODA 官方軟體源

建立金鑰目錄並下載 GPG 以確保安全性：

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://free.nchc.org.tw/odfrepo/desktop/deb/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/moda.gpg
```

### 4. 安裝 MODA ODF 工具 (Installation Challenges)

> [!IMPORTANT]
> **遇到挑戰：**
> 嘗試使用 `modaodfapplicationtools*` 萬用字元安裝時，會因為部分擴充套件（如 `cpmlibre`）的版本號（4.0.3-2）與核心套件不匹配，導致嚴重的依賴衝突。
> 
> **解決方案：**
> 參考官方手冊列出的**特定套件清單**進行精確安裝，避開那些不相容的舊版擴充套件。

### 5. 驗證與最後清理

在安裝完成後，務必檢查是否有殘留：

> [!TIP]
> LibreOffice 的部分底層執行環境（如 `ure` 與 `uno-libs-private`）在移除主程式後可能依然存在。手動執行二次清理有助於提升 MODA 工具的穩定性。

最後確認 `/usr/share/applications/` 目錄下已出現啟動項目，即可開始使用。

---

## 關鍵經驗總結 (Lessons Learned)

1. **精確安裝優於萬用字元：** 官方軟體源中可能混雜不同時期的套件，優先選擇基礎套件組合能大幅減少 dependency hell。
2. **底層依賴清理：** 移除大型軟體（如 LibreOffice）時，徹底清理 `ure` 等殘留項對後續安裝非常重要。
3. **高版系統相容性：** 即使官方標註為 Ubuntu 22.04，在 24.04 上透過正確配置金鑰路徑（signed-by）仍可順利部署。

---
*文件產生日期：2026-03-02*
