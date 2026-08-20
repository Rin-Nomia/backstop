# 更新紀錄

Backstop 所有重大變更都記錄在這裡,由新到舊排列。日期反映的是「這項變更確認在正式環境生效」的時間,不是開始開發的時間。

English version: [CHANGELOG.md](./CHANGELOG.md)

---

## 2026-08

### 新增
- **Shadow Explainability**——Shadow Mode 報告現在包含一份對外可見的 `shadow_explainability` 摘要(風險類型、原因代碼、建議因應策略、建議措辭),不再只有一個決策標籤。用來組成 AI 引導建議的完整內部指令集,仍然只保留在系統內部。
- **`pipeline_version_fingerprint`**——每一次 API 回應現在都會包含一組指紋,標識目前處理這次請求的規則庫跟語氣分類邏輯是哪個版本,任何人都能自行驗證自己測試的是哪個部署版本。
- **租戶報告下載頁**——一個上線的網頁介面([`/report-download`](https://rinnomia-continuum-api.hf.space/report-download)),可以選擇租戶,直接下載該租戶的 Risk Report 或 Shadow Observations(JSON 或 Markdown 格式),不需要手動組合查詢網址。
- **P1 規則庫擴充**——補齊了賠償、合約、法律保證這三個承諾類別的同義詞跟講法涵蓋範圍(例如 *restitution*、*indemnify*、*binding commitment*、*legally guaranteed*),另外也補了退款/折扣的講法(例如 *reimburse*、*credit your account*、百分比折扣的各種變化)。
- **`REGRESSION_CHECKLIST_P1`**——一份固定的回歸測試矩陣,每次規則庫或語氣分類邏輯改動後都會重新跑一次,同時追蹤風險偵測準確度跟誤判率。

### 變更
- **產品更名為 Backstop**(先前稱為 Trust Layer),原因是商標清晰度考量。
- **語氣敏感度調校**——調整了路由信心門檻,並加入短句/禮貌用語防護機制,將一份 12 案例標準測試集上的誤判率從 25% 降到 0%。

### 修正
- 修正了一個部署延遲問題:正式環境當時跑的是比上述語氣敏感度修正還要舊的版本——這個問題是透過比對 `pipeline_version_fingerprint` 才被抓到的,修正後也已經在正式環境重新驗證過。

---

## 2026-07

### 新增
- **正式環境延遲效能基準測試**——首次公開發布的效能數字:單路徑評估 P95 91ms,雙路徑評估在 5 併發情境下 P95 350ms,所有測試情境的錯誤率皆為 0%。

---

## 更早之前

- 初版四層治理管線(Detect、Evaluate、Control、Audit)上線至正式環境。
- Shadow Mode(僅觀察、零風險導入模式)完成實作。
- 多租戶政策綁定——每個租戶對應一份獨立的政策設定檔,無法辨識的租戶會直接被拒絕,不會退回使用預設值處理。
- 新增中英文危機偵測規則。

---

*本更新紀錄涵蓋的是產品面向使用者的變更。不包含內部重構、純文件修訂、或非公開工具的異動。*
