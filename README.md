# Every PM's Biggest Problem: Building a Churn Prediction System

> Churn is the defining retention problem for any product with recurring engagement. Once a user is gone, they're gone — and by the time it shows up in your metrics, it's already too late to act. This project walks through how a PM should frame, build, and evaluate a churn prediction system, using a Waze behavioral dataset as the vehicle.

---

## The Problem

You lose users silently. They don't close an account — they just stop showing up. By the time this surfaces in a dashboard, they're already gone.

Without prediction, retention interventions are reactive: campaigns go to everyone, and high-risk users slip away undetected while aggregate metrics look healthy.

The obvious workaround — "flag anyone inactive for 14 days" — fails in both directions. Casual users trigger it without churning. Disengaging users often remain minimally active right up until they leave.

**Dataset:** [Waze User Behavior Dataset](https://github.com/InaRailean/Waze-Project) — ~15,000 users, 11 behavioral features, ~18% churn rate.

---

## Why AI

1. **Volume.** Scoring millions of users each cycle is many repeated decisions at scale — ML, not analytics or statistics.
2. **Feature interaction.** Churn risk emerges from combinations — session frequency, drive completion, trip length, days since onboarding. No threshold rule captures this at the individual level.
3. **The naive baseline fails.** Predicting "nobody churns" scores ~82% accuracy on this dataset and is useless for targeting.

---

## What I Built

1. **Problem framing** — defined label, metric, and thresholds *before* touching data
2. **EDA** — class imbalance (~18% churn), feature distributions by label, device breakdown
3. **Modeling** — logistic regression → random forest → XGBoost; compared on recall @ precision ≥ 35%
4. **PM decision layer** — tiered intervention strategy mapped to model confidence

---

## How I Built It

- Used **Claude Code** to generate the Python notebook from a problem brief
- Ran and iterated on the notebook in **Google Colab**
- Dataset loads automatically from a public GitHub URL — no setup needed

---

## Tradeoffs

Optimized for **recall with a precision floor (~35%)**, not accuracy. A missed churner is gone; re-acquisition is expensive. A false positive is one unwanted notification — meaningful only if over-notification itself triggers churn.

Accuracy was explicitly rejected. A model predicting "nobody churns" hits 82% — and is worthless for targeting.

---

## Concepts: Precision, Recall & the Cost of Each Error

See [concepts-precision-recall.md](./concepts-precision-recall.md) for a plain-English walkthrough — what false positives and false negatives mean in this context, how precision and recall are calculated, and why the cost asymmetry drives metric selection.

---

## PM's Reasoning

I set recall as the primary metric because the cost of missing a churner (permanent loss, re-acquisition cost) outweighs the cost of a false positive (a push notification to an active user). The precision floor exists because over-notification has its own churn effect.

I set thresholds before building, not after. Standards set post-hoc drift toward whatever the model happened to achieve.

---

## Tech Stack

- Python (pandas, scikit-learn, xgboost, matplotlib, seaborn)
- Google Colab
- Dataset loads directly from GitHub — no download needed

---

## How to Run

Open the notebook in Colab and run all cells — data loads automatically.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](Apr_waze_churn_prediction.ipynb)

---

## Key Findings

| Signal | Insight |
|--------|---------|
| `activity_days` | Strongest predictor — low monthly activity is the clearest churn signal |
| `drives_per_day` | Sessions without completed drives suggest friction or disengagement |
| `km_per_drive` | Short trips correlate with churn — habit not yet formed |
| `n_days_after_onboarding` | Newer users churn more — early lifecycle is highest risk |

---

## What I Learned

- **Define the business problem before touching data.** The most important work happened in a doc, not a notebook — who is churning, what does churn mean exactly, what's the cost of each type of mistake. Without that, the model has no anchor.
- **Set success metrics before you see results.** I defined recall as the primary metric and set the precision floor before running a single model. If you set thresholds after seeing outputs, they drift toward whatever the model happened to achieve.
- **Accuracy is almost never the right metric for a churn problem.** A model predicting "nobody churns" can score 82% accuracy and be completely useless. The PM's job is to reject the obvious metric and define the right one.
- **The intervention design is a PM decision, not a model decision.** The model gives probabilities. Deciding what to do with a 70% churn-probability user vs. a 40% one — send a push, show an in-app nudge, do nothing — is product judgment, not engineering.
- **Framing churn as an ML problem (vs. analytics or a rules system) requires a specific argument.** Volume of decisions, feature interaction complexity, and failure of the naive baseline — these are the three tests. Being able to articulate why ML is the right tool (not just the cool one) is a PM skill.
