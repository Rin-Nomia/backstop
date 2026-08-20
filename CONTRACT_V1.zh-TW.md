# 資料合約(v1)

本文件說明 Backstop 公開 API 的請求/回應格式——也就是你串接時會看到的欄位,不是底層規則的實作邏輯。完整互動式規格請見 [Swagger UI](https://rinnomia-continuum-api.hf.space/docs) 或 [openapi.json](https://rinnomia-continuum-api.hf.space/openapi.json)。

English version: [CONTRACT_V1.md](./CONTRACT_V1.md)

---

## 基礎網址

```
https://rinnomia-continuum-api.hf.space
```

---

## 身份驗證

設定為 `requires_api_key: true` 的租戶,必須透過 `X-API-Key` 標頭傳入自己的金鑰。公開/展示用端點不需要金鑰。

---

## 端點

| 方法 | 路徑 | 用途 |
|---|---|---|
| `GET` | `/health` | 存活檢查 |
| `POST` | `/api/v1/analyze` | 單路徑評估(僅 `text`) |
| `POST` | `/api/v1/analyze_dual` | 雙路徑評估(`user_text` + `ai_draft`) |
| `GET` | `/api/v1/metrics/kpi` | 彙總商業 KPI |
| `GET` | `/api/v1/metrics/casebook` | 案例集式證據紀錄 |
| `GET` | `/api/v1/metrics/shadow_observations` | Shadow Mode 觀察報告,可依租戶篩選 |
| `GET` | `/api/v1/reports/risk_14d` | 14 天 AI 風險報告,可依租戶篩選 |

---

## 標頭

| 標頭 | 值 | 用途 |
|---|---|---|
| `Content-Type` | `application/json` | 所有 `POST` 請求都必須帶 |
| `X-Governance-Mode` | `Shadow`(不帶則為正式執行模式) | 設為 `Shadow` 時,Control 層對外實際生效的決策永遠會解析成 `ALLOW`;原本應該會是什麼決策,仍會被記錄下來 |
| `X-API-Key` | 租戶專屬金鑰 | 僅設定為 `requires_api_key: true` 的租戶需要 |

---

## 請求:`POST /api/v1/analyze_dual`

```json
{
  "user_text": "你們再不處理我就投訴媒體。",
  "ai_draft": "我可以幫你退款,這筆我現在直接處理。",
  "tenant_id": "tenant-a"
}
```

| 欄位 | 型別 | 是否必填 | 說明 |
|---|---|---|---|
| `user_text` | string | 是 | 5–500 個字元。使用者的訊息內容。 |
| `ai_draft` | string | 選填 | AI 打算送出的草稿回覆。Evaluate 層要有東西可以檢查,就需要這個欄位——沒有的話,只有 Detect 層會針對 `user_text` 運作。 |
| `tenant_id` | string | 選填 | 未提供時預設為 `default`。傳入無法辨識的 `tenant_id` 會回傳 `403`。 |

請求內容只接受上述欄位——出現未定義的欄位會被拒絕(`422`)。

### `POST /api/v1/analyze` 用的是另一種、更簡單的格式

```json
{
  "text": "你們再不處理我就投訴媒體。",
  "source": "user",
  "tenant_id": "tenant-a"
}
```

| 欄位 | 型別 | 是否必填 | 說明 |
|---|---|---|---|
| `text` | string | 是 | 5–500 個字元。 |
| `source` | string | 選填 | `"user"` 或 `"ai_draft"`——代表 `text` 是對話中的哪一方所說的。 |
| `tenant_id` | string | 選填 | 行為與上方相同。 |

回應格式與 `POST /api/v1/analyze_dual`(見下方)相同,差別在於 `audit_summary.trigger` 會固定對應你傳入的 `source` 參數值,而不是像雙路徑那樣依「同時有 `user_text` 跟 `ai_draft`」去推斷。

---

## 回應:`POST /api/v1/analyze_dual`

```json
{
  "shadow_mode_active": true,
  "final_decision_state": "ALLOW",
  "observed_final_decision_state": "GUIDE",
  "observed_final_intervention_reason_code": "unauthorized_refund_commitment",
  "observed_final_risk_label": "Unauthorized commitment risk",
  "user_analysis": {
    "audit": {
      "pipeline_version_fingerprint": "88ff8c4d36793534467e58ab019a3b4d993674f205f1bffecbb894a17de7c32d"
    }
  },
  "ai_draft_analysis": {
    "audit": {
      "pipeline_version_fingerprint": "88ff8c4d36793534467e58ab019a3b4d993674f205f1bffecbb894a17de7c32d"
    }
  },
  "audit_summary": {
    "trigger": "ai_draft",
    "evidence": {},
    "risk": "Unauthorized commitment risk",
    "timestamp": "2026-08-02T14:37:17.850638+00:00",
    "reason_code": "unauthorized_refund_commitment",
    "shadow_explainability": {
      "risk_type": "Unauthorized commitment risk",
      "reason_code": "unauthorized_refund_commitment",
      "recommended_strategy": "Avoid direct refund promises. Guide user to official refund workflow and approval path.",
      "required_phrases": [
        "I will help you follow the official process.",
        "If needed, I can connect you with a specialist."
      ],
      "mode": "summary_only",
      "full_instruction_disclosure": "internal_only"
    }
  }
}
```

| 欄位 | 型別 | 說明 |
|---|---|---|
| `final_decision_state` | `"ALLOW"` \| `"GUIDE"` \| `"BLOCK"` | 這次回應實際生效的決策。當請求帶 `X-Governance-Mode: Shadow` 時,永遠是 `"ALLOW"`。 |
| `observed_final_decision_state` | `"ALLOW"` \| `"GUIDE"` \| `"BLOCK"` \| `null` | 如果真的執行治理,原本會是什麼決策。Shadow Mode 下會有值;非 Shadow Mode 通常是 `null`。 |
| `observed_final_intervention_reason_code` | string \| `null` | 機器可讀的原因代碼(見下方〈原因代碼〉)。沒有偵測到風險時為 `null`。 |
| `observed_final_risk_label` | string | 人類可讀的風險分類。 |
| `user_analysis.audit.pipeline_version_fingerprint` | string | 一組雜湊值,用來標識目前處理這次請求的規則庫跟語氣分類邏輯是哪個版本。每一次回應(不管 Shadow 或正式執行模式)都會有這個欄位。可以用它來確認你現在測的是哪個部署版本——詳見 [REGRESSION_CHECKLIST_P1.zh-TW.md](./docs/REGRESSION_CHECKLIST_P1.zh-TW.md) 裡怎麼用這個欄位驗證測試結果。 |
| `ai_draft_analysis.audit.pipeline_version_fingerprint` | string | 跟上面相同,但對應的是 `ai_draft` 那個路徑的分析結果。只有請求裡有帶 `ai_draft` 時才會出現(也就是 `analyze_dual` 呼叫,或是 `analyze` 呼叫且 `source: "ai_draft"` 的情況)。 |

> **為什麼會有兩個「audit」命名空間?** `audit_summary` 是這次請求整體的**決策層級治理記錄**。`user_analysis.audit`(以及雙路徑端點裡的 `ai_draft_analysis.audit`)則是**單一路徑的執行層級溯源資訊**——例如跑的是哪個版本的處理流程、快取/保底機制的行為、耗時等執行層級的中繼資料。`pipeline_version_fingerprint` 屬於「溯源資訊」,不屬於「決策摘要」,這就是為什麼它放在 `*_analysis.audit` 這個命名空間下,而不是放進 `audit_summary` 裡。
>
> **穩定性提醒**:`user_analysis.audit` / `ai_draft_analysis.audit` 這兩個命名空間本身是穩定的,但底下的物件結構是開放式的——未來可能會新增或調整欄位。`pipeline_version_fingerprint` 這個欄位本身是穩定、建議使用的,但整合時請用容錯的方式解析這個物件(不要假設欄位清單是固定不變的,遇到不熟悉的欄位或某個非必要欄位缺席時,不要讓整合流程因此失敗)。
| `audit_summary` | object | Audit 層保留的記錄,詳見下方。 |
| `audit_summary.trigger` | string | `"user"` / `"ai_draft"` / `"both"`——對話中是哪一方觸發了這次評估。 |
| `audit_summary.evidence` | object | 結構化的、依來源而異的判斷依據。內部結構不在本公開合約揭露範圍內。 |
| `audit_summary.risk` | string | 與 `observed_final_risk_label` 相同。 |
| `audit_summary.timestamp` | string(ISO 8601,UTC) | 決策發生的時間。 |
| `audit_summary.reason_code` | string \| `null` | 與 `observed_final_intervention_reason_code` 相同。 |
| `audit_summary.shadow_explainability` | object | 當決策為 `GUIDE`/`BLOCK` 時才會出現。一份經過篩選、對外可見的摘要——風險類型、原因代碼、建議因應策略、建議使用措辭。`mode` 固定為 `"summary_only"`;完整的內部修正指令不會透過這個欄位對外揭露。 |

---

## 回應:正式執行模式(非 Shadow)

當請求以正式執行模式運作(未帶 `X-Governance-Mode: Shadow` 標頭,且該租戶的後端 Profile Shadow 已關閉——詳見 [PARTNER_INTEGRATION_GUIDE_V1.md](./docs/PARTNER_INTEGRATION_GUIDE_V1.md)),且決策為 `GUIDE` 或 `BLOCK` 時,回應會包含上方 Shadow Mode 範例裡沒有的治理輔助欄位。

真實範例(`GUIDE`,正式執行模式):

請求:
```json
{
  "tenant_id": "tenant-a",
  "user_text": "Please check my order status.",
  "ai_draft": "I can refund you immediately."
}
```

回應(本段相關欄位):
```json
{
  "shadow_mode_active": false,
  "final_decision_state": "GUIDE",
  "observed_final_decision_state": null,
  "user_analysis": {
    "audit": {
      "pipeline_version_fingerprint": "88ff8c4d36793534467e58ab019a3b4d993674f205f1bffecbb894a17de7c32d"
    }
  },
  "assistant_instruction": {
    "objective": "Provide an empathetic and actionable response without unauthorized commitments.",
    "forbidden_commitments": [
      "unauthorized_refund_commitment",
      "unauthorized_discount_commitment",
      "unauthorized_compensation_commitment",
      "unauthorized_legal_guarantee",
      "unauthorized_policy_override",
      "unauthorized_contract_change",
      "mandatory_human_handoff"
    ],
    "must_include_phrases": [
      "I will help you follow the official process.",
      "If needed, I can connect you with a specialist."
    ],
    "focus_reason_codes": ["unauthorized_refund_commitment"],
    "delivery_mode": "reference_only",
    "do_not_use_as_final_reply": true
  },
  "draft_reference": "This request may involve a commitment beyond assistant authority. Please follow official policy and authorized review flow.",
  "safe_message": null,
  "audit_summary": {
    "trigger": "ai_draft",
    "risk": "Unauthorized commitment risk",
    "timestamp": "2026-08-20T08:08:37.474422+00:00",
    "reason_code": "unauthorized_refund_commitment"
  }
}
```

| 欄位 | 型別 | 說明 |
|---|---|---|
| `assistant_instruction` | object \| `null` | 正式執行模式下,`GUIDE`/`BLOCK` 時會有值。是一份結構化的修正指引——目標、禁止出現的承諾類型、必要措辭、對應的原因代碼。`delivery_mode: "reference_only"` 跟 `do_not_use_as_final_reply: true` 一定會出現:**這個物件是用來輔助重新產生回覆的參考資料,絕對不是可以直接送出的回覆本身。** Shadow Mode 下為 `null`(改看 `audit_summary.shadow_explainability`)。 |
| `draft_reference` | string \| `null` | 用於重新產生回覆的參考文字。這段文字通常是經過治理處理過的改寫版本,不一定會等同於你原本傳入的 `ai_draft`——不要假設兩者相同。 |
| `safe_message` | string \| `null` | 正式執行模式下的 `BLOCK` 決策會有值(系統層級有預設的保底訊息)。`GUIDE`/`ALLOW` 時通常是 `null`,Shadow Mode 的觀察結果裡也一律是 `null`。 |

---

## 原因代碼

原因代碼依 Evaluate 層檢查的六大類未授權承諾分組:

| 類別 | 原因代碼 |
|---|---|
| 退款 | `unauthorized_refund_commitment` |
| 折扣 | `unauthorized_discount_commitment` |
| 賠償 | `unauthorized_compensation_commitment` |
| 法律保證 | `unauthorized_legal_guarantee` |
| 政策例外 | `unauthorized_policy_override` |
| 合約變更 | `unauthorized_contract_change` |

另外還有一批用於路由、語氣、安全訊號的操作性原因代碼(不對應特定承諾類別),包括但不限於:`mandatory_human_handoff`、`crisis_self_harm`、`escalation_pressure`、`ambiguous_commitment_risk`、`pressure_induced_commitment`、`TONE_PUSHY`、`TONE_ANXIOUS`、`TONE_COLD`、`TONE_SHARP`、`TONE_BLUR`、`TONE_UNKNOWN`、`TONE_RHYTHM`、`OOS_CRISIS`,以及一個備援用的 `intervention_unknown`。

這份清單記錄的是原因代碼的**呈現形式**,不是目前完整的代碼集合——底層真正產生這些代碼的規則定義不公開。這些代碼如何對應到各治理層,請見 [ARCHITECTURE.md](./ARCHITECTURE.zh-TW.md)。

**命名風格說明**:承諾類代碼(`unauthorized_*`)使用小寫底線(`snake_case`),語氣/危機訊號類代碼(`TONE_*`、`OOS_CRISIS`)使用全大寫。這反映的是兩者分別來自不同的實作模組(Evaluate 層的 `commitment_guard`,對比 Detect 層的 `classifier`/`safety_gate`——詳見 [ARCHITECTURE.md](./ARCHITECTURE.zh-TW.md) 裡的模組對照表),不是 API 設計上的不一致。

---

## 錯誤

| 狀態碼 | 意義 |
|---|---|
| `403` | 在租戶層被拒絕——無法辨識的 `tenant_id`、無效的 API 金鑰、或金鑰跟請求的租戶對不上。系統不會默默退回用預設租戶處理。 |
| `422` | 驗證錯誤——`user_text`/`text` 超出 5–500 字元範圍、請求內容出現未定義的欄位、enum 值不合法、或型別不符。 |

---

## 報告端點

### `GET /api/v1/metrics/shadow_observations`

查詢參數:`tenant_id`、`policy_profile`、`days`、`limit`。

### `GET /api/v1/reports/risk_14d`

查詢參數:`tenant_id`、`policy_profile`、`shadow_mode_only`、`limit`。報告區間固定為 14 天——這個端點不接受 `days` 參數。

```
GET /api/v1/reports/risk_14d?tenant_id=tenant-a&shadow_mode_only=true&limit=10
```

*以下範例是「零資料初始狀態」的示範(一個剛上線、還沒發生任何事件的租戶)——這裡的 `deployment_confidence_score` 反映的是「觀察期間內沒發生任何事」,不是套用某個公式對 `prevented_risk_count` 計算出來的分數。*

```json
{
  "ok": true,
  "report_version": "v1",
  "window_days": 14,
  "generated_at_utc": "2026-08-02T14:37:21.546867+00:00",
  "applied_filters": {
    "tenant_id": "tenant-a",
    "policy_profile": "default",
    "shadow_mode_only": true
  },
  "summary": {
    "total_conversations": 10,
    "prevented_risk_count": 0,
    "handoff_rate": 0.0,
    "exceptions_per_1k_conversations": 0.0,
    "deployment_confidence_score": 100.0,
    "policy_reuse_ratio": 0.0,
    "median_hours_to_policy_onboard": 0.0
  },
  "cases": [],
  "shadow_explainability_cases": []
}
```

---

## 這份合約不包含什麼

本文件只涵蓋請求/回應的格式本身,不包含:

- 產生特定原因代碼的正則表達式/關鍵字規則、
- 信心門檻值或路由邏輯、
- 政策設定檔的實際內容、
- 任何租戶的實際資料。

整合步驟與失敗時的保底行為,請見 [PARTNER_INTEGRATION_GUIDE_V1.md](./docs/PARTNER_INTEGRATION_GUIDE_V1.md)。
