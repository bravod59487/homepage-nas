# homepage-nas — Homepage 儀表板設定

> **這個資料夾裡的容器不在這台 PC 上跑。**
> Homepage 部署在 **Synology NAS（192.168.3.100:3000）**，這裡放的是它的設定檔。

網址：`http://192.168.3.100:3000`

---

## 怎麼部署

DSM → **Container Manager → 專案 → 新增 → 選擇本資料夾**。

這個資料夾本身就是一個獨立的 **private git repo**，NAS 要拿到它有兩種方式：

**A. git clone（建議）**

```bash
# 在 NAS 上。第一次要先設好唯讀 deploy key，見下方「NAS 上的 git 設定」
git clone gh-homepage:bravod59487/homepage-nas.git /volume1/docker/homepage
cd /volume1/docker/homepage
cp .env.example .env          # 照註解填入金鑰；.env 不在版控裡
docker compose up -d
```

之後 PC 上改完設定 push，NAS 只要 `git pull && docker compose up -d`。

**B. 共用資料夾 / 手動複製** —— 記得 `.env` 要另外帶過去（它不會被 clone 下來）。

改完設定後在 Container Manager 重建專案，或

```bash
# 在 NAS 上
docker compose up -d
docker restart homepage      # 大部分 yaml 改動 Homepage 會自己 reload
```

---

## 雙來源架構

Homepage 同時顯示兩台機器的容器狀態：

| `server:` | 讀取方式 | 對象 |
|---|---|---|
| `local` | 直接掛 NAS 本機的 `/var/run/docker.sock`（唯讀） | NAS 上的容器 |
| `pc` | 透過 PC 的唯讀 socket-proxy（`192.168.3.11:2375`） | 這台 PC 的容器 |

設定在 `config\docker.yaml`。PC 端的細節見 `..\socket-proxy\README.md`。

> PC 關機時 Homepage 的「PC 主機」區塊會顯示離線 —— 這是刻意的，正好當作
> PC 是否在線的指示。

---

## 必填的環境變數

```yaml
- HOMEPAGE_ALLOWED_HOSTS=localhost:3000,127.0.0.1:3000,192.168.3.100:3000,homeassistant.local:3000
```

**沒填這行會直接顯示 `Host validation failed`，整個頁面打不開。**
以後多一個存取網址（例如反向代理的域名）就要加進這個清單。

---

## 設定檔

| 檔案 | 內容 |
|---|---|
| `config\settings.yaml` | 標題、深色主題、語言 zh-TW、六個分類的版面與圖示 |
| `config\services.yaml` | 所有服務卡片與 widget（最主要的檔案） |
| `config\widgets.yaml` | 頁首：logo、資源用量、搜尋、台北天氣、時間 |
| `config\bookmarks.yaml` | 書籤：常用網站 + Homepage 官方文件連結 |
| `config\custom.css` | 卡片間距微調 |
| `config\custom.js` | 空的，備用 |
| `.env` | **所有主機位址、帳密、API key（不進版控）** |

分類（`settings.yaml` 的 `layout`）：媒體、智慧家庭、生產力、基礎設施、家庭應用、PC 主機。

---

## .env — 唯一的機密來源

`services.yaml` 裡一律用 `{{HOMEPAGE_VAR_XXX}}` 佔位，實際值只寫在 `.env`。
`.env` 已被根目錄 `.gitignore` 排除（`.env` / `**/.env` / `*.env` 三條都有擋）。

位址類：`NAS`、`PC`、`ROUTER`、`DECO`、`QNAP`、`N8N`
金鑰 / 帳密類：`JELLYFIN_KEY`、`IMMICH_KEY`、`HASS_TOKEN`、`KUMA_SLUG`、
`DSM_USER/PASS`、`QNAP_USER/PASS`、`NC_USER/PASS`、`CALIBRE_USER/PASS`、
`ADGUARD_USER/PASS`

**新增服務的 widget 時，帳密一律加到 `.env` 用變數引用，不要直接寫進 `services.yaml`。**

---

## 已知待補

`services.yaml` 開頭註記著：部分 **NAS 上的容器名稱還沒填**。
`container:` 沒填的話功能照常，只是少了 running/stopped 燈號與 CPU/RAM 數字。

要補的話到 DSM → Container Manager → 容器，把實際名稱填進對應服務的
`container:` 欄位。目前已填的有 `immich_server`、`calibre-web`、
`homeassistant`、`nextcloud-app`、`n8n`、`uptime-kuma`、`adguardhome`、
`homepage`、`catv_*`、`dbi_nginx`、`e-paper_photo_frame-epf-1`。

另外 `widgets.yaml` 的 `resources` 區塊在 Docker Desktop 環境下讀到的是
**WSL2 虛擬機**的數字，不是 Windows 主機真實用量 —— 不過這份是跑在 NAS 上，
所以顯示的是 NAS 本身，這條註解是從 PC 版設定沿用下來的。

---

## `images\` 資料夾

compose 掛了 `./images:/app/public/images` 給自訂圖示 / 背景用。
這個資料夾在 `.gitignore` 裡（`images/`），目前不存在，
第一次啟動時 Docker 會自動建立。

圖示優先用內建名稱（`jellyfin.png`）或 Material Design Icons（`mdi-cctv`），
兩者都找不到才需要自己丟檔案進 `images\`。

---

## NAS 上的 git 設定

Clone private repo 用**唯讀 deploy key**，權限只限這一個 repo。

```bash
# 在 NAS 上（DSM 需先啟用 SSH、啟用使用者家目錄服務、裝好 Git Server 套件）
ssh-keygen -t ed25519 -C "nas-homepage" -f ~/.ssh/id_ed25519_homepage -N ""
cat ~/.ssh/id_ed25519_homepage.pub
# → 貼到 GitHub repo → Settings → Deploy keys → Add（不要勾 Allow write access）

cat >> ~/.ssh/config <<'CFG'
Host gh-homepage
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_homepage
  IdentitiesOnly yes
CFG
chmod 700 ~/.ssh && chmod 600 ~/.ssh/config ~/.ssh/id_ed25519_homepage
ssh-keyscan github.com >> ~/.ssh/known_hosts
ssh -T gh-homepage        # 看到 successfully authenticated 就成功
```

> **不要用 root 跑排程 `git pull`** —— root 讀不到你 SSH 使用者家目錄下的金鑰。
> 任務排程器裡的「使用者」要選建立金鑰的那個帳號。
