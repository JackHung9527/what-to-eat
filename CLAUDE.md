# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案定位

「吃什麼？」是一個 **純前端、零伺服器、零相依** 的個人口袋名單 PWA，目的在於「隨機抽選今天要吃哪一家店」。整個專案只有兩個檔案：

- `index.html` — 整個 app（HTML + CSS + JavaScript 全部內聯在同一檔）
- `README.md` — 使用者面向的完整說明文件

## 建置、測試、部署

- **無建置工具**：沒有 npm、webpack、bundler，也沒有任何 package.json。修改後直接開瀏覽器即可預覽。
- **無測試框架**：此專案沒有自動化測試。驗證方式是開啟 `index.html` 手動測試主要流程（新增店家、抽店家、匯出匯入、Google Maps 連結解析）。
- **本地預覽**：直接用瀏覽器打開 `index.html`，或任意 static server（例如 `python -m http.server`）。PWA 行為需透過 HTTPS / localhost 才會完整。
- **部署**：push 到 `main` 分支後，GitHub Pages 會在 1–2 分鐘內自動將 `index.html` 發佈到 <https://jackhung9527.github.io/what-to-eat/>。沒有 CI、沒有 build step。
- **Git 主分支**：`main`。

## 架構重點

### 單檔 SPA（Single-File App）

所有程式都在 `index.html`，大致分布：

- L1–L1530：`<style>` 區塊，放所有 CSS（含籤筒 / 扭蛋機 / 結果紙等視覺元件）。
- L1531–L1540：`TWEAK_DEFAULTS` 被 `/*EDITMODE-BEGIN*/ ... /*EDITMODE-END*/` 標記包住；**此區塊會被外部編輯模式工具動態改寫**，修改時務必保留註解標記，否則 edit-mode 會壞掉。
- L1540–L1565：全域資料字典（`CATEGORIES`、`MEAL_LABELS`、`PRICE_BUCKETS`、`SAMPLE`）——新增餐別 / 種類 / 價位 bucket 都從這裡動手，UI 會自動渲染。
- L1566 以後：狀態、渲染、事件處理、Google Maps URL 解析、匯出匯入等邏輯。

### 資料模型與 localStorage

- 唯一的 storage key：`wte_stores`，存一個 store 陣列的 JSON。
- Store 欄位：`{ id, name, meals[], categories[], stars, prices[], address, hours, note, lat, lng }`。
- **Schema migration**：舊版只有單一 `category` 欄位，`loadStores()` 會在載入時自動把它轉成 `categories[]`；新增欄位或改 schema 時，請在 `loadStores()` 加上對應的 migration，**不要**直接丟掉舊欄位，以免老使用者資料壞掉。
- `allCats(s)` / `firstCat(s)` helper 會同時容納新舊 schema，讀取 category 時永遠透過它們，不要直接 `s.category`。

### 匯入解析

`parseInput(raw)` 是匯入功能入口，會依內容分派到：

- `parseGoogleMapsUrl(raw)`：處理 `/maps/place/...`、`/maps/search/...`、`?q=...`、`@lat,lng`、`!3d!4d` 等格式。**不支援** `maps.app.goo.gl` 短連結（瀏覽器 CORS 無法展開），文件已說明解法。
- `parsePlainTextAddress(raw)`：處理「郵遞區號+地址+店名」黏在一起的複製格式。正則寫死在 L2180 附近，若要擴充地址格式請修改 `addrPattern`。

### UI 狀態

分 3 個頁面（扭蛋抽店、抽方向、我的名單），由 `showPage(name)` 切換。三個頁面共用同一個 modal（新增/編輯店家）。`formState` 是新增表單的暫存狀態，切出 modal 時要記得 reset。

### PWA / 匯出行為

匯出優先使用 `<a download>`；偵測到 PWA standalone 模式時會 fallback 到 Web Share API（否則 iOS PWA 無法下載檔案）。改匯出邏輯時記得兩條路徑都測。

## 程式風格約定

- 目前的 JavaScript 沒有 module 化、沒有 class，函式採 top-level 宣告。修改時維持同風格，避免引入 build step。
- XSS 防護：所有寫入 innerHTML 的使用者資料都要走 `esc()`。新增渲染邏輯時記得套用。
- 不要引入外部 JS / CSS 相依（除了現有的 Google Fonts），以維持「零伺服器、離線可用」的核心定位。

---

## 今日總結

### 2026/04/22

#### 完成項目
- 建立專案 CLAUDE.md，記錄單檔 SPA 架構、localStorage schema 與 migration 規則、`EDITMODE` 標記、匯入解析與 PWA 匯出雙路徑等非顯而易見的重點
- 修正手機上底部 nav 擋住主內容：新增 CSS 變數 `--nav-h: 84px`，`body` 底部 padding 改為 `calc(env(safe-area-inset-bottom) + var(--nav-h) + 32px)`，把原本僅 12px 的緩衝拉高到約 32px

#### 問題與踩坑
- 底部 nav 實際高度（8 + 68 + 8 + safe-area ≈ 84 + safe-area）超過原先 `body` 保留的 96px 緩衝空間，導致最後一張 store-card 在某些手機上被貼到 nav 邊界。根因為 nav-btn 的 glyph(30px) + gap(4px) + 文字 + 上下 padding 比預期高，已透過集中式 `--nav-h` 變數加 buffer 解決
