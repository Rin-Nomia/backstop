# Regression Checklist (P1)

This document describes how Backstop's Evaluate and Detect layers are regression-tested, and reports aggregate results. It does not list the specific test phrases or rule patterns used — see [ARCHITECTURE.md](../ARCHITECTURE.md) for why that detail stays internal.

中文版：[REGRESSION_CHECKLIST_P1.zh-TW.md](./REGRESSION_CHECKLIST_P1.zh-TW.md)

---

## What Gets Tested

Every rule-set change is checked against two categories of test cases before it ships:

1. **Risk detection tests** — known high-risk phrasings across all six unauthorized-commitment categories (refund, discount, compensation, legal guarantee, policy override, contract modification), confirming they still correctly resolve to `GUIDE` or `BLOCK`.
2. **False-positive tests** — ordinary, low-risk customer service phrasing (greetings, order-status inquiries, thank-yous, routine confirmations) in both English and Chinese, confirming they resolve to `ALLOW` and don't get incorrectly flagged.

Both categories are run together on every change to the rule set or tone-classification logic, so a fix in one direction (catching more risk) can be checked against regressions in the other (flagging normal conversation).

---

## Latest Results

### False-Positive Rate (Tone/Detect Layer)

A batch of 12 ordinary customer-service sentences (mixed English/Chinese: greetings, order-status checks, thank-yous) is used as a standing regression set, re-run after every rule-set or tone-classification change.

| | Before tuning | After tuning (confirmed live, 2026-08) |
|---|---|---|
| Incorrectly flagged as `GUIDE`/`BLOCK` | 3 / 12 (25%) | 0 / 12 (0%) |

This result was verified directly against the production deployment (not just a local/staging environment) — every response from the live API includes a `pipeline_version_fingerprint` field, which lets anyone confirm which version of the rule set actually handled their request. At time of writing, the production fingerprint is `88ff8c4d36793534467e58ab019a3b4d993674f205f1bffecbb894a17de7c32d`.

**This is a point-in-time result, not a standing guarantee.** Fingerprints change with every rule-set update. Treat the specific percentage and fingerprint above as accurate as of the date shown — if you're verifying this independently, check that the fingerprint in your own API response matches (or is newer than) the one listed here.

### Risk Detection (Evaluate Layer)

High-risk test cases across all six unauthorized-commitment categories, confirmed against the production deployment (fingerprint above):

| Category | Result |
|---|---|
| Refund | Correctly resolves to `GUIDE` |
| Discount | Resolves to `GUIDE` by default; some tenant policy profiles are configured to escalate this to `BLOCK` (see note below) |
| Compensation | Correctly resolves to `BLOCK` |
| Legal guarantee | Correctly resolves to `BLOCK` |
| Policy override | Correctly resolves to `GUIDE` under tested phrasing — this category is more susceptible to missing novel phrasing when a policy exception is mixed into a sentence with other content, see [Known Limitation](#known-limitation-pattern-based-matching) below |
| Contract modification | Correctly resolves to `BLOCK` |

**Tenant/profile variance**: the decision a given risk category resolves to is not identical across all tenants — policy profiles can be configured with different strictness per category (for example, one tenant may have `discount` set to escalate to `BLOCK` while the default is `GUIDE`). This is an intentional per-tenant configuration option, not inconsistent behavior. See [ARCHITECTURE.md](../ARCHITECTURE.md) for how tenant policy profiles work.

**Evidence tier**: the results above come from two different sources — rule-layer regression tests (run against the current rule set directly) and live E2E spot-checks (actual API calls against the production deployment, verified via `pipeline_version_fingerprint`). Where the two diverge, we report the live/production result as authoritative.

### How to Verify This Yourself

Every response from `POST /api/v1/analyze` or `POST /api/v1/analyze_dual` includes an audit field with the current `pipeline_version_fingerprint`. To confirm you're testing against the version described in this document:

1. Call the API with a test payload (see [Swagger UI](https://rinnomia-continuum-api.hf.space/docs))
2. Check the fingerprint in the response against the one listed above
3. If it doesn't match, the deployment has since changed — the specific numbers in this document may no longer reflect the current state, and it should be re-run

---

## Known Limitation: Pattern-Based Matching

As documented in [ARCHITECTURE.md](../ARCHITECTURE.md), the Evaluate layer currently relies on pattern matching, not semantic understanding. This means:

- Novel phrasings not yet covered by the rule set can be missed, even when the intent is clearly an unauthorized commitment.
- Coverage is expanded iteratively as new phrasings are identified (through testing, real usage, or reported gaps) — this is an ongoing process, not a one-time fix.
- This tradeoff is deliberate: pattern matching is fast, cheap, and fully deterministic, which matters for the latency and auditability guarantees described in [ARCHITECTURE.md](../ARCHITECTURE.md). A move to semantic-understanding matching would trade some of that determinism and latency for broader coverage — see ARCHITECTURE.md's design principles for more on this tradeoff.

**Recommended external-facing framing**: "Unauthorized-commitment detection has broad coverage across six categories and is under continuous expansion; tone/behavioral detection has been tuned to a 0% false-positive rate on standard test phrasing as of this writing." Avoid claiming detection is exhaustive or immune to novel phrasing — it isn't, by design, and that's a tradeoff we're upfront about.

---

## What's Not in This Document

Consistent with [ARCHITECTURE.md](../ARCHITECTURE.md) and [CONTRACT_V1.md](../CONTRACT_V1.md), this checklist does not include:

- the specific test phrases used (risk or false-positive sets),
- the regex/keyword rules themselves,
- confidence thresholds or routing gate values,
- or a list of currently known undetected phrasings.

If you're evaluating Backstop for a specific use case and want to understand coverage for your domain's likely phrasings, contact us directly (see [README.md](../README.md)) — we're glad to run a scoped test against representative examples under NDA rather than publish a roadmap of current gaps.
