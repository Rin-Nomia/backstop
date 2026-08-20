# Partner Integration Guide (v1)

This guide is for engineering teams at SIs and AI platform companies integrating Backstop in front of a customer-facing AI agent. It covers the integration flow, how to handle each governance decision, and what to expect when something goes wrong.

For field-level API reference, see [CONTRACT_V1.md](../CONTRACT_V1.md). For how governance decisions are produced, see [ARCHITECTURE.md](../ARCHITECTURE.md).

---

## Why `ai_draft` Has to Be Sent First

Backstop's Evaluate layer — the layer that catches unauthorized commitments (refunds, discounts, legal guarantees, policy overrides, contract changes) — checks the AI's *draft reply*, not just the user's message. This is a deliberate design choice: the risk isn't in what the user says, it's in what your AI is about to commit to in response.

**Practical consequence for integration**: your system needs to generate the AI's draft reply *before* calling Backstop, then send that draft alongside the user's message via `ai_draft` in `POST /api/v1/analyze_dual`. Without `ai_draft`, unauthorized-commitment risk detection does not run against any AI reply content — only the Detect path (tone/crisis signals on the user's message) is active.

If your integration currently sends the user's reply to your AI system and returns it directly to the end user without an intermediate governance check, you'll need to add a step: **generate the draft → call Backstop with the draft → act on the decision → then release (or don't release) the reply.**

---

## Shadow Mode: Two Control Mechanisms

Shadow Mode can be activated two ways, and they can be used independently or together:

| Mechanism | Scope | How it's set | Best for |
|---|---|---|---|
| **Header Shadow** | Per-request | `X-Governance-Mode: Shadow` on an individual API call | PoCs, gradual rollout, A/B testing, or observing a specific slice of traffic without changing backend configuration |
| **Profile Shadow** | Tenant/profile-wide default | Configured on the backend via the tenant's policy profile (`policy_profiles/{profile_id}/shadow_mode`) | A formal observation period where an entire tenant should run in Shadow Mode by default |

The two combine as: **`shadow_mode_active = header_shadow OR profile_shadow`** — if either is on, the call runs in Shadow Mode.

**This matters for going live**: removing `X-Governance-Mode: Shadow` from your requests is not enough on its own to reach enforced mode. You also need to confirm the tenant's profile-level shadow setting is turned off on the backend — otherwise the integration will continue running observe-only even with the header removed.

---

## Go-Live Flow

We recommend a three-stage rollout, not a direct cutover:

### Stage 1 — Draft, Don't Enforce (Shadow Mode)
Integrate the API call with `X-Governance-Mode: Shadow` set on every request (or have us enable Profile Shadow on your tenant on the backend). In this mode, Backstop never blocks or alters what your AI sends — `final_decision_state` is always `ALLOW`. You're only collecting `observed_final_decision_state` data to see what governance *would* have done.

This lets you validate the integration itself (are you sending `ai_draft` correctly? is `tenant_id` resolving?) without any risk to production traffic.

### Stage 2 — Review
After a observation period (we suggest at least 14 days — see [README.md](../README.md) for the Shadow Mode timeline), pull a report from `GET /api/v1/reports/risk_14d` and review the observed decisions with your team and, if applicable, the end customer's legal/risk stakeholders. This is the point to confirm the policy strictness matches expectations before anything is actually enforced.

### Stage 3 — Enforce
Remove `X-Governance-Mode: Shadow` from your requests **and** confirm Profile Shadow is disabled for your tenant on the backend — both conditions need to be met before `final_decision_state` will reflect real enforcement. At this point, `GUIDE` and `BLOCK` decisions will affect what reaches the end user — make sure your integration handles all three states correctly (see below) before flipping this switch.

---

## Handling Each Decision State

Your integration is responsible for acting on `final_decision_state` — Backstop tells you what to do, but your system has to actually do it.

| State | What Backstop returns | What your system should do |
|---|---|---|
| `ALLOW` | The draft reply as-is is cleared | Send the AI's draft reply to the user unchanged |
| `GUIDE` | A flagged risk, plus governance-assist fields — `assistant_instruction` (in enforced mode), `draft_reference`, and/or a `shadow_explainability` observation summary — with a recommended strategy and required phrasing | Do **not** send the original draft. Use the returned guidance to regenerate a compliant reply, or substitute a safe fallback message, before it reaches the user |
| `BLOCK` | A flagged risk requiring human review, potentially including a `safe_message` | Do **not** send the original draft under any circumstance. Route the conversation to a human agent |

**Important**: Backstop returns governance decisions and assist fields (`assistant_instruction`, `draft_reference`, `safe_message` where applicable) — but it does not automatically regenerate, substitute, or send a corrected reply on your behalf. Your integration is responsible for the final step: using that guidance to produce a compliant reply, or handing off to a human.

Note that `assistant_instruction` and related enforcement-mode fields are populated when the call is running enforced (not Shadow); in Shadow Mode, the equivalent information surfaces as an *observation* summary (`shadow_explainability`) rather than a live correction directive.

---

## Tenant Configuration

Each integration is bound to a `tenant_id` on the backend, mapped to a policy profile. There is no client-side mechanism to select or switch policy profiles at request time beyond what's configured for your `tenant_id` — this is intentional, so that policy strictness can't be accidentally (or maliciously) overridden from the calling side.

To get a `tenant_id` and, if required, an `X-API-Key` provisioned, contact us directly (see [README.md](../README.md) for contact info). An unrecognized `tenant_id` will be rejected with `403` rather than silently falling back to default policy.

---

## Failure Behavior

If a call to Backstop fails (timeout, 5xx error, network issue), **do not fail open by sending the AI's original draft unreviewed.** This is not something the API enforces on your behalf — it's a fail-closed pattern your integration needs to implement:

1. Substitute a conservative, pre-approved safe message (e.g. "Let me connect you with a specialist to help with this."), and
2. Route the conversation to a human agent.

Treat a failed governance check the same way you'd treat a `BLOCK` — the absence of a clear "yes" should default to caution, not to releasing an unreviewed AI reply.

---

---

## Things to Know Before You Integrate

- **Don't send `policy_profile` in the request body.** The API rejects unrecognized fields (`422`) — policy profile is bound to your `tenant_id` on the backend, not passed per-request.
- **`422` isn't only about text length.** It also covers unrecognized fields in the request body, invalid enum values, and type mismatches — not just `user_text`/`text` being outside 5–500 characters.
- **`403` isn't only about unknown tenants.** It also covers an invalid API key, or a key that doesn't match the tenant it's being used for.
- **`risk_14d` and `shadow_observations` don't take the same parameters.** `risk_14d` accepts `tenant_id`, `policy_profile`, `shadow_mode_only`, and `limit` — it does **not** accept `days` (its window is fixed at 14 days). `shadow_observations` does accept `days`. See [CONTRACT_V1.md](../CONTRACT_V1.md) for the full parameter reference.
- **`observed_*` fields are most meaningful in Shadow Mode.** Outside Shadow Mode, these observation fields are typically `null` — don't build integration logic that depends on them being populated during enforced-mode calls.
- **Hard-code this rule into your integration**: only `ALLOW` may be sent to the end user as-is. `GUIDE` and `BLOCK` must never result in the original draft reaching the user.

---

## Checklist Before Going Live

- [ ] Draft reply generation happens *before* the Backstop call, not after
- [ ] `ai_draft` is populated on every `analyze_dual` call (not just `user_text`)
- [ ] `tenant_id` is correctly set and resolves without `403`
- [ ] Your integration has run in Shadow Mode (Header or Profile) for at least 14 days
- [ ] `GUIDE` and `BLOCK` responses are handled distinctly — neither one results in the original draft reaching the user
- [ ] A conservative fallback path exists for API failures, implemented on your side
- [ ] Someone on your team has reviewed at least one `risk_14d` report before enforcement is turned on
- [ ] Before flipping to enforced mode, confirmed **both** the `X-Governance-Mode: Shadow` header is removed **and** Profile Shadow is disabled on the backend for your tenant
