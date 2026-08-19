# Backstop

**Runtime Governance for Enterprise AI — Don't replace your AI. Backstop it.**

Built for SIs (system integrators) and AI platform companies delivering Agentic AI into high-compliance industries like finance and healthcare — solving the problem of "AI making unauthorized commitments, stalling projects at the customer's legal/risk review gate."

中文版：[README.zh-TW.md](./README.zh-TW.md)

---

## What This Is

Backstop is a Runtime Enforcement Gateway that sits independently of the underlying LLM. It adds one more layer of review before a customer-facing AI's reply is sent, detecting and intercepting **unauthorized commitments** — refunds, discounts, legal guarantees, policy exceptions.

It doesn't replace your existing AI system or change model behavior. It plugs in as an add-on layer.

---

## The Problem

Once enterprises deploy AI-powered customer service, the costliest risk isn't the AI saying something wrong — it's **the AI saying something no one authorized it to say**.

Refunds, discounts, legal guarantees, policy exceptions — under pressure, AI agents readily make commitments the business never authorized. Once it's said, there's no record, and no one can explain why.

This isn't an isolated problem. According to a survey published by the Cloud Security Alliance in April 2026 [^1], 53% of organizations have already experienced AI agents exceeding their intended permissions. OWASP formally defines this phenomenon as **Excessive Agency** — one of the primary risk categories in LLM deployments today.

**For SIs and AI platform companies delivering AI to customers in finance, healthcare, and other high-compliance industries, this problem is even more direct: every new customer means rewriting risk-control rules from scratch; and when something goes wrong, there's no record to explain what actually happened — this is the primary reason projects get stuck at the customer's legal/risk review gate, with final payment held up.**

[^1]: Cloud Security Alliance, *Enterprise AI Security Starts with AI Agents* (commissioned by Zenity, published April 16, 2026; survey fielded September–November 2025; 445 IT/security professionals surveyed). [Official press release](https://cloudsecurityalliance.org/press-releases/2026/04/16/more-than-half-of-organizations-experience-ai-agent-scope-violations-cloud-security-alliance-study-finds)

---

## Try It Now

No docs to read, no code to write — try it directly in your browser:

- 🎮 **Live Playground**: [rinnomia-continuum-api.hf.space/playground](https://rinnomia-continuum-api.hf.space/playground) — enter a conversation and see Evaluate/Audit decisions with a Before/After comparison in real time

---

## Core Architecture: Four Layers

Backstop is composed of four layers, each with a distinct responsibility:

| Layer | Name | Function |
|---|---|---|
| 1 | **Detect** | Classifies the user's high-risk interaction state (e.g. high-pressure escalation, emotional distress, crisis signals) |
| 2 | **Evaluate** | Before an AI reply is sent, checks the draft for unauthorized commitments (refund, discount, compensation, legal guarantee, policy override, contract modification). Currently **pattern-based, continuously expanding** — not yet semantic understanding; known limitations and test coverage are tracked in [REGRESSION_CHECKLIST](./docs/REGRESSION_CHECKLIST_P1.md) |
| 3 | **Control** | Real-time intervention across three tiers: pass through and log / guide and correct the reply / intercept and route to a human |
| 4 | **Audit** | Every high-risk moment produces a complete record: trigger reason, risk type, governance decision, timestamp |

**Measured performance (production, 2026-07-31)**: P95 350ms at 5 concurrent requests; P95 91ms for a single request. Cloud-to-cloud benchmark — see [ARCHITECTURE.md](./ARCHITECTURE.md) for details.

Full architecture documentation: [ARCHITECTURE.md](./ARCHITECTURE.md).

---

## Quick Start (for engineers)

📖 Full API documentation (Swagger UI): [rinnomia-continuum-api.hf.space/docs](https://rinnomia-continuum-api.hf.space/docs)

Direct call example:

```bash
curl -X POST https://rinnomia-continuum-api.hf.space/api/v1/analyze_dual \
  -H "Content-Type: application/json" \
  -H "X-Governance-Mode: Shadow" \
  -d '{
    "user_text": "If you don't fix this, I'll report you to the media.",
    "ai_draft": "I can process a refund for you right now."
  }'
```

The response includes the governance decision (ALLOW/GUIDE/BLOCK), risk type, reason code, and a complete audit record. Full field reference: [CONTRACT_V1.md](./CONTRACT_V1.md).

---

## Adoption Path: Shadow Mode (Zero-Risk Observation)

Not sure whether to roll it out fully yet? Start with zero-risk observation:

- **Day 1**: API mounted silently — no responses are intercepted, zero business risk
- **Day 7**: High-risk conversations are automatically logged — trigger reasons and risk types fully retained
- **Day 14**: First AI Risk Report auto-generated — ready to hand directly to legal/risk teams for review

---

## Current Status

Backstop is an actively iterating product. The rule set covers six categories of unauthorized commitment — refund, discount, compensation, legal guarantee, policy override, contract modification — built on research across 30 cross-industry AI incident cases. For known limitations and test results, see [REGRESSION_CHECKLIST.md](./docs/REGRESSION_CHECKLIST_P1.md).

---

## Contact

Shih Huan Chen
Founder, Backstop | AI Governance Systems
Email: rin.nomia.series@gmail.com
LinkedIn: [linkedin.com/in/shih-huan-chen-7998503aa](https://www.linkedin.com/in/shih-huan-chen-7998503aa/)

---

🇹🇼 [中文版本](./README.zh-TW.md)
