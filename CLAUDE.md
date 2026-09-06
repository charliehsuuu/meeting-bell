# 會議鈴 (Meeting Bell)

一個純前端的會議計時器 PWA。使用者先列出議程（每個項目一個名稱＋分鐘數），開會時逐項倒數計時，時間到會自動響鈴提醒，超時後每分鐘再響一次；開完會產生一份「分配時間 vs 實際時間」的紀錄表。

沒有後端、沒有帳號、沒有網路請求（除了載入自己），所有狀態存在瀏覽器 `localStorage`。目標使用情境是主持人把手機放在會議桌上當計時鈴用。

## 架構

**整個 app 就是一個檔案：[index.html](index.html)**（HTML + CSS + JS 全部內聯，無建置流程、無框架、無套件依賴）。其他檔案都是靜態資產：

| 檔案 | 用途 |
|---|---|
| [index.html](index.html) | 全部邏輯與畫面 |
| [sw.js](sw.js) | Service Worker，快取 app shell 以支援離線／PWA 安裝 |
| [manifest.webmanifest](manifest.webmanifest) | PWA manifest（圖示、standalone 顯示模式、繁中） |
| `icon-*.png` | 各尺寸 app 圖示（含 maskable） |
| `bell.wav` | 鈴聲來源檔（**僅供參考／重新產生用**，實際播放的音檔是 base64 內嵌在 index.html 的 `B64` 常數裡，並非在執行期載入這個檔案） |

**沒有 build step。** 修改後直接存檔、重新整理瀏覽器即可看到結果，不需要 `npm install`、bundler 或任何編譯動作。部署方式就是把這幾個靜態檔案丟到任何靜態主機上。

### 核心狀態（都在 `<script>` 內，全域變數）

- `items`：議程陣列 `{n: 名稱, m: 分鐘, used?: 實際秒數}`，存在 `localStorage` key `meetingbell.v1`
- `opts`：`{warn, over, vol}` — 是否 1 分鐘預告、是否超時每分鐘再響、音量百分比
- `idx` / `running` / `startAt` / `acc`：目前議程項目索引、是否計時中、本段開始時間戳、已累積毫秒數（暫停/切換項目時累加進 `acc`，避免用 `Date.now()` 直接算會受暫停影響）
- `firedWarn` / `firedEnd` / `lastOver`：本項目是否已響過預告／已響過時間到／上次超時響鈴的秒數門檻，換項目時（`jump`）要重置

### 三個畫面（view），用 `.hide` class 切換，不是路由

`view()` 函式控制三個 `<section>` 互斥顯示：
1. **run**（預設）：倒數計時主畫面 + 上一項/開始暫停/下一項 + 按鈴 + 音量
2. **editor**：編輯議程（新增/刪除/改名/改分鐘數），對應 `#edit` 按鈕切換進出
3. **report**：結束後的時間紀錄表格（分配 vs 實際 vs 差異）

### 音效與計時機制

- `AudioContext` 延遲到第一次互動才建立（`audio()`），並在 `touchstart` 時預先解鎖，避開行動瀏覽器的自動播放限制
- 鈴聲用 Web Audio 的 `AudioBufferSourceNode` 播放解碼後的 base64 WAV，可疊多下（`ring(n)` 用固定間隔排程多次 `hit`）模擬連續響鈴，並觸發 `navigator.vibrate`
- 計時用 `setInterval(tick, 250)` 輪詢並比對 `alloc() - spent()`，不是用 `setTimeout` 排到未來時間點 —— 修改計時邏輯時要注意這個輪詢間隔（250ms）決定了響鈴的最大延遲誤差
- 計時中會嘗試拿 Screen Wake Lock（`wake()`），並在 `visibilitychange` 恢復可見時重新申請，因為系統可能在切到背景時自動釋放

## 設計規範

### 視覺

- 深色系「復古黃銅」配色，全部走 CSS 變數（`:root` 裡的 `--ink` / `--brass` / `--paper` / `--alarm` 等），新增樣式一律引用變數，不要寫死色碼
- 字體用系統字型堆疊（含繁中字型 `Noto Sans TC` / `PingFang TC` / `Microsoft JhengHei`），不外掛字型檔
- 手機直式單欄版面（`max-width:520px` 置中），大數字倒數時鐘用 `clamp()` 做響應式字級
- 狀態用顏色語意化：正常＝白／米白，剩餘 ≤60 秒＝黃銅 `warn`，超時＝紅 `over`

### 語言與文案

- 介面語言固定為繁體中文（`lang="zh-Hant"`），新增文案一律用繁中，語氣簡短口語（例：「上一項」「按 鈴」「套用議程」）

### 程式風格（沿用既有寫法，維持一致）

- 全部塞在一個 `<script>` 裡，變數與函式命名刻意簡短（`$`、`fmt`、`esc`、`hit`），這是刻意的極簡風格，不要在既有程式碼中改成長名稱或拆成模組
- 用 `$(selector)` 這個小工具取代 `document.querySelector`，新增 DOM 操作沿用它
- 不使用任何框架、不引入外部套件；新增功能請直接寫原生 JS/CSS，維持零依賴
- 使用者輸入（議程名稱）顯示前一律用 `esc()` 做 HTML escape（見 `drawRows` / `finish`），避免 XSS —— 這是既有防線，新增會顯示使用者輸入的地方也要套用

### 資料持久化

- 唯一的持久化管道是 `localStorage`（key: `meetingbell.v1`），存取都包在 `try/catch` 裡（隱私模式或容量滿了可能丟例外，靜默失敗即可，不需要跳錯誤訊息）
- 沒有雲端同步、沒有匯出/匯入功能 —— 如果要加，需另外設計格式與 UI，目前完全沒有這塊

## 修改時的注意事項

- 動到 `index.html` 的檔案結構（哪些檔案會被快取）時，記得同步更新 [sw.js](sw.js) 裡的 `F` 陣列與快取版本號 `C`（`meeting-bell-v2`），否則舊使用者的 Service Worker 不會抓到新檔案
- `B64` 那行是內嵌的鈴聲音檔（base64 WAV），檔案很長、不要手動編輯；要換鈴聲的話從 `bell.wav`（或新音檔）重新編碼取代整段字串
- 沒有測試框架、沒有 lint 設定 —— 目前驗證方式是直接在瀏覽器（含手機 Safari，因為有 iOS PWA 相關 meta tag 與 Wake Lock）手動操作驗證
