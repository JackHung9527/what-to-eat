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
- 建立專案 CLAUDE.md 初版，記錄單檔 SPA 架構、localStorage schema 與 migration 規則、`EDITMODE` 標記、匯入解析、PWA 匯出雙路徑等非顯而易見的重點
- 底部 nav 擋內容問題最終解法：從 CSS padding 調整 → JS 動態量測 → 硬保底值，全部 iterate 過仍卡關，**根本改造**為右下 FAB + 彈出 sheet：
  - 新增 `.nav-fab`（右下 56×56 圓形按鈕，顯示當前頁面 glyph）
  - 新增 `.nav-sheet-overlay` / `.nav-sheet`（點 FAB 從底部滑上來的選單）
  - `switchPage(pageId)` 統一處理 page 切換、sheet 關閉、FAB glyph 同步（`FAB_LABELS`）、listPage 自動 `renderStoreList()`
  - 移除 `.bottom-nav` / `.nav-btn` / `--nav-h` CSS 變數 / `syncNavPadding()` JS
- 補上 iOS PWA 識別 meta：`apple-mobile-web-app-capable`、`mobile-web-app-capable`、`apple-mobile-web-app-status-bar-style`、`apple-mobile-web-app-title`
- 抽店家 / 抽方向加入「避免連抽相同」邏輯：pool > 1 時以 `lastPickedId` / `lastPickedCat` 排除上次結果；pool 只剩 1 家時 toast 提示使用者新增更多店家
- 結果卡新增「價格」專屬 row（與地址、時間、備註同格式），比原本 meta row 的小 `＄` tag 醒目得多
- FAB 在 sheet 開啟時淡出：透過 `body.nav-sheet-open` class 驅動 `opacity: 0` + `pointer-events: none`，與 sheet slide 動畫同步（0.25s）；抽出 `openNavSheet()` / `closeNavSheet()` helper 統一管理 overlay + body class
- FAB 換主題色：背景從 `var(--ink)` 改為 `var(--vermilion)`（抹茶綠），並加 `var(--vermilion-dark)` 2px 描邊與抹茶色陰影，跟全黑的 `pick-btn` 做視覺區隔避免混淆

#### 問題與踩坑
- **iOS PWA 一旦加入主畫面就凍結 HTML 快取**：網頁更新後 PWA 不會跟著，必須刪除主畫面 icon 重新加入。改了很多次 CSS/JS 使用者一直看到舊版，診斷耗時。此為 iOS 設計，無法從程式端繞過
- `env(safe-area-inset-bottom)` 在 PWA 與 Safari 模式回傳值不同，靜態 CSS 加固定 buffer 很難全 cover 所有裝置
- `document.fonts.ready` Promise 在 iOS PWA 下可能不 resolve，改用 `setTimeout(300)` / `setTimeout(1500)` 多時間點補算
- 最後判斷繼續 iterate padding 值 ROI 太低，直接改造 nav 形式徹底繞開——這個決策來自使用者明確要求「下面的選單可以移到別的地方嗎」

#### 架構變更備註（供下次 Claude 參照）
- 切換 page 統一走 `switchPage(pageId)`；不要再找 `.nav-btn` 或操作 `.bottom-nav`
- `body` 底部 padding 已回歸正常值 `calc(env(safe-area-inset-bottom) + 32px)`，不再需要保留 nav 空間（FAB 是 fixed 獨立小按鈕，不大範圍擋內容）
- `FAB_LABELS = { pickPage: '抽', catPage: '方', listPage: '單' }`，新增 page 要同步這張對應表
