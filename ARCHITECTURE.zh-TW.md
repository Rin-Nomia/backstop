# 架構說明

Backstop 是一層「執行期治理閘道(Runtime Enforcement Gateway)」,運作在客服 AI 跟真正的使用者之間。它不會修改、重新訓練、或替換底層的 LLM——它做的是即時審查進出這個模型的內容。

English version: [ARCHITECTURE.md](./ARCHITECTURE.md)

---

## 設計原則

1. **不侵入式。** Backstop 透過 API 外掛接入,不需要更動現有的模型、Prompt 鏈路或基礎設施。

2. **目前是規則式比對,不是語意理解——這是刻意的設計選擇。** 現在的偵測方式,靠的是規則比對(正則表達式 + 關鍵字規則),不是用 LLM 去理解語意。這是一個刻意的取捨:規則比對速度快(見下方「效能」段落)、大量運算時成本低,而且**結果完全可預測**——同樣的輸入,永遠得到同樣的判斷結果,這一點對「可稽核性」非常重要。代價是覆蓋率:還沒被收錄進規則庫的新講法,可能會被漏掉。目前已知的漏洞跟涵蓋範圍,記錄在 [REGRESSION_CHECKLIST_P1.md](./docs/REGRESSION_CHECKLIST_P1.md)。

3. **無法判斷時,保守處理。** 當一個請求沒辦法有信心地判斷時,系統會導向人工轉接,而不是默默放行。

4. **租戶互相隔離。** 每個客戶(租戶)都有自己獨立的政策設定、事件記錄路徑、跟嚴格程度設定。一家客戶的資料跟規則,不會外洩到另一家客戶身上。如果收到一個沒有登記過的 `tenant_id`,系統會直接拒絕(回傳 403 錯誤),不會默默用預設值處理。

5. **先用 Shadow Mode 上線。** 治理功能可以先用「只觀察、不攔截」的模式打開,在真正開始攔截任何正式回覆之前,先確認不會影響正式環境,讓導入這件事零風險。

---

## 四層模型

*(架構圖與 English 版本共用,見 [ARCHITECTURE.md](./ARCHITECTURE.md) 內的 Mermaid 流程圖)*

流程概念:使用者訊息進到 **Detect**,AI 草稿回覆進到 **Evaluate**,兩者的結果一起匯入 **Control** 做出最終決策,每一次決策都會被 **Audit** 記錄下來。

### 1. Detect(偵測)
分類「使用者」訊息屬於六種高風險互動狀態之一(例如高壓施壓、情緒崩潰、模糊不安、危機訊號)。這一層運作在「AI 都還沒開始草擬回覆之前」的輸入端,目前的用途是當作路由/情境判斷的參考訊號,不是獨立的攔截觸發點。

### 2. Evaluate(評估)
掃描「AI 的草稿回覆」(`ai_draft`),偵測六大類未授權承諾:退款、折扣、賠償、法律保證、政策例外、合約變更。**這是商業風險最高的一層**——它是擋在「AI 代理」跟「企業從未授權的承諾」之間的那道牆。呼叫時必須同時傳入 `ai_draft` 跟 `user_text`;如果沒有提供草稿,這一層就沒有東西可以評估。

### 3. Control(控制)
把 Detect 跟 Evaluate 的結果,收斂成三種治理模式之一:

| 模式 | 行為 |
|---|---|
| 🟢 **Sense**(`ALLOW`) | 放行,並記錄存檔 |
| 🟡 **Guide**(`GUIDE`) | 送出前先限制/修正回覆內容 |
| 🔴 **Block**(`BLOCK`) | 攔截回覆,轉交人工處理 |

在 **Shadow Mode**(請求標頭帶 `X-Governance-Mode: Shadow`)下,Control 層對「外部實際生效」的決策**永遠會解析成 `ALLOW`**——不會真的攔截任何東西——但同時仍會記錄「原本應該會是什麼決策」(`observed_final_decision_state`),供之後審閱。

### 4. Audit(稽核)
每一次被評估過的互動,都會產生一筆不可竄改的記錄:觸發來源(來自使用者 / AI 草稿 / 兩者皆有)、風險標籤、原因代碼、決策結果、跟 UTC 時間戳記。這一層,就是把「我們有治理機制」這句話,變成「我們能證明當時發生了什麼、什麼時候發生的」的關鍵。

### 概念層與程式碼模組的對應關係
本文件使用的 Detect / Evaluate / Control / Audit,是給讀者理解用的**概念層命名**,不是程式碼裡實際的模組或檔案名稱。以下對應關係已由開發端確認為目前最新版本:

| 概念層 | 對應實作模組 | 備註 |
|---|---|---|
| Detect(危機訊號部分) | `core/safety_gate.py` | 透過 `check_out_of_scope(...)` 做危機/越界偵測 |
| Detect(語氣分類部分) | `core/classifier.py` | 分類分數會結合 `core/rhythm_analyzer.py` 的語氣節奏訊號 |
| Evaluate | `core/commitment_guard.py` + `configs/commitment_rules.yaml` | 規則群組依承諾類型命名(如 `refund_commitment`、`compensation_commitment`) |
| Control(`analyze_dual`) | `core/dual_orchestrator.py` | 雙路徑(使用者訊息 + AI 草稿)決策整合 |
| Control(`analyze`,單路徑) | `pipeline/z1_pipeline.py` + `core/router.py` | |

這是概念層跟實作層命名不同的正常現象,不代表文件與程式碼不一致。本文件不揭露正則表達式/關鍵字清單、信心門檻值、白名單邏輯、或反規避規則,這些內容保留在內部。

---

## 可解釋性(Explainability)

當單次呼叫 `/api/v1/analyze` 或 `/api/v1/analyze_dual` 的判斷結果落在 `GUIDE`/`BLOCK` 狀態時,**API 回應本身就會即時附上**一個 `shadow_explainability` 欄位(`mode: "summary_only"`),內容包含風險類型、原因代碼、建議的因應策略、建議使用的措辭——不需要等到 Day 14 報告才看得到。同樣的摘要內容,也會出現在 `/api/v1/metrics/shadow_observations` 與 `/api/v1/reports/risk_14d` 這兩個報告端點的對應欄位(`observed_guidance_summary`)裡,供之後回顧查詢。

**這份摘要刻意是經過篩選的版本**:用來組成 AI 引導建議的完整內部指令集(`assistant_instruction`),只保留在系統內部,對外的公開回應中此欄位固定回傳 `null`——即使在 Shadow Mode 下也一樣。

---

## 效能

以下數字來自正式環境的實測(雲端到雲端,測試日期 2026-07-31):

| 情境 | P95 延遲 |
|---|---|
| 單路徑 `/analyze`(序列請求) | 91ms |
| 雙路徑 `/analyze_dual`(序列,依情境分) | 119–134ms |
| 雙路徑 `/analyze_dual`(5 併發) | 350ms |

5 併發情境下的吞吐量約為每秒 19.9 次請求,所有測試情境的錯誤率皆為 0%。這些數字不含用戶端的網路波動,且是單日測試結果,不是長期穩定性保證。

---

## 多租戶架構

每個租戶都會在後端綁定一組 `tenant_id` 對應到政策設定檔。每個租戶的:

- 事件記錄路徑、
- 政策嚴格程度(例如同一類風險,對 A 客戶可以設成 GUIDE,對 B 客戶設成 BLOCK)、
- 以及 Shadow Mode 的開關狀態,

都是各自獨立設定的。跨租戶的資料外洩,會在路由層就被擋下——一個無法辨識的 `tenant_id`,系統會直接失敗拒絕,而不是退回用預設值繼續處理。

---

## 公開了什麼,沒公開什麼

這份文件描述的是系統的整體樣貌——資料怎麼流動、每一層負責什麼、要怎麼跟它對接。**不包含**:

- 底層的規則/樣式定義本身、
- 政策設定檔的實際內容、
- 任何特定租戶的資料。

資料格式與請求/回應合約,請見 [CONTRACT_V1.md](./CONTRACT_V1.md);整合步驟請見 [PARTNER_INTEGRATION_GUIDE.md](./docs/PARTNER_INTEGRATION_GUIDE.md)。
