# Changelog

All notable changes to Backstop are documented here, most recent first. Dates reflect when a change was confirmed live in production, not when work began.

中文版：[CHANGELOG.zh-TW.md](./CHANGELOG.zh-TW.md)

---

## 2026-08

### Added
- **Shadow Explainability** — Shadow Mode reports now include a customer-facing `shadow_explainability` summary (risk type, reason code, recommended strategy, required phrasing) instead of just a decision label. The full internal instruction set used to construct AI guidance remains internal-only.
- **`pipeline_version_fingerprint`** — every API response now includes a fingerprint identifying which version of the rule set and tone-classification logic handled the request, so anyone can independently verify which deployed version they're testing against.
- **Per-tenant report download** — a hosted interface at [`/report-download`](https://rinnomia-continuum-api.hf.space/report-download) for selecting a tenant and downloading its Risk Report or Shadow Observations as JSON or Markdown, without manually constructing query URLs.
- **P1 rule-set expansion** — added synonym and phrasing coverage across the compensation, contract, and legal-guarantee commitment categories (e.g. *restitution*, *indemnify*, *binding commitment*, *legally guaranteed*), plus additional refund/discount phrasing (e.g. *reimburse*, *credit your account*, percentage-off variants).
- **`REGRESSION_CHECKLIST_P1`** — a standing regression test matrix, re-run after every rule-set or tone-classification change, tracking both risk-detection accuracy and false-positive rate.

### Changed
- **Renamed the product to Backstop** (previously referred to as Trust Layer) for trademark clarity.
- **Tone-sensitivity tuning** — adjusted routing confidence thresholds and added a short-text/politeness guard, reducing the false-positive rate on a 12-case standard test set from 25% to 0%.

### Fixed
- A deployment lag where the production environment was running an earlier commit than the tone-sensitivity fix described above — caught via `pipeline_version_fingerprint` comparison, then corrected and re-verified live.

---

## 2026-07

### Added
- **Production latency benchmark** — first published performance figures: P95 91ms for single-path evaluation, P95 350ms for dual-path evaluation at 5 concurrent requests, 0% error rate across all tested scenarios.

---

## Earlier

- Initial four-layer governance pipeline (Detect, Evaluate, Control, Audit) deployed to production.
- Shadow Mode (observe-only, zero-risk onboarding) implemented.
- Multi-tenant policy binding — each tenant mapped to an independent policy profile, with unrecognized tenants rejected outright rather than falling back to a default.
- Crisis-detection rules added for both English and Chinese input.

---

*This changelog covers product-facing changes. It does not include internal refactors, documentation-only edits, or changes to non-public tooling.*
