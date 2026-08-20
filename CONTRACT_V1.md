# Data Contract (v1)

This document describes the request/response shape for Backstop's public API — the fields you'll see when integrating, not the internal rule definitions behind them. For the full interactive spec, see the [Swagger UI](https://rinnomia-continuum-api.hf.space/docs) or [openapi.json](https://rinnomia-continuum-api.hf.space/openapi.json).

中文版：[CONTRACT_V1.zh-TW.md](./CONTRACT_V1.zh-TW.md)

---

## Base URL

```
https://rinnomia-continuum-api.hf.space
```

---

## Authentication

Tenants configured with `requires_api_key: true` must supply their key via the `X-API-Key` header. Public/demo endpoints do not require one.

---

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/health` | Liveness check |
| `POST` | `/api/v1/analyze` | Single-path evaluation (`text` only) |
| `POST` | `/api/v1/analyze_dual` | Dual-path evaluation (`user_text` + `ai_draft`) |
| `GET` | `/api/v1/metrics/kpi` | Aggregate business KPIs |
| `GET` | `/api/v1/metrics/casebook` | Casebook-style evidence records |
| `GET` | `/api/v1/metrics/shadow_observations` | Shadow Mode observation report, filterable by tenant |
| `GET` | `/api/v1/reports/risk_14d` | 14-day AI Risk Report, filterable by tenant |

---

## Headers

| Header | Values | Purpose |
|---|---|---|
| `Content-Type` | `application/json` | Required on all `POST` requests |
| `X-Governance-Mode` | `Shadow` (omit for enforced mode) | When set to `Shadow`, Control always resolves the externally enforced decision to `ALLOW`; the would-be decision is still recorded |
| `X-API-Key` | tenant-issued key | Required only for tenants configured with `requires_api_key: true` |

---

## Request: `POST /api/v1/analyze_dual`

```json
{
  "user_text": "If you don't fix this, I'll report you to the media.",
  "ai_draft": "I can process a refund for you right now.",
  "tenant_id": "tenant-a"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `user_text` | string | yes | 5–500 characters. The end user's message. |
| `ai_draft` | string | optional | The AI's proposed reply. Required for the Evaluate layer to have anything to check — without it, only Detect runs on `user_text`. |
| `tenant_id` | string | optional | Defaults to `default` if omitted. An unrecognized `tenant_id` returns `403`. |

The request body only accepts the fields listed above — unrecognized fields are rejected (`422`).

### `POST /api/v1/analyze` uses a different, simpler shape

```json
{
  "text": "If you don't fix this, I'll report you to the media.",
  "source": "user",
  "tenant_id": "tenant-a"
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `text` | string | yes | 5–500 characters. |
| `source` | string | optional | `"user"` or `"ai_draft"` — which side of the conversation `text` represents. |
| `tenant_id` | string | optional | Same behavior as above. |

The response format is the same as `POST /api/v1/analyze_dual` (see below), except `audit_summary.trigger` is fixed to whatever value you passed as `source` rather than being inferred from having both `user_text` and `ai_draft` present.

---

## Response: `POST /api/v1/analyze_dual`

```json
{
  "shadow_mode_active": true,
  "final_decision_state": "ALLOW",
  "observed_final_decision_state": "GUIDE",
  "observed_final_intervention_reason_code": "unauthorized_refund_commitment",
  "observed_final_risk_label": "Unauthorized commitment risk",
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

| Field | Type | Notes |
|---|---|---|
| `final_decision_state` | `"ALLOW"` \| `"GUIDE"` \| `"BLOCK"` | The decision actually enforced on this response. Always `"ALLOW"` when `X-Governance-Mode: Shadow` is set. |
| `observed_final_decision_state` | `"ALLOW"` \| `"GUIDE"` \| `"BLOCK"` \| `null` | What the decision would have been if enforced. Populated in Shadow Mode; typically `null` outside Shadow Mode. |
| `observed_final_intervention_reason_code` | string \| `null` | Machine-readable reason (see [Reason Codes](#reason-codes) below). `null` when no risk is detected. |
| `observed_final_risk_label` | string | Human-readable risk category. |
| `audit_summary` | object | The record retained for Audit. See below. |
| `audit_summary.trigger` | string | `"user"` / `"ai_draft"` / `"both"` — which side of the conversation triggered the evaluation. |
| `audit_summary.evidence` | object | Structured, source-specific evidence supporting the decision. Internal structure not part of this public contract. |
| `audit_summary.risk` | string | Same as `observed_final_risk_label`. |
| `audit_summary.timestamp` | string (ISO 8601, UTC) | When the decision was made. |
| `audit_summary.reason_code` | string \| `null` | Same as `observed_final_intervention_reason_code`. |
| `audit_summary.shadow_explainability` | object | Present when the decision is `GUIDE`/`BLOCK`. A redacted, customer-facing summary — risk type, reason code, a recommended strategy, and suggested required phrasing. `mode` is always `"summary_only"`; the full internal rewrite instruction is never exposed via this field. |

---

## Response: Enforced Mode (not Shadow)

When a call runs in enforced mode (no `X-Governance-Mode: Shadow` header, and Profile Shadow disabled for the tenant — see [PARTNER_INTEGRATION_GUIDE_V1.md](./docs/PARTNER_INTEGRATION_GUIDE_V1.md)) and the decision is `GUIDE` or `BLOCK`, the response includes governance-assist fields not present in the Shadow Mode example above.

Real example (`GUIDE`, enforced mode):

Request:
```json
{
  "tenant_id": "tenant-a",
  "user_text": "Please check my order status.",
  "ai_draft": "I can refund you immediately."
}
```

Response (fields relevant to this section):
```json
{
  "shadow_mode_active": false,
  "final_decision_state": "GUIDE",
  "observed_final_decision_state": null,
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

| Field | Type | Notes |
|---|---|---|
| `assistant_instruction` | object \| `null` | Populated in enforced mode on `GUIDE`/`BLOCK`. A structured correction directive — objective, forbidden commitment types, required phrases, and the reason codes it's addressing. `delivery_mode: "reference_only"` and `do_not_use_as_final_reply: true` are always present: **this object is guidance for regenerating a reply, never the reply itself.** `null` in Shadow Mode (see `audit_summary.shadow_explainability` instead). |
| `draft_reference` | string \| `null` | A reference text string for regeneration. It is often a governed rewrite rather than an echo of the original `ai_draft` — do not assume it equals what you sent in. |
| `safe_message` | string \| `null` | Populated for enforced `BLOCK` decisions (a default fallback exists at the system level). Typically `null` for `GUIDE`/`ALLOW`, and always `null` in Shadow Mode observations. |

---

## Reason Codes

Reason codes are grouped by the six unauthorized-commitment categories the Evaluate layer checks for:

| Category | Reason code |
|---|---|
| Refund | `unauthorized_refund_commitment` |
| Discount | `unauthorized_discount_commitment` |
| Compensation | `unauthorized_compensation_commitment` |
| Legal guarantee | `unauthorized_legal_guarantee` |
| Policy override | `unauthorized_policy_override` |
| Contract modification | `unauthorized_contract_change` |

Additional operational reason codes exist for routing, tone, and safety signals (not tied to a specific commitment category), including but not limited to: `mandatory_human_handoff`, `crisis_self_harm`, `escalation_pressure`, `ambiguous_commitment_risk`, `pressure_induced_commitment`, `TONE_PUSHY`, `TONE_ANXIOUS`, `TONE_COLD`, `TONE_SHARP`, `TONE_BLUR`, `TONE_UNKNOWN`, `TONE_RHYTHM`, `OOS_CRISIS`, and a fallback `intervention_unknown`.

This list documents the *shape* of reason codes, not the full current set — the underlying rule definitions that generate them are not public. See [ARCHITECTURE.md](./ARCHITECTURE.md) for how these map to governance layers.

**Note on naming convention**: commitment-type codes (`unauthorized_*`) use lower `snake_case`, while tone/crisis-signal codes (`TONE_*`, `OOS_CRISIS`) use `ALL_CAPS`. This reflects the two different originating modules (Evaluate's `commitment_guard` vs. Detect's `classifier`/`safety_gate` — see the module mapping table in [ARCHITECTURE.md](./ARCHITECTURE.md)), not an inconsistency in the API design.

---

## Errors

| Status | Meaning |
|---|---|
| `403` | Rejected at the tenant layer — an unrecognized `tenant_id`, an invalid API key, or a key that doesn't match the requested tenant. The request is not silently processed against a default tenant. |
| `422` | Validation error — `user_text`/`text` outside the 5–500 character range, an unrecognized field in the request body, an invalid enum value, or a type mismatch. |

---

## Reports

### `GET /api/v1/metrics/shadow_observations`

Query parameters: `tenant_id`, `policy_profile`, `days`, `limit`.

### `GET /api/v1/reports/risk_14d`

Query parameters: `tenant_id`, `policy_profile`, `shadow_mode_only`, `limit`. The reporting window is fixed at 14 days — this endpoint does not accept a `days` parameter.

```
GET /api/v1/reports/risk_14d?tenant_id=tenant-a&shadow_mode_only=true&limit=10
```

*The example below is a zero-data initial-state sample (a freshly onboarded tenant with no incidents yet) — `deployment_confidence_score` here reflects "nothing has gone wrong in the observed window," not a scoring formula applied to `prevented_risk_count`.*

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

## What's Not in This Contract

This document covers the request/response shape only. It does not include:

- the regex/keyword rules that produce a given reason code,
- confidence thresholds or routing gates,
- policy profile configuration format,
- or any tenant-specific data.

See [PARTNER_INTEGRATION_GUIDE_V1.md](./docs/PARTNER_INTEGRATION_GUIDE_V1.md) for integration steps and fallback behavior.
