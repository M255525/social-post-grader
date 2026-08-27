# 社群貼文健診站

🔗 線上使用：<https://m255525.github.io/social-post-grader/>

貼上一則社群貼文文字，8 個維度（開頭吸引力／核心訊息清晰度／語氣與受眾契合度／CTA明確度／長度與排版易讀性／Hashtag使用／情感共鳴／演算法友善度）逐一評分並附說明與建議，最後產生一份「改善後貼文」。

- 純前端單頁網站，無需安裝、無後端、無序號授權。
- 5 個快速範例按鈕，涵蓋不同品質程度的虛構貼文情境，一鍵套用快速體驗。
- 免 API 金鑰即可使用規則式引擎完成基礎健診（字數／CTA／hashtag／濫用字詞偵測等），不連網。
- 選填 Claude／OpenAI／Gemini／OpenRouter 的 API 金鑰（BYOK，只存瀏覽器 localStorage），可取得更精準的語意層評分與改寫；每家服務商皆列出幾個常用模型可直接選用，也可自訂模型名稱。
- 可依 Facebook／Instagram／X／Threads／通用切換目標平台，套用各自的字數建議、hashtag 數量與演算法規則。
- 可「加入主畫面」安裝成 App；完整操作說明見 [`manual.html`](./manual.html)。

詳見 [`CLAUDE.md`](./CLAUDE.md)。
