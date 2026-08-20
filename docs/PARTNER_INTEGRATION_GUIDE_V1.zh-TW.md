# 合作夥伴整合指南(v1)

本指南適用於在客服 AI 前面整合 Backstop 的 SI、AI 平台公司工程團隊。內容涵蓋整合流程、每種治理決策該怎麼處理,以及出狀況時該預期什麼。

English version: [PARTNER_INTEGRATION_GUIDE_V1.md](./PARTNER_INTEGRATION_GUIDE_V1.md)

完整欄位規格請見 [CONTRACT_V1.zh-TW.md](../CONTRACT_V1.zh-TW.md)。治理決策如何產生,請見 [ARCHITECTURE.zh-TW.md](../ARCHITECTURE.zh-TW.md)。

---

## 為什麼一定要先送出 `ai_draft`

Backstop 的 Evaluate 層——負責抓未授權承諾(退款、折扣、法律保證、政策例外、合約變更)的那一層——檢查的是 AI 的**草稿回覆**,不只是使用者訊息。這是刻意的設計選擇:風險不在使用者說了什麼,而在你的 AI 即將回應時打算承諾什麼。

**對整合的實際影響**:你的系統需要在呼叫 Backstop **之前**,先產生 AI 的草稿回覆,再透過 `POST /api/v1/analyze_dual` 的 `ai_draft` 欄位,把這份草稿跟使用者訊息一起送過來。沒有 `ai_draft`,未授權承諾風險偵測就不會對任何 AI 回覆內容生效——只有 Detect 路徑(針對使用者訊息的語氣/危機訊號)會運作。

如果你目前的整合方式,是把使用者的訊息送進你的 AI 系統、直接把結果回傳給終端使用者,中間沒有經過治理檢查,你需要加入一個步驟:**先產生草稿 → 帶著草稿呼叫 Backstop → 依決策採取行動 → 再決定要不要送出這則回覆。**

---

## Shadow Mode:兩種控制機制

Shadow Mode 有兩種啟用方式,可以獨立使用,也可以同時並用:

| 機制 | 作用範圍 | 怎麼設定 | 適合的情境 |
|---|---|---|---|
| **Header Shadow** | 單次請求層級 | 單一 API 呼叫帶 `X-Governance-Mode: Shadow` | PoC、漸進式上線、A/B 測試,或是不改後端設定、只想觀察特定一批流量 |
| **Profile Shadow** | 租戶/設定檔層級預設值 | 在後端透過租戶的政策設定檔設定(`policy_profiles/{profile_id}/shadow_mode`) | 正式的觀察期,整個租戶預設都要跑在 Shadow Mode 下 |

兩者的生效邏輯是:**`shadow_mode_active = header_shadow OR profile_shadow`**——只要任一為真,這次呼叫就會跑在 Shadow Mode。

**這點對「正式上線」很關鍵**:只是把請求裡的 `X-Governance-Mode: Shadow` 拿掉,並不足以切換到正式執行模式。你還需要確認後端該租戶的 Profile 層級 shadow 設定也已經關閉——否則就算拿掉了標頭,整合仍然會持續跑在「只觀察」的狀態。

---

## 上線流程

我們建議分三階段推進,不要直接一次切換:

### 階段一——先觀察,不執行(Shadow Mode)
在每次 API 呼叫都帶上 `X-Governance-Mode: Shadow`(或請我們幫你在後端把該租戶的 Profile Shadow 打開)。在這個模式下,Backstop 永遠不會攔截或改變你的 AI 送出的內容——`final_decision_state` 永遠是 `ALLOW`。你收集的只是 `observed_final_decision_state` 這個資料,用來了解「如果真的執行治理,原本會怎麼判斷」。

這讓你可以先驗證整合本身有沒有做對(`ai_draft` 有沒有正確傳送?`tenant_id` 有沒有正確解析?),完全不會影響正式環境的流量。

### 階段二——審閱
觀察一段時間之後(我們建議至少 14 天——Shadow Mode 的時間軸請見 [README.zh-TW.md](../README.zh-TW.md)),從 `GET /api/v1/reports/risk_14d` 拉一份報告,跟你的團隊、以及(如果適用)終端客戶的法務/風控相關人員一起審閱這些觀察到的決策。這是在真正開始執行治理之前,確認政策嚴格程度符合預期的時機點。

### 階段三——正式執行
把請求裡的 `X-Governance-Mode: Shadow` 拿掉,**同時**確認該租戶在後端的 Profile Shadow 已經關閉——這兩個條件都要滿足,`final_decision_state` 才會真正反映實際執行結果。到了這個階段,`GUIDE` 跟 `BLOCK` 的決策會真的影響到終端使用者收到的內容——切換前,務必確認你的整合能正確處理這三種狀態(見下方)。

---

## 處理每一種決策狀態

你的整合系統要負責根據 `final_decision_state` 採取行動——Backstop 只告訴你該怎麼做,實際執行要靠你的系統。

| 狀態 | Backstop 回傳什麼 | 你的系統該做什麼 |
|---|---|---|
| `ALLOW` | 草稿回覆原封不動通過 | 把 AI 的草稿回覆原樣送給使用者 |
| `GUIDE` | 標記出的風險,加上治理輔助欄位——`assistant_instruction`(正式執行模式下)、`draft_reference`、和/或 `shadow_explainability` 觀察摘要——包含建議因應策略跟建議措辭 | **不要**送出原始草稿。用回傳的引導資訊重新產生一份合規的回覆,或替換成安全的保底訊息,再送給使用者 |
| `BLOCK` | 標記出需要人工審閱的風險,可能包含 `safe_message` | **在任何情況下都不要**送出原始草稿。把對話轉交給真人客服 |

**重要提醒**:Backstop 回傳的是治理決策跟輔助欄位(`assistant_instruction`、`draft_reference`、`safe_message`,視情況而定)——但它**不會**自動幫你重新產生、替換、或代為送出修正後的回覆。整合方要負責最後一步:用這些引導資訊產生合規的回覆,或是轉交人工。

`assistant_instruction` 跟相關的正式執行模式欄位,只有在請求跑在正式執行模式(非 Shadow)時才會有值;在 Shadow Mode 下,對應的資訊會以「觀察摘要」的形式呈現(`shadow_explainability`),而不是即時的修正指令。`assistant_instruction` 是一個結構化物件(包含目標、禁止出現的承諾類型、必要措辭),設計目的是給你的回覆產生邏輯參考使用——它明確標註為「僅供參考」,絕對不能原封不動送給終端使用者。完整欄位規格請見 [CONTRACT_V1.zh-TW.md](../CONTRACT_V1.zh-TW.md)。

---

## 租戶設定

每個整合都在後端綁定一組 `tenant_id`,對應到一份政策設定檔。除了你的 `tenant_id` 已經設定好的內容之外,沒有任何客戶端機制可以在請求時選擇或切換政策設定檔——這是刻意的設計,避免政策嚴格程度被呼叫端不小心(或惡意)覆蓋掉。

要申請 `tenant_id`,以及(如果需要的話)`X-API-Key`,請直接與我們聯繫(聯絡方式見 [README.zh-TW.md](../README.zh-TW.md))。傳入無法辨識的 `tenant_id` 會被 `403` 拒絕,不會默默退回用預設政策處理。

---

## 失敗時的處理方式

如果呼叫 Backstop 失敗(逾時、5xx 錯誤、網路問題),**不要因此直接放行 AI 未經審查的原始草稿。** 這不是 API 會自動幫你做的事——這是一個 fail-closed(失敗時保守處理)的模式,需要你的整合系統自己實作:

1. 替換成保守的、事先核准過的安全訊息(例如「讓我幫您轉接專員協助處理這個問題」),並且
2. 把對話轉交給真人客服。

把治理檢查失敗,當成跟 `BLOCK` 一樣處理——沒有得到明確的「可以」,就應該預設保守處理,而不是放行一則未經審查的 AI 回覆。

---

## 整合前必知事項

- **不要在請求內容裡帶 `policy_profile`。** API 會拒絕未定義的欄位(`422`)——政策設定檔是綁定在後端的 `tenant_id` 上,不是每次請求各自傳入的。
- **`422` 不是只跟字數有關。** 也包含請求內容出現未定義的欄位、enum 值不合法、或型別不符——不只是 `user_text`/`text` 超出 5–500 字元範圍。
- **`403` 不是只有「未知租戶」這一種情況。** 也包含無效的 API 金鑰、或金鑰跟請求的租戶對不上。
- **`risk_14d` 跟 `shadow_observations` 接受的參數不一樣。** `risk_14d` 接受 `tenant_id`、`policy_profile`、`shadow_mode_only`、`limit`——**不**接受 `days`(報告區間固定為 14 天)。`shadow_observations` 才有 `days` 這個參數。完整參數規格請見 [CONTRACT_V1.zh-TW.md](../CONTRACT_V1.zh-TW.md)。
- **`observed_*` 欄位主要在 Shadow Mode 下才有意義。** 在 Shadow Mode 之外,這些觀察欄位通常會是 `null`——不要把整合邏輯建立在「正式執行模式下這些欄位一定有值」這個假設上。
- **把這條規則寫死進你的整合系統**:只有 `ALLOW` 可以原封不動送給終端使用者。`GUIDE` 跟 `BLOCK` 絕對不能讓原始草稿送達使用者。

---

## 上線前檢查清單

- [ ] 草稿回覆的產生,發生在呼叫 Backstop **之前**,不是之後
- [ ] 每一次 `analyze_dual` 呼叫都有帶 `ai_draft`(不是只有 `user_text`)
- [ ] `tenant_id` 設定正確,不會回傳 `403`
- [ ] 整合已經在 Shadow Mode(Header 或 Profile 皆可)下跑過至少 14 天
- [ ] `GUIDE` 跟 `BLOCK` 的回應有分別處理——兩者都不會讓原始草稿送達使用者
- [ ] 已經在你自己的系統裡實作好 API 失敗時的保守保底路徑
- [ ] 團隊裡至少有人在正式啟用治理前,審閱過至少一份 `risk_14d` 報告
- [ ] 切換到正式執行模式前,已確認**同時**做到:請求裡的 `X-Governance-Mode: Shadow` 已移除,**且**後端該租戶的 Profile Shadow 已關閉
