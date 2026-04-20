# Business Problem & Success Metrics: Waze Churn Prediction

> This document is the PM brief — written before any modeling — to anchor every subsequent technical decision in a business outcome.

---

## 1. Problem Statement

**User:** Active Waze users who have been using the app for at least some time but are showing signs of disengagement.

**Problem:** Users who are about to uninstall give no obvious signal. They don't close their account — they simply stop opening the app and eventually uninstall. By the time this shows up in a DAU dashboard, the user is already gone.

**Current workaround:** Threshold-based flagging — e.g., "send a re-engagement push to anyone inactive for 14+ days." This fails in two directions:
- **False positives:** Occasional users (weekend drivers, vacation-trippers) go quiet mid-week routinely. They trigger the rule, receive an unwanted notification, and are now *more* likely to uninstall.
- **False negatives:** A user quietly disengaging — opening the app less frequently, taking shorter trips — won't cross an inactivity threshold until after the decision to leave has already been made.

**Business impact:**

| Impact | Description |
|--------|-------------|
| Lost ad impressions | Each churned user = future ad inventory gone. Waze monetizes through in-map ads, branded pins, and promoted search. A churned user generates $0. |
| Re-acquisition cost | Getting a user who uninstalled to reinstall and re-engage is significantly more expensive than retaining them with a timely nudge |
| DAU pressure | Waze's value to advertisers and to Google depends on maintaining active user counts. Silent churn erodes DAU without a visible drop-off event to trigger investigation |
| Invisible at scale | At Waze's scale (millions of MAU), a 1% improvement in monthly retention meaningfully affects ad inventory and negotiating leverage with advertisers |

---

## 2. Why This Is an ML Problem

Three reasons a rule-based or manual approach can't solve this:

**1. Volume of decisions.** Waze needs to score millions of users every retention cycle. This isn't one high-stakes decision (statistics) or an exploratory question (analytics) — it's *many repeated decisions at scale*. That's the definition of an ML problem.

**2. Feature interaction complexity.** Churn risk emerges from the combination of signals: sessions per day, drive frequency, trip length, time since onboarding, favorite destinations used, activity days in a rolling window. A user who drives once a day every weekday is very different from a user who drove 20 times in one week and then nothing. No threshold rule captures these interaction patterns at the individual level. An ML model draws the decision boundary from data; the PM's job is to define what "correct" means.

**3. The naive baseline is deceptively dangerous.** A model that predicts "nobody churns" hits ~82% accuracy on this dataset — and is completely useless for retention targeting. The moment you evaluate against business impact rather than raw accuracy, rule-based and naive models collapse.

---

## 3. Label Definition

Defined precisely before any modeling begins.

| Term | Definition |
|------|------------|
| **Churn = 1** | User uninstalls the app or becomes permanently inactive within the observation window |
| **Churn = 0** | User remains active (opens app, completes navigation sessions) |
| **Prediction horizon** | Rolling 30-day window |
| **Observation window** | Behavioral features drawn from the prior 30 days of in-app activity |

**Why behavioral signals?** Unlike a subscription product where churn is a transactional event (non-renewal), Waze churn is behavioral — there's no cancellation moment. The model must infer intent from engagement patterns before the uninstall decision is made.

**Edge cases to define upfront:**
- Users with zero drives but non-zero sessions: may be using Waze passively (e.g., traffic checks). Not immediately churn-risk if session count is stable.
- Users with very short tenure (<30 days): early-lifecycle risk is real but driven by onboarding failure, not the same behavioral pattern as mid-lifecycle churn. Same model applies but intervention differs.
- Seasonal drivers (summer road-trippers, holiday commuters): behavioral patterns vary. Model should be retrained or re-evaluated per season if deployment data shows distributional drift.

---

## 4. Success Metrics — Layered Framework

Success metrics are set here, before building, so evaluation is objective rather than post-hoc.

### Layer 1: Technical Metrics (model evaluation)

These determine whether the model is worth deploying.

| Metric | Target | Rationale |
|--------|--------|-----------|
| **Recall (sensitivity)** | ≥ 65% | Primary metric. A missed churner = permanent lost user — re-acquisition is expensive and uncertain. The cost of a false negative (lost user) exceeds the cost of a false positive (unnecessary push notification). |
| **Precision floor** | ≥ 35% | Hard floor, not a target. Unlike a subscription product where a false positive means wasting a $10 discount, Waze's false positive cost is attention friction from an unwanted notification. The floor is lower, but still matters: over-notification drives churn in its own right. |
| **ROC-AUC** | ≥ 0.75 | Used to compare models and rank candidates — not the optimization target. Useful because it's threshold-agnostic. |
| **Baseline to beat** | Logistic regression recall @ precision ≥ 35% | Any complex model must outperform the interpretable baseline on this metric to justify the added complexity cost. |

**Why not accuracy?** On this dataset, churn rate is roughly 18%. A model predicting "nobody churns" scores ~82% accuracy. It is completely useless for retention targeting. Accuracy is rejected as a primary metric.

### Layer 2: Business Metrics (operational success)

These determine whether the model is *actually working* in production. Measured via A/B test: model-targeted re-engagement vs. control (no intervention).

| Metric | Target | Rationale |
|--------|--------|-----------|
| **Retention rate (flagged users)** | ≥ 20% of flagged users retained who would have churned | Did the intervention actually prevent uninstall? Requires holdout group to measure counterfactual. |
| **Notification response rate** | CTR on retention push > 8% | Are flagged users actually engaging with the re-engagement prompt? Low CTR signals that targeting or message is off. |
| **DAU impact** | No measurable decline in DAU for control group vs. flagged group after intervention | Confirm the model isn't generating so many false positives that over-notification is accelerating churn in the flagged segment. |

### Layer 3: Guardrail Metrics (don't break these)

These are non-negotiable. Violating them means the model should not ship.

| Metric | Limit | Rationale |
|--------|-------|-----------|
| **Precision floor** | ≥ 35% at chosen operating threshold | Below this, re-engagement campaigns flag so many active users that notification fatigue accelerates churn rather than preventing it. |
| **% of base flagged** | ≤ 25% of monthly active users per cycle | If the model flags everyone, it has no targeting value. Over-messaging loyal users is a product-harming false positive. |
| **Inference latency** | Batch scoring must complete ≥ 24 hours before retention campaigns deploy | Operational constraint. Model outputs must be ready before the CRM or push team schedules sends. |
| **Demographic parity check** | Churn prediction rate should not vary >15% across device cohorts (iOS vs. Android) without a behavioral explanation | Fairness guardrail — model should not systematically under-serve any user segment. |

---

## 5. PM's Reasoning

**Why recall as the primary metric?**
The asymmetry of costs drives this. A false negative (missed churner) means a permanently lost user — with uncertain, expensive re-acquisition. A false positive (active user flagged for a re-engagement nudge) costs one push notification send and minor attention friction. These costs are not symmetric. Optimize for the more expensive error.

**Why a lower precision floor than a subscription product?**
In a subscription context (e.g., KKBox), a false positive costs an actual discount offer ($10+). Here, the false positive cost is a push notification — attention friction, not dollars. So the floor is lower at 35%. But it's not zero. If the model flags 70% of users as churn risk, two bad things happen:
1. Loyal, active users receive unwanted nudges and become annoyed — which can itself cause churn
2. The product team stops trusting the model's output and reverts to manual rules

The floor is still a credibility and product-safety constraint, just calibrated to the actual cost of the false positive in this context.

**Why set thresholds before modeling?**
If performance thresholds are set after seeing results, they drift toward whatever the model happened to achieve. "Set standards while sober, before you've poured your heart into it." The targets above were set using business logic (re-acquisition costs, notification fatigue research, operational constraints) — not by looking at model outputs first.

**Why logistic regression as the baseline?**
When pitching a re-engagement campaign budget to marketing or the product lead, you need to explain *why* specific users were flagged. "Low activity days this month and few completed drives relative to sessions" is a sentence a non-technical stakeholder can act on and trust. A gradient boosting feature importance chart is not. Starting with an interpretable baseline also forces rigorous feature engineering before introducing model complexity — complexity that should only be added if it materially improves recall@precision≥35%.

---

## 6. How Thresholds Were Derived

Thresholds are business decisions, not model decisions. They are derived from the cost of each type of mistake — not from model outputs.

### The Core Tradeoff

- **High recall, low precision** — flag more users, catch more churners, but also flag many active users for re-engagement. Comprehensive but risks notification fatigue.
- **High precision, low recall** — flag only confident churners, send fewer nudges, but miss more users who quietly uninstall.

The right balance depends on: cost of a lost user (re-acquisition), cost of over-notification (potential acceleration of churn), and stakeholder risk tolerance.

### Deriving the Precision Floor from Notification Economics

Waze's re-engagement intervention has near-zero direct cost (push notification). But notification fatigue has a *secondary churn effect*.

Research benchmark: users who receive >3 unsolicited push notifications per week have meaningfully higher uninstall rates than users who receive targeted, relevant notifications. This means false positives carry an indirect cost:

| Scenario | Impact |
|---|---|
| Low precision (10–20%) | 80–90% of flagged users are active and don't need intervention → notification fatigue → higher uninstall risk for that group |
| Moderate precision (35–40%) | Most flagged users are genuinely at risk → notifications feel timely, not intrusive |
| High precision (60%+) | Achievable only by sacrificing recall — too many churners slip through undetected |

35% is the business floor because it keeps the false positive rate low enough to avoid measurable notification fatigue effects while preserving enough recall to intervene on the majority of at-risk users.

### Deriving the Recall Target from Re-Acquisition Cost

At 65% recall, 35% of actual churners are missed. Each missed churner = one user who uninstalls and requires re-acquisition.

Illustrative re-acquisition cost at Waze scale:

| Assumption | Estimate |
|---|---|
| Cost to re-acquire one user (app store ads, referral) | ~$2–5 |
| Monthly active users at risk of churn | ~500K |
| Churn rate | ~18% → ~90K churners/month |
| Missed at 35% miss rate | ~31,500 users |
| Re-acquisition cost for misses | ~$63K–$157K/month |

A retention push notification costs essentially $0. Even catching 10,000 additional churners per month with a well-targeted nudge is financially justified at virtually any recall improvement.

**65% is the initial deployment target** because:
1. It's achievable with gradient boosting on this dataset
2. It materially outperforms rule-based inactivity thresholds
3. Once the team trusts the model, the precision floor can be renegotiated upward and recall pushed higher

### Summary: What Sets Each Number

| Threshold | Driven by |
|---|---|
| Precision floor (35%) | Notification fatigue risk + stakeholder credibility |
| Recall target (65%) | Re-acquisition cost of missed churners vs. attention friction of over-notification |
| ROC-AUC floor (0.75) | Model ranking benchmark — ensures the model has real discriminatory power |

---

## 7. Decision Framework Summary

```
Business question:  Which Waze users are likely to uninstall in the next 30 days?
ML task:            Binary classification (churn / retained)
Label:              Churn = user uninstalls or becomes permanently inactive
Primary metric:     Recall ≥ 65%
Hard floor:         Precision ≥ 35%
Summary metric:     ROC-AUC ≥ 0.75
Baseline:           Logistic regression
Rejected metric:    Accuracy (misleading on imbalanced data — 82% baseline accuracy by predicting nobody churns)
Intervention:       Tiered re-engagement (high confidence churn → personalized push + feature highlight; medium → in-app nudge; low → monitor)
```
