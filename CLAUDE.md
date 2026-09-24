# 今天練一下（Practice for a bit today）

薩克斯風每日練習的 PWA。純靜態網站：沒有建置步驟、沒有框架、沒有 npm 相依套件。
網址：https://wc0511.github.io/practice-for-a-bit-today/（GitHub Pages，main 分支根目錄）
使用者主要用 iPhone（Safari 或加到主畫面）。

## 合作規則（一定要遵守）

- 回覆一律用**繁體中文、台灣用語**，包含總結和說明。
- 改完先說明改了什麼，等擁有者確認。擁有者說「**發布**」才可以 commit + push。每一次「發布」只對當下這一版有效。
- **每次發布都要把 `sw.js` 第二行的 `CACHE` 版本號 +1**（目前是 `practice-for-a-bit-today-v13`），不然手機會一直用舊的快取。
- push 之後大約等 1 分鐘，抓線上的 `sw.js` 確認版本號已經更新。
- 分頁名稱、副標、按鈕文字都是擁有者指定的，不要自己改。
- 檔名用 ASCII。

## 檔案

| 檔案 | 內容 |
|---|---|
| `index.html` | 整個 app：HTML、CSS、JS 全部在這一個檔案裡 |
| `sw.js` | service worker。開頁面時先抓網路上的最新版，沒網路才用快取 |
| `manifest.webmanifest` | PWA 設定（名稱、顏色、圖示） |
| `icon-*.png`、`apple-touch-icon.png` | 圖示 |

## index.html 結構

- `<style>`：用 `/* ---------- 區塊名 ---------- */` 分段。顏色都是 `:root` 上的 CSS 變數，有深色模式（`prefers-color-scheme` 和 `[data-theme]`）。
- `<body>` 的順序：
  - 首頁 `#splash`（按鈕 `#spGo`）；
  - `.appbar`，右上角是 `#tbMet` 節拍器和 `#tbTun` 調音器；
  - 工具面板 `#tool`（內含 `#toolMet`、`#toolTun`）；
  - 四個分頁；
  - `.tabbar`。
- 四個分頁（標題：副標）：
  - `#view-today` 今天練這些：有練有進步，路過別錯過
  - `#view-rhythm` 節奏樂透：運氣也是一種實力
  - `#view-vib` 抖音神曲：抖音一響，觀眾鼓掌
  - `#view-class` 樂理小教室：慘了，越看越有精神
- 小教室的子區塊是 `CL_SECS`：`sight` 視譜練習、`keys` 調性、`terms` 音樂術語、`symbols` 樂譜符號、`chords` 和弦、`fing` 指法表、`etudes` 練習曲庫。
- 主程式是一個 `(function(){ ... })()`，用 `/* ============ 區塊名 ============ */` 分段。最後另外有一段 `<script>` 負責註冊 service worker。

## 改之前要知道的技術細節

### 設定儲存

- 設定用 `load(key, 預設值)` / `save(key, 值)` 存取，實際存在 `localStorage` 的 `tenorapp:<key>`。
- 存值的意義變了就換新的 key（例如音量改成分貝刻度時換成 `drVol2`、`mtVol2`），舊值才不會套到新的意思上。

### 音訊

- 全部共用一個 `AudioContext`，就是 `ctx`。要出聲前一定先呼叫 `ensureCtx()`，而且要在使用者點擊的事件裡呼叫。
- **iPhone 靜音鍵**：用 `navigator.audioSession.type` 處理，平常設 `"playback"`，收音時設 `"play-and-record"`，由 `audioSessionFor()` 決定。舊版 iOS 改用 `iosUnlock()` 播一段無聲 WAV。
- **輸出總線 `outBus`**：一個限幅器（DynamicsCompressor，threshold −4 dB）。節拍器和持續音都接到這裡，開到最大也不會破音。節奏樂透的 `clickAt`、抖音節拍器和小教室的示範音目前還是直接接 `ctx.destination`。
- **音量拉桿**一律經過 `volGain(pos)`，用分貝刻度：拉滿是 0 dB，一半約 −18 dB，最左約 −34 dB。不要改回線性，之前線性刻度被回報「拉桿一半以上都一樣大聲」。
- **持續音**是加法合成。
  - 音色有 `DR_TONES.soft`（柔和，給耳機用）和 `DR_TONES.bright`（響亮，給手機喇叭用；把能量移到 2–8 倍泛音）。
  - `drPeak()` 把峰值正規化到 0.8。
  - 主音是平均律，從 A4 換算，可以選 A = 440 或 442。五度是純律的 1.5 倍。
- **工具節拍器**（`mtClick`）是木魚聲：兩個衰減的正弦波，加上一小段帶通雜訊。排程用 lookahead，每 25 ms 把接下來 0.12 秒的拍子排好。

### 樂器移調

- `INSTS` 定義四種樂器：`sop`、`alto`、`tenor`、`bari`，目前選的樂器用 `INS()` 取得。
- 畫面上的音名一律是**記譜音**，發聲一律用**實際音高**（`concertOf()`、`INS().off`）。

### 收音與音高偵測

- 演算法是 McLeod（NSDF），先降取樣 2 倍再算。
- 雜訊門檻會自己適應環境：沒吹的時候慢慢往上追，有聲音的時候很快往下掉。
- 靈敏度設定在 `MIC_SENS`（`high`、`mid`、`low`）。
- 調音器有 3 格的遲滯，再加上 EMA 平滑。

### 五線譜

- 用 SVG 畫，字型是內嵌的 `"TenorMusic"`（Noto Music 的子集，以 base64 woff2 放在 CSS 裡）。
- 線距 `SP = 8`，音高用 step 表示（E4 = 0）。字形的基線在最下面那條線上。
- 子集裡只有目前用到的字形。要用新的音樂符號，得從 Noto Music 重新做子集，再換掉 base64。

### 其他

- **首頁 splash** 會在每天第一次打開時出現；日期改變時也會再出現（回到前景時檢查，閒置時每 30 秒檢查一次）。
- 每天的練習內容由日期決定，程式在「日期與輪替」那一段。
- **節奏樂透**的一拍節奏型在 `CELLS`，`lv` 對應難度按鈕 Lv1–Lv6（基礎、十六分、附點、切分、三連音、三連切分）；「附點 ↔ 三連音」特訓用 `fam` 挑節奏型（`dot`、`tri`），跟等級無關。三連音的 `ev` 存實際長度，繪譜時乘 3/2 換成記譜長度。
- **視譜練習**有 9 個等級，定義在 `SR_LV`：
  - `lo`／`hi` 是音域（記譜音的 MIDI），`show` 是音符幾毫秒後消失（0 = 不消失）。
  - `acc` 為 true 時含升降記號，改用一個八度的鋼琴鍵盤作答，同音異名都算對；`bare` 為 true 時鍵盤不標音名（等級 7–9）。
  - 目前的等級和最佳紀錄存在 `srLv2`、`srBest2`。
  - 譜面上下範圍依 `SR_INK`（依音符數實測的筆畫範圍）再留 6，改了音域或畫法要重新量，不然會切到筆畫或留太多白。
  - 鋼琴鍵按下去會發出記譜 C5–B5 那個音（`srKeyTone`，固定八度，只是按鍵回饋）；題目本身不發聲，這裡不練聽力。
  - 不支援電腦鍵盤快捷鍵（擁有者決定拿掉）。

## 測試

- **本機預覽**：`python3 -m http.server 8000`，然後開 http://localhost:8000/。service worker 在 localhost 也會啟用，改完記得強制重新整理，或在 DevTools 勾「Bypass for network」。
- **語法檢查**：把 `<script>` 的內容抽出來，跑 `node --check`。
- **Playwright（Chromium）**：
  - context 設 `serviceWorkers: 'block'`，進頁面後先點 `#spGo` 關掉首頁。
  - 啟動參數加 `--autoplay-policy=no-user-gesture-required`。
  - 收音功能用假麥克風：`--use-fake-ui-for-media-stream --use-fake-device-for-media-stream --use-file-for-fake-audio-capture=<wav 檔>`，並給 context `permissions: ['microphone']`。
  - 要測日期或 iOS 的行為，用 `addInitScript` 假造 `Date` 或 `navigator.audioSession`。
  - 每次都要檢查有沒有 `pageerror`，並用 390px 寬的手機尺寸截圖看版面。
- **iPhone 實機**：聲音、靜音鍵、音量、收音靈敏度只能用實機確認。改到這些地方時，要請擁有者用 iPhone 測。

## 待辦／想法

- 還沒做：進步追蹤分頁（可量化的進步指標）。
- 等擁有者用 iPhone 回報：調音器和抖音神曲的收音靈敏度。
- iPhone 已確認正常：靜音模式下有聲音、v10 的音量與「響亮」音色。
