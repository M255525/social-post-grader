# CLAUDE.md — 社群貼文健診站（social-post-grader）

單檔前端工具：貼上一則社群貼文全文 → 依 8 個維度給出評分（0-100）＋一句話說明＋具體建議 → 產生一份「改善後貼文」處方箋。跟 `SocialPost`（從主題**生成**貼文）不同，這是對**既有貼文做評分＋改寫**的健診工具。無建置步驟、無框架、無 package.json，直接開啟 `index.html`（`file://`）或以靜態伺服器託管即可。

**此資料夾本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

**不套用序號授權**（`member-license-gate`）：比照 `food-finder`／`coffee-ig-planner`／`restaurant-feasibility-calculator` 等「一般用途、無教學版」工具的慣例。**無可攜式桌面版 exe**：比照 `coffee-ig-planner`，純網頁工具本機用 `python -m http.server` 預覽即可。

## 架構

單一 `index.html`：內嵌 `<style>` 與三個獨立 `<script>` IIFE（主程式／跑馬燈／PWA安裝），無外部資源除了選用的 AI API fetch。

### 8 個評量維度（`DIMENSIONS`）

`hook`（開頭吸引力）／`clarity`（核心訊息清晰度）／`toneFit`（語氣與受眾契合度）／`cta`（CTA明確度）／`readability`（長度與排版易讀性）／`hashtag`（Hashtag使用）／`emotion`（情感共鳴／故事性）／`algoFriendly`（演算法友善度），權重分別為 15/15/10/15/10/10/15/10%。**總分永遠由前端依這組權重自行加權計算**（`computeOverall()`），不信任 AI 自行回傳的總分，避免 AI 加權邏輯跟前端定義不一致。等第換算：90-100=A／75-89=B／60-74=C／&lt;60=D（`gradeOf()`），對應綠/青/黃/紅四色（`GRADE_COLOR`）。

### 平台規則表（`PLATFORM_RULES`）

Facebook／Instagram／X／Threads／通用五種，各自定義 `charMin`/`charMax`（建議字數範圍）、`hardLimit`（平台硬性上限，通用無上限故為 `null`）、`hookCutoff`（App「查看更多」摺疊點，目前資料表中保留但規則式評分未實際使用，供未來擴充）、`hashtagMin`/`hashtagMax`、`toneNote`、`algoNote`。新增/調整平台規則只需改這個表，`scoreReadability()`/`scoreHashtag()`/`buildEvaluatePrompt()`/字數提示 UI 都吃同一份資料。

### 規則式健診引擎（免 AI，`computeRuleDimensions()`）

8 個 `score*()` 函式各自回傳 `{score, comment, suggestion}`，純字串/正則規則、不連網：
- `scoreHook`：首句字數 8-30 字加分、陳腔濫調開頭（「大家好」等）扣分、含數字/問句/emoji 加分
- `scoreClarity`：句子平均長度、是否單一超長句、是否同時命中多組主題關鍵詞（優惠/報名/公告/徵才）
- `scoreToneFit`：依 `PURPOSE_TONE_EXPECT` 映射表比對正式/口語標記詞出現次數；未填目的與受眾時給中性 70 分
- `scoreCta`：CTA 關鍵詞表比對 + 位置是否在文末 60% + 是否含具體時間/地點詞
- `scoreReadability`：依平台字數範圍比對、偵測無換行長文字磚（`avgParaLen > 150`）
- `scoreHashtag`：依平台 hashtag 建議區間比對、偵測虛詞湊數 hashtag
- `scoreEmotion`：第一人稱代名詞計數 + 感官/情緒詞庫命中數
- `scoreAlgoFriendly`：連續驚嘆號/全大寫/誇大詞彙（保證/最強/全網最低）/engagement bait 句式（留言+數字/快分享/標記朋友）偵測，起始分 90 分逐項扣分

`ruleBasedImprove(text, platformKey)` 產生規則式改善版貼文：陳腔濫調或過長開頭時加提問式開場、每 2 句重新分段、無 CTA 時補通用 CTA 句、hashtag 不足時用 `extractHashtags()`（沿用 `SocialPost` 的停用詞表關鍵詞抽取邏輯）補到平台建議下限，超過平台硬上限則截斷。回傳值含 `isWeakHook`/`hasCta`/`addedTags` 供 `ruleBasedRationale()` 組出對應的改寫理由條列。

### AI 深度健診（選用，BYOK，比照 `SocialPost`/`coffee-ig-planner` 既有模式）

`AI_PROVIDERS`（Claude/OpenAI/Gemini/OpenRouter）、`callLLM()`（Claude 需 `anthropic-dangerous-direct-browser-access` header、429/500/503/529 重試 3 次、180 秒逾時）、`extractJsonObject()` 皆為同一套實作，修改任一邊時考慮是否同步其他姊妹工具。**與姊妹工具的差異**：本工具不支援圖片輸入（純文字健診），`callLLM()` 簡化為純文字 prompt，無 `imagesDataUrls` 參數。

**常用模型下拉選單（`MODEL_OPTIONS`，2026-08-27 新增，本工具獨有的模式）**：使用者要求「增加常用的語言模型讓使用者填入 api key」，因此模型欄從姊妹工具的單一文字輸入框改為 `<select id="aiModel">` 列出每家服務商 3-4 個常用模型（含簡短標註如「最強」/「均衡，推薦」/「快速省錢」），選「✏️ 自訂模型名稱…」才顯示 `#aiModelCustom` 文字輸入框。`getResolvedModel()` 統一處理「select 選了 custom 就讀自訂輸入框，否則讀 select 值」的邏輯，`persistApiConfig()`/`callLLM()` 呼叫都改用這個函式取得實際模型字串。`renderModelOptions(provider, presetModel)` 在服務商切換或還原 localStorage 設定時，判斷 `presetModel` 是否命中該服務商的常用清單，命中就選中對應選項、否則自動視為自訂並帶入文字框。**若未來要調整常用模型清單，只需改 `MODEL_OPTIONS` 這個表，不需要動其他邏輯。**

`buildEvaluatePrompt()` 组合貼文原文＋所選平台的 `PLATFORM_RULES`＋選填的目的/受眾＋8個維度中文說明，要求回傳純 JSON（`overallSummary`/`dimensions`/`improvedPost`/`improvementRationale`）。`validateAiEvaluation(parsed, ruleDims)` 是安全網覆核：8個維度 key 逐一檢查 `score` 是否為 0-100 數字（`clampScore()`）、`comment`/`suggestion` 是否為非空字串（截斷至300字），**任一維度缺失或型別錯誤時只有該維度單獨退回 `ruleDims` 對應結果**（不是整批失敗，比照 `SocialPost` 的 `missing[]` 逐項 fallback 機制），並在 `dim-card` 標記 `data-source="rule"` 顯示灰色「規則式」徽章讓使用者知道哪些維度是 AI 評的、哪些是規則式補上的。`improvedPost` 非空字串且不超過3000字才採用，否則整份改用規則式 `ruleImproveMeta.text`；`improvementRationale` 需為字串陣列才採用，否則改用規則式理由。

### 結果渲染（`renderResult()`）

- 總分健檢儀表：SVG 圓弧（`stroke-dasharray`/`stroke-dashoffset` 技巧，`GAUGE_CIRC = 2*Math.PI*52`），中央顯示分數與等第徽章，`transition` 動畫呈現分數變化
- 8 張維度卡片（`.dim-grid`，`repeat(auto-fit,minmax(220px,1fr))`）：圖示＋分數橫條＋一句話說明＋一句建議
- 改善後貼文可編輯 `<textarea>` + 複製按鈕（`navigator.clipboard` 失敗時退回 `document.execCommand('copy')` fallback）+ 改寫理由條列
- 未使用 AI 時顯示 `#warnBanner` 提示「目前為規則式健診」

健診進行中會在 `#inputCard` 加上 `.scanning` class（`::after` 偽元素跑一道由左至右的掃描線動畫），呼應「健診」視覺主題，健診完成後移除。

### 內建範例（`EXAMPLES`）

5 組虛構情境，用途是「示範不同品質程度的貼文原文」而非示範不同產業：`good_story`（寫得好的開幕故事）/`bad_hype`（促銷灌爆文）/`plain_flat`（普通流水帳）/`oversell`（過度促銷硬廣）/`info_type`（資訊科普型）。套用範例前若輸入框已有內容會 `confirm()` 二次確認（比照工作區既有慣例）。

### 草稿持久化

`localStorage` key：`postGraderDraft`（貼文原文/平台/目的/自訂目的/受眾/自訂受眾）、`postGraderApiConfig`（AI設定）、`postGraderMarquee`（跑馬燈快取）。

### 目標受眾（2026-08-27 新增，選填，比照貼文目的的「預設下拉+自訂」模式）

`#audienceSelect`（8 個預設受眾：年輕上班族/學生族群/親子家庭/樂齡銀髮族/精打細算的省錢族/重視品質的中高端客群/科技愛好者／早期採用者/在地熟客／回頭客 + 未指定 + 自訂）＋ `#audienceCustomInput`（選「✏️ 自訂受眾…」才顯示）。`getAudienceText()` 統一讀值（custom 讀自訂輸入框，否則讀 select 值本身，因為 select 的 value 就是顯示用的中文標籤，跟 `PURPOSE_LABELS` 那種另外查表不同）。`applyAudienceValue(text)` 供 `EXAMPLES` 套用範例時使用：文字若命中預設選項就直接選中，命中不到（例如範例裡較長的情境描述）就自動視為自訂並帶入文字框——`EXAMPLES` 資料本身不需要跟著改，範例的 `audience` 欄位仍是自由文字。

## 頂部跑馬燈

比照 `SocialPost`/`coffee-ig-planner` 已驗證的共用實作，串接工作區多個工具共用的同一顆 Google Apps Script 端點（POST 空 `serial`，只取回傳的 `marquee` 陣列，忽略 `valid`/`reason`）。本頁**有 sticky `.topbar`**，所以跑馬燈版面整合比照 `SocialPost` 用 `body.has-marquee .topbar{top:30px}`（不是 `margin-top`），而非 `coffee-ig-planner` 那種無 sticky header 的簡化版本。

## 加入主畫面（PWA）

`manifest.json`＋`service-worker.js`（network-first + 同源快取備援，`{cache:'reload'}` 細節保留）＋`icons/`（PIL＋`msjhbd.ttc` 產生，診斷青色 `#22d3c7` 底＋深色「診」字，192/512/maskable-512/apple-touch-icon 四種尺寸，產生腳本未進 repo，比照工作區慣例）。安裝按鈕沿用 `SocialPost` 已修好 bug 的版本（自帶 `notify()`，不依賴外部 `showToast()`）。

## 視覺主題

「貼文健診」隱喻（醫療診斷儀）：深石墨藍背景（`--bg:#0a141f`）＋診斷青色主色（`--cyan:#22d3c7`，工作區內首次有工具用 teal/cyan 當主識別色，其餘工具僅用作次色）。健檢四色階（A綠/B青/C黃/D紅）沿用工作區既有的紅/黃/綠健檢判色慣例（`restaurant-feasibility-calculator` 的 `healthState()` 邏輯精神），擴展為四段。

## 隱私與警語

無自建後端、無資料上傳到本工具以外的伺服器。AI 金鑰、貼文草稿只存在使用者瀏覽器的 `localStorage`。首頁與手冊皆明列使用警語：健診結果為參考建議不保證正確、請勿輸入真實個資或機密資料、僅供教學與個人使用禁止商業化。

**創作者資訊直接嵌入 index.html（2026-08-27 新增，本工具與姊妹工具的差異點）**：多數姊妹工具（`SocialPost`/`coffee-ig-planner` 等）的完整創作者資料（專長/證照/經歷）只放在 `manual.html`，`index.html` 只有一行 footer credit。本工具在 `index.html` 的 `<footer>` 內、`.warn-box` 之後新增 `.creator-box`；**最初版本曾放入與 `manual.html` 相同的完整內容（專長/證照/經歷），使用者反饋後改為只保留姓名**（`.creator-box` 現在只有標題＋「Mark Tsai（蔡豐全）」一行，對應的 `h4`/`ul` CSS 規則已一併移除，避免留下無用樣式）。完整履歷仍只在 `manual.html`，`index.html` 只做極簡署名。

## 指令

無建置/測試指令。修改 `index.html` 或 `manual.html` 後用瀏覽器開啟驗證，或 `python -m http.server 8799 --directory 行銷內容工具/social-post-grader` 暫起伺服器測完關閉（工作區行銷內容工具資料夾埠號連號 8777/8778/8791/8792/8793/8796/8797/8798 的下一個空號）。

驗證 AI 路徑不需要真實金鑰：可在瀏覽器 console 攔截 `window.fetch` 回傳假的 provider 回應格式，確認 `callLLM → extractJsonObject → validateAiEvaluation → renderResult` 整條管線正確（含單一維度驗證失敗時的單點 fallback），測完記得還原 `window.fetch`。

## 部署

已推公開 GitHub repo：<https://github.com/M255525/social-post-grader>，已用 `.github/workflows/deploy-pages.yml`（Actions 部署模式，`gh api repos/M255525/social-post-grader/pages -f build_type=workflow` 啟用，比照 `workspace-git-repos` 記載的「不要用 legacy branch-source」慣例）啟用 GitHub Pages：<https://m255525.github.io/social-post-grader/>（2026-08-27 上線，已用 Playwright 對正式網址驗證頁面正常渲染）。頁尾已加訪客次數計數器（`visitor-badge.laobi.icu`，`page_id=m255525.socialpostgrader`）。
