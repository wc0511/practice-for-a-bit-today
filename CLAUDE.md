# 今天練一下（Practice for a bit today）

薩克斯風每日練習的 PWA。純靜態網站：沒有建置步驟、沒有框架、沒有 npm 相依套件。
網址：https://wc0511.github.io/practice-for-a-bit-today/（GitHub Pages，main 分支根目錄）
使用者主要用 iPhone（Safari 或加到主畫面）。

## 合作規則（一定要遵守）

- 回覆一律用**繁體中文、台灣用語**，包含總結和說明。
- 改完先說明改了什麼，等擁有者確認。擁有者說「**發布**」才可以 commit + push。每一次「發布」只對當下這一版有效。
- **每次發布都要把 `sw.js` 第二行的 `CACHE` 版本號 +1**（目前是 `practice-for-a-bit-today-v19`），不然手機會一直用舊的快取。
- push 之後大約等 1 分鐘，抓線上的 `sw.js` 確認版本號已經更新。
- 分頁名稱、副標、按鈕文字都是擁有者指定的，不要自己改。需要新名稱時，先提出建議讓擁有者選。
- 檔名用 ASCII。
- 規格有兩種讀法時（例如等級的音域、互動方式），先問擁有者，不要自己猜。
- 這個 repo 是**公開的**：commit 的內容（包括這份文件）不要放個人資料、帳號或公司資訊。

## 擁有者的偏好（做設計決定時參考）

- **這是協助進步的小工具，不是競技。** 不要加作答倒數、答錯懲罰、排名這類壓力設計。擁有者明確拒絕過作答倒數：音符可以限時消失（那是練習本身），但作答不限時。
- **畫面不要跳動。** 同一個功能在不同狀態下（說明、出題、作答、答完、換等級），鍵盤和按鈕要固定在同一個位置。寧可多留一點空白。
- **一個畫面內要能完成操作。** 用 iPhone 看的時候，不用捲動就要看得到題目、作答區和主要按鈕。可以用 390×844（加到主畫面）和 390×720（Safari 上下工具列都展開）這兩種尺寸檢查。
- **視譜練習不練聽力**：題目本身不發聲，按鍵的聲音只是回饋。
- 擁有者習慣用 Claude Code 桌面版右邊的瀏覽器窗格邊看邊調，常常會一次只改一個小地方，例如動畫秒數、顏色、文字。

## 檔案

| 檔案 | 內容 |
|---|---|
| `index.html` | 整個 app：HTML、CSS、JS 全部在這一個檔案裡 |
| `sw.js` | service worker。開頁面時先抓網路上的最新版，沒網路才用快取 |
| `manifest.webmanifest` | PWA 設定（名稱、顏色、圖示） |
| `icon-*.png`、`apple-touch-icon.png` | 圖示 |
| `.claude/launch.json` | Claude Code 桌面版的本機預覽設定（`static`：`python -m http.server 8000`） |

## index.html 結構

- `<style>`：用 `/* ---------- 區塊名 ---------- */` 分段。顏色都是 `:root` 上的 CSS 變數，有深色模式（`prefers-color-scheme` 和 `[data-theme]`）。
- `<body>` 的順序：
  - 首頁 `#splash`（按鈕 `#spGo`）；
  - `.appbar`，右上角是 `#tbMet` 節拍器和 `#tbTun` 調音器；
    - 只有「今天練這些」會顯示完整的標題列。其他三個分頁的 `body` 會加上 `bare`：標題列不佔位置，只留右上角的工具按鈕，跟著頁面捲動；持續音的停止按鈕移到工具按鈕下方（小教室則改用標語那一行的 `#clDrone`），小教室的分類列改黏在畫面最上面。
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
- **輸出總線 `outBus`**：一個限幅器（DynamicsCompressor，threshold −4 dB）。節拍器、持續音和視譜作答鍵的鋼琴聲（`pianoNote`）都接到這裡，開到最大也不會破音。節奏樂透的 `clickAt`、抖音節拍器和小教室的其他示範音（`playNotes`）目前還是直接接 `ctx.destination`。
- **音量拉桿**一律經過 `volGain(pos)`，用分貝刻度：拉滿是 0 dB，一半約 −18 dB，最左約 −34 dB。不要改回線性，之前線性刻度被回報「拉桿一半以上都一樣大聲」。
- **持續音**是加法合成。
  - 音色有 `DR_TONES.soft`（柔和，給耳機用）和 `DR_TONES.bright`（響亮，給手機喇叭用；把能量移到 2–8 倍泛音）。
  - `drPeak()` 把峰值正規化到 0.8。
  - 主音是平均律，從 A4 換算，可以選 A = 440 或 442。五度是純律的 1.5 倍。
- **工具節拍器**（`mtClick`）是木魚聲：兩個衰減的正弦波，加上一小段帶通雜訊。排程用 lookahead，每 25 ms 把接下來 0.12 秒的拍子排好。

### 樂器移調

- `INSTS` 定義四種樂器：`sop`、`alto`、`tenor`、`bari`，目前選的樂器用 `INS()` 取得。
- 畫面上的音名一律是**記譜音**，發聲一律用**實際音高**（`concertOf()`、`INS().off`）。
- 例外：**樂理小教室的示範音**（視譜作答鍵的 `pianoNote`，以及「▶ 聽」、調性、和弦的 `playNotes`）依 `pitchMode`（設定 key `pitchMode`）：預設 `concert` 標準音，照譜上寫的音高發聲；按小教室標語右側的「發聲：標準音／移調」切到 `inst` 才換算成樂器實際音高。持續音永遠用實際音高，不受這個設定影響。這是擁有者指定的。

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

- **首頁 splash** 會在每天第一次打開時出現；日期改變時也會再出現（回到前景時檢查，閒置時每 30 秒檢查一次）。英文副標跟中文用同一組無襯線字體（`--sans`）。按鈕「Go!」跳過 Manrope，直接用標題中文的字型（蘋方／思源黑體／微軟正黑體），字形才會像標題；白字黑框（`--sp-go`、`--sp-go-line`，`paint-order:stroke fill`）；「!」（`.sp-bang`）在首頁出現 0.7 秒後開始抖（每次 0.6 秒），停在首頁時每 3 秒重播一次。
- 每天的練習內容由日期決定，程式在「日期與輪替」那一段。
- **今天練這些**的項目在 `ITEMS`（`how` 可以是字串或函式，`score` 是卡片上「今日」下面的譜）。第 6 項「手指 · 放鬆加速」每天都做（30 分、15 分模式跳過），節奏在 `FING_PAT` 四天一輪（`durs` 以十六分音符為單位，3 = 附點八分）；譜由 `fingSVG()` 畫今天調性的大調音階往上 8 個音，符槓群由 `beamGroup()` 畫。
- **節奏樂透**的一拍節奏型在 `CELLS`，`lv` 對應難度按鈕 Lv1–Lv6（基礎、十六分、附點、切分、三連音、三連切分）；「附點 ↔ 三連音」特訓用 `fam` 挑節奏型（`dot`、`tri`），跟等級無關。三連音的 `ev` 存實際長度，繪譜時乘 3/2 換成記譜長度。
- **視譜練習**有 9 個等級，定義在 `SR_LV`：
  - `lo`／`hi` 是音域（記譜音的 MIDI），`show` 是音符幾毫秒後消失（0 = 不消失）。
  - `acc` 為 true 時含升降記號，改用一個八度的鋼琴鍵盤作答，同音異名都算對；`bare` 為 true 時鍵盤不標音名（等級 7–9）。
  - 目前的等級和最佳紀錄存在 `srLv2`、`srBest2`。
  - 譜面上下範圍依 `SR_INK`（依音符數實測的筆畫範圍）再留 6，改了音域或畫法要重新量，不然會切到筆畫或留太多白。
  - 譜面區 9 級統一用最高那一級的高度（`SR_HMAX`，目前是 Lv9），題目和說明文字都在裡面置中；換等級、說明、顯示題目、作答、答完時，鍵盤上緣都在同一個位置（擁有者要求，不要讓畫面跳動）。
  - 作答鍵（Lv1–4 的唱名鍵、Lv5–9 的鋼琴鍵）按下去會發出那個音的鋼琴聲（`srKeyTone` → `pianoNote`，合成的鋼琴聲，只是按鍵回饋，固定八度：標準音 C4–B4、移調時記譜 C5–B5）；題目本身不發聲，這裡不練聽力。
  - 不支援電腦鍵盤快捷鍵（擁有者決定拿掉）。

## 測試

- **本機預覽**：`python3 -m http.server 8000`（Windows 用 `python` 也可以），然後開 http://localhost:8000/。
  - `.claude/launch.json` 裡有一個叫 `static` 的設定，在 Claude Code 桌面版可以直接開右邊的瀏覽器窗格預覽。
  - service worker 在 localhost 也會啟用，改完看到舊畫面時，先在頁面裡取消註冊 service worker、清掉 caches，再重新整理；或在 DevTools 勾「Bypass for network」。
- **語法檢查**：把 `<script>` 的內容抽出來，跑 `node --check`：
  `node -e "const fs=require('fs');const s=fs.readFileSync('index.html','utf8');fs.writeFileSync('check.js',[...s.matchAll(/<script>([\s\S]*?)<\/script>/g)].map(m=>m[1]).join('\n;\n'))" && node --check check.js`（`check.js` 放暫存資料夾，不要 commit）
- **在瀏覽器裡驗證**：量元素位置（`getBoundingClientRect`）比只看截圖可靠。截圖有時會慢一步，或邊緣多截一塊。
  - 要測聲音，可以攔截 `createOscillator` 記下頻率，或在 `outBus` 接一個 AnalyserNode 看音量。
  - 限時的等級要等計時器跑完才能出下一題，可以暫時把 `setTimeout` 的長延遲縮短。
- **Playwright（Chromium）**：
  - context 設 `serviceWorkers: 'block'`，進頁面後先點 `#spGo` 關掉首頁。
  - 啟動參數加 `--autoplay-policy=no-user-gesture-required`。
  - 收音功能用假麥克風：`--use-fake-ui-for-media-stream --use-fake-device-for-media-stream --use-file-for-fake-audio-capture=<wav 檔>`，並給 context `permissions: ['microphone']`。
  - 要測日期或 iOS 的行為，用 `addInitScript` 假造 `Date` 或 `navigator.audioSession`。
  - 每次都要檢查有沒有 `pageerror`，並用 390px 寬的手機尺寸截圖看版面。
- **iPhone 實機**：聲音、靜音鍵、音量、收音靈敏度只能用實機確認。改到這些地方時，要請擁有者用 iPhone 測。

## 發布與部署

- push 需要登入 GitHub，由 Windows 的 Git Credential Manager 處理。登入失效時，push 會跳出 GitHub 登入視窗，由擁有者自己登入；不要替擁有者輸入帳號密碼。
- GitHub Pages 平常在 push 後 30 秒到 1 分鐘內更新。v13 有一次完全沒觸發部署，下一次 push 才把它一起帶上去。
  - 等太久時，查 `https://api.github.com/repos/wc0511/practice-for-a-bit-today/actions/runs?per_page=2`，看最新一筆的 `head_sha` 是不是剛 push 的 commit。
- 確認上線：抓 `https://wc0511.github.io/practice-for-a-bit-today/sw.js?t=<時間>`，看 `CACHE` 版本號。也可以抓 `index.html`，確認新程式碼在裡面。

## 待辦／想法

- 還沒做：進步追蹤分頁（可量化的進步指標）。
- **等擁有者用 iPhone 回報**：
  - 調音器和抖音神曲的收音靈敏度；
  - 視譜作答鍵的鋼琴聲：音量、音色，還有靜音鍵打開時有沒有聲音；
  - 首頁「Go!」白字黑框的樣子，以及驚嘆號的抖動；
  - 隱藏標題列後，右上角的節拍器、調音器好不好按；
  - 視譜換等級、按「開始」時，畫面有沒有跳動。
- **提過、但擁有者還沒決定的想法**：
  - 小教室「發聲：移調」的按鈕要不要顯示目前的樂器，例如「發聲：移調（次中音）」；
  - 「▶ 聽」、調性、和弦的示範音要不要也改成鋼琴聲；
  - 淺色模式的首頁按鈕底色接近白色，白字黑框比較不顯眼，要不要改成綠色底；
  - 用 390×720 檢查時（Safari 工具列都展開），視譜鋼琴等級的「開始／下一題」會被分頁列蓋住幾 px。
- iPhone 已確認正常：靜音模式下有聲音、v10 的音量與「響亮」音色。
