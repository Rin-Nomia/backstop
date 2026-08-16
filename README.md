# Backstop

**Runtime Governance for Enterprise AI — 不替換你的 AI,治理它。**

專為把 Agentic AI 交付給金融、醫療等高合規產業的 SI(系統整合商)與 AI 平台公司設計——解決「AI 做出未授權承諾,導致專案卡在法務/風控驗收關卡」的問題。

---

## 這是什麼

Backstop 是一層獨立於 LLM 底層之外的 Runtime Enforcement Gateway(執行期治理層),在 AI 客服的回覆送出之前,多做一層審查,偵測並攔截 AI 做出的**未授權承諾**——例如退款、折扣、法律保證、政策例外。

不替換你現有的 AI 系統,不改變模型行為,以外掛的方式接入。

---

## 解決什麼問題

企業導入 AI 客服之後,最貴的風險不是 AI 說錯話,而是 **AI 說了一句沒有人授權它說的話**。

退款、折扣、法律保證、政策例外——AI 在高壓對話裡很容易做出企業沒有授權的承諾。說出去之後,沒有記錄,沒有人知道為什麼。

這不是個案。根據 Cloud Security Alliance 2026 年 4 月發布的調查 [^1],53% 的組織已經遇過 AI Agent 超出預期權限的情況。OWASP 將這個現象正式定義為 **Excessive Agency**,是當前 LLM 部署的主要風險類別之一。

**對於把 AI 交付給金融、醫療等高合規產業客戶的 SI 或 AI 平台公司來說,這個問題更直接:每接一個新客戶,就要重寫一次風控規則;出事之後,沒有記錄可以說清楚當時發生了什麼——這正是專案卡在客戶法務/風控驗收關卡、尾款卡住的主因。**

[^1]: Cloud Security Alliance, *Enterprise AI Security Starts with AI Agents*(commissioned by Zenity,2026 年 4 月 16 日發布,調查執行於 2025 年 9-11 月,445 位 IT/資安專業人士受訪)。[官方新聞稿](https://cloudsecurityalliance.org/press-releases/2026/04/16/more-than-half-of-organizations-experience-ai-agent-scope-violations-cloud-security-alliance-study-finds)

---

## 立即試用

不用讀文件、不用寫程式,直接在瀏覽器體驗:

- 🎮 **Live Playground**:[rinnomia-continuum-api.hf.space/playground](https://rinnomia-continuum-api.hf.space/playground) — 直接輸入對話,即時看到 Evaluate/Audit 判斷結果與 Before/After 對照

---

## 核心架構:四層模型

Backstop 由四個層次組成,各司其職:

| 層 | 名稱 | 功能 |
|---|---|---|
| 1 | **Detect** | 辨識使用者的高風險互動狀態(例如高壓施壓、情緒崩潰、危機訊號) |
| 2 | **Evaluate** | 在 AI 回覆送出前,判斷草稿是否包含未授權承諾(退款、折扣、賠償、法律保證、政策例外、合約變更)。目前為**規則式比對(pattern-based),持續擴充中**,尚非語意理解;已知限制與測試覆蓋範圍見 [REGRESSION_CHECKLIST](./docs/REGRESSION_CHECKLIST_P1.md) |
| 3 | **Control** | 即時介入,分三種等級:放行並記錄 / 引導修正回覆 / 攔截並轉交人工 |
| 4 | **Audit** | 每一個高風險時刻都留下完整記錄:觸發原因、風險類型、治理決策、時間戳記 |

**效能實測(Production 環境,2026-07-31)**:5 併發情境下,P95 350ms 內完成風險判斷;單一請求 P95 91ms。雲端到雲端實測,詳見 [ARCHITECTURE.md](./ARCHITECTURE.md)。

詳細架構說明請見 [ARCHITECTURE.md](./ARCHITECTURE.md)。

---

## Quick Start(給工程師)

📖 完整 API 文件(Swagger UI):[rinnomia-continuum-api.hf.space/docs](https://rinnomia-continuum-api.hf.space/docs)

直接呼叫範例:

```bash
curl -X POST https://rinnomia-continuum-api.hf.space/api/v1/analyze_dual \
  -H "Content-Type: application/json" \
  -H "X-Governance-Mode: Shadow" \
  -d '{
    "user_text": "你們再不處理我就投訴媒體。",
    "ai_draft": "我可以幫你退款,這筆我現在直接處理。"
  }'
```

回傳結果會包含判斷決策(ALLOW/GUIDE/BLOCK)、風險類型、原因代碼與完整稽核紀錄。完整欄位說明見 [CONTRACT_V1.md](./CONTRACT_V1.md)。

---

## 試用方式:Shadow Mode(零風險觀測)

不確定要不要正式導入?可以先用零風險的方式觀察:

- **Day 1**:API 靜默掛載,不攔截任何回覆,零業務風險
- **Day 7**:自動記錄高風險對話,觸發原因與風險類型全程留存
- **Day 14**:自動產出第一份 AI Risk Report,可直接交給法務、風控審閱

---

## 目前狀態

Backstop 目前是持續迭代中的產品,規則庫涵蓋退款、折扣、賠償、法律保證、政策例外、合約變更六大類未授權承諾類型,建構基礎來自 30 個跨產業 AI 事故案例研究。詳細的已知限制與測試結果,請見 [REGRESSION_CHECKLIST.md](./docs/REGRESSION_CHECKLIST_P1.md)。

---

## 聯絡方式

Shih Huan Chen
Founder, Backstop | AI Governance Systems
Email: rin.nomia.series@gmail.com
LinkedIn: [linkedin.com/in/shih-huan-chen-7998503aa](https://www.linkedin.com/in/shih-huan-chen-7998503aa/)

---

🇺🇸 [English version](./README.md)
