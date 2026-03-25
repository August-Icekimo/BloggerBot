---
title: Gitea Actions + SBOM 技術實踐與踩坑全紀錄
labels: [Gitea, Actions, SBOM, Podman, Linux, CI/CD]
published: false
post_id: 8526467074969358082
---

# Gitea Actions + SBOM 環境說明與經驗總結

在軟體供應鏈安全日益受重視的今天，自動化產生軟體清單 (SBOM) 已成為 CI/CD 流程中的標準配備。本篇文章總結了在基於 Linux Mint + Podman 的 Gitea 環境下，如何透過 Gitea Actions 順利整合 Syft 與 CycloneDX，並記錄了多個關鍵的技術坑點。

---

## 為什麼需要這份紀錄？

{{sms-thread-start}}
{{sms-right: 之前的 SBOM workflow 總是在權限或路徑上出錯，有沒有一個穩定的參考範本？ | name=Dev | time=14:00}}
{{sms-left: c好的。這份紀錄是從 CEPP-HRM POC 的對話中提取的實務經驗，旨在幫助其他專案快速避開相同的坑。 | name=Claude | time=14:02}}
{{sms-thread-end}}

---

## 環境概覽

| 項目 | 內容 |
|------|------|
| **主機** | Linux Mint (Mint2Bee), icekimo, UID=1000 |
| **Gitea 版本** | 1.25.4（支援 `workflow_dispatch` UI 觸發） |
| **Gitea 部署方式** | Podman pod（`gitea-pod`），**非** Docker |
| **act_runner 部署** | ✅ Host binary 直接跑（不在 container 裡） |
| **act_runner 版本** | v0.3.0 |
| **Container runtime** | Podman 4.9.3 rootless（`/run/user/1000/podman/podman.sock`） |
| **SBOM 主力工具** | Syft（host binary，`~/.local/bin/syft`） |
| **SBOM 驗證工具** | cyclonedx-cli linux-x64 binary（workflow 內動態下載） |

---

## 檔案結構（已驗證可用）

```
~/gitWrk/gitea-local/
├── compose.yaml                    ← Podman pod 定義（server + runner）
└── runner_data/
    ├── .runner                     ← act_runner 自動產生，勿手動改
    ├── .token                      ← Registration token
    └── config.yaml                 ← act_runner 設定（關鍵！）

專案 Repo（任意路徑）/
└── .gitea/
    └── workflows/
        └── sbom.yaml               ← SBOM workflow（本文件附範本）
```

---

## act_runner 啟動方式

**⚠️ 不要用 container 版 runner**，用 host binary：

```bash
# 啟動
act_runner daemon --config ~/gitWrk/gitea-local/runner_data/config.yaml

# systemd 常駐（推薦）
systemctl --user start act_runner
systemctl --user status act_runner
```

systemd service 檔位置：`~/.config/systemd/user/act_runner.service`

---

## runner_data/config.yaml 關鍵設定

以下三個地方**必須正確**，否則 job container 會失敗：

```yaml
runner:
  # act_runner 讀 config 需靠環境變數 CONFIG_FILE 指定路徑
  # 若用 host binary 直接 --config 指定就不需要這個

container:
  network: "host"                      # ← 必須是 host，否則 job container 網路建立失敗
  docker_host: ""                      # ← 留空，讓 Syft 自己找；不要填 docker.sock
  valid_volumes:
    - "/home/icekimo/**"               # ← 允許掛載 HRM 專案目錄
    - "/run/user/1000/**"              # ← 允許掛載 Podman socket

  envs:
    {}                                 # ← 必須是空 map，不能有垃圾字串在裡面
                                       #   （sed 清理假變數時容易寫壞這裡）
```

---

## compose.yaml runner 區段（已驗證）

```yaml
  runner:
    image: gitea/act_runner:latest
    container_name: gitea_runner
    restart: always
    depends_on:
      - server
    volumes:
      - /run/user/1000/podman/podman.sock:/var/run/docker.sock
      - ./runner_data:/data
    environment:
      - GITEA_INSTANCE_URL=http://server:3000
      - GITEA_RUNNER_REGISTRATION_TOKEN_FILE=/data/.token
      - GITEA_RUNNER_NAME=Mazinger-Runner
      - CONFIG_FILE=/data/config.yaml  # ← 這行讓 run.sh 帶入 --config 參數
    security_opt:
      - label=disable
```

> **注意**：若改用 host binary 跑 runner，這個 compose.yaml 的 runner 區段就不需要了。
> host binary 方式是目前**唯一驗證可用**的方式。

---

## 已驗證的 SBOM Workflow 範本（sbom_v3.yaml）

下一個專案直接複製此範本，只需改**三個地方**：

```yaml
# ① 修改這裡
env:
  APP_NAME: "CEPP-HRM"               # ← 改成新專案名稱
  APP_VERSION: ${{ github.ref_name }}
  HRM_LIB_PATH: "./HRM/WEB-INF/lib"  # ← 改成新專案的 lib 路徑
  HRM_WAR_PATH: "./HRM"              # ← 改成新專案的 WAR 根目錄
  SBOM_OUTPUT_DIR: "/tmp/sbom-output" # ← 保持 /tmp，不要改回 ./
```

```yaml
# ② paths filter（push 觸發條件），改成對應的 lib 路徑
on:
  push:
    paths:
      - "HRM/WEB-INF/lib/**"         # ← 改成新專案路徑
```

```yaml
# ③ artifact name（避免多專案衝突）
      - name: 上傳 SBOM Artifacts
        uses: actions/upload-artifact@v3   # ← 必須是 v3，v4 不支援 Gitea
        with:
          name: sbom-${{ env.APP_NAME }}-${{ github.sha }}
```

---

## 踩坑清單（必讀）

### 🔴 坑 1：upload/download-artifact 必須用 v3

```yaml
# ❌ 錯誤 — Gitea 不支援
uses: actions/upload-artifact@v4
uses: actions/download-artifact@v4

# ✅ 正確
uses: actions/upload-artifact@v3
uses: actions/download-artifact@v3
```

---

### 🔴 坑 2：SBOM 輸出目錄不能用相對路徑（`./sbom-output`）

Host runner 模式下，`/workspace` 目錄不存在，相對路徑會解析到 `/workspace/...` 導致 `permission denied`。

```yaml
# ❌ 錯誤
SBOM_OUTPUT_DIR: "./sbom-output"

# ✅ 正確
SBOM_OUTPUT_DIR: "/tmp/sbom-output"
```

---

### 🔴 坑 3：CycloneDX CLI 驗證不要用 Docker

Host runner + Podman rootless 環境，`docker run` 掛 volume 會有 `/workspace` 權限問題。

```yaml
# ❌ 錯誤 — 不要用
docker run --rm \
  -v "$(pwd)/sbom-output:/work:ro" \
  cyclonedx/cyclonedx-cli validate ...

# ✅ 正確 — 直接下載 binary
wget -q \
  https://github.com/CycloneDX/cyclonedx-cli/releases/latest/download/cyclonedx-linux-x64 \
  -O /tmp/cyclonedx-cli
chmod +x /tmp/cyclonedx-cli
/tmp/cyclonedx-cli validate --input-file "$SBOM_FILE" --fail-on-errors
```

---

### 🔴 坑 4：act_runner container 版無法存取 Podman socket

Podman rootless socket 在 user namespace 裡，container 內的 root ≠ host 的 icekimo，
socket 雖然 mount 進去但實際無法通訊。

**唯一解法：用 host binary 跑 act_runner，不要跑在 container 裡。**

---

### 🟡 坑 5：config.yaml 的 `envs:` 不能有垃圾內容

用 `sed` 清理假環境變數時容易把路徑字串寫進 `envs:` 區塊，
導致 `yaml: unmarshal errors: cannot unmarshal !!str` 啟動失敗。

```yaml
# ❌ 錯誤（sed 清壞的樣子）
  envs:
    runner_data/config.yaml

# ✅ 正確
  envs:
    {}
```

---

### 🟡 坑 6：act_runner 不會自動讀 `/data/config.yaml`

container 版 `run.sh` 靠環境變數 `CONFIG_FILE` 決定是否帶 `--config`，
沒設這個變數就用預設值（無 config），`network: host` 等設定全部無效。

```yaml
# compose.yaml 必須加這行
environment:
  - CONFIG_FILE=/data/config.yaml
```

---

### 🟡 坑 7：`.runner` 的 address 可能殘留 localhost

自動產生的 `.runner` 有時會記錄 `"address": "http://localhost:3000"`，
但 container 版 runner 應該用 `http://server:3000`（service name）。
Host binary 版用 `http://localhost:3000` 是正確的（因為 port 3000 有對外 expose）。

---

### 🟡 坑 8：`podman compose` 在 Python 3.12 壞掉

Mint2Bee 上的 `docker-compose` 1.29.2 在 Python 3.12 缺少 `distutils` 模組。

```bash
# ❌ 會噴 ModuleNotFoundError: No module named 'distutils'
podman compose down
podman compose up -d

# ✅ 改用
podman pod restart gitea-pod
# 或安裝新版
pip3 install podman-compose --break-system-packages
```

---

## 下一個專案的 Checklist

開始新專案前，確認以下：

- [ ] `act_runner daemon` 正在跑（`ps aux | grep act_runner`）
- [ ] Gitea UI → Settings → Runners 看到 Mazinger-Runner 是 online
- [ ] 新 Repo 的 Settings → Actions 已啟用
- [ ] `APP_NAME` / `HRM_LIB_PATH` / `HRM_WAR_PATH` 已改成新專案的值
- [ ] `SBOM_OUTPUT_DIR` 是 `/tmp/sbom-output`（不是 `./`）
- [ ] `upload-artifact` / `download-artifact` 是 `@v3`
- [ ] CycloneDX CLI 驗證用 binary，不用 `docker run`
- [ ] `paths:` filter 已改成新專案的 lib 目錄

---

## 快速指令參考

```bash
# 確認 runner 狀態
ps aux | grep act_runner

# 手動觸發（空 commit）
git commit --allow-empty -m "ci: 觸發 SBOM 掃描" && git push origin main

# 偵察新專案 lib 結構
find ./專案/WEB-INF/lib -name "*.jar" | sort > /tmp/lib_list.txt
wc -l /tmp/lib_list.txt

# 偵察無版本號 JAR
find ./專案/WEB-INF/lib -name "*.jar" -exec basename {} \; \
  | grep -v -- '-[0-9]' | sort

# 本機快速 SBOM（不走 Gitea Actions）
syft dir:./專案/WEB-INF/lib \
  -o cyclonedx-json=/tmp/sbom-quick.json \
  --source-name "專案名稱"
```

---

## Open Questions

- ? 其他 5 個專案是否也是 Struts2 + 手動編譯的 WAR？還是有不同技術棧？
- ? 是否需要把多個專案的 SBOM 合併成一份（cyclonedx-cli merge）？
- ? Dependency-Track 是否已有自架環境，或需要另外建？
- ? Windows Gitea 何時移機？移機後 runner 要重新向新 Gitea 註冊。

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-03-11 | 從 CEPP-HRM POC 對話提取，涵蓋完整踩坑紀錄 |