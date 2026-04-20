# Concept: Precision, Recall, False Positives & False Negatives

> A plain-English explanation using the Waze churn prediction problem as the example. Written for anyone new to ML evaluation metrics.

---

## Step 1: The Setup

Waze has users. Some will uninstall the app (churn). Some will keep using it.

Let's pretend there are only **10 users** to keep the math tiny.

*(Assumed: 10 users total, just to make numbers easy.)*

Of these 10, **2 will actually churn** and **8 will keep using Waze.**

*(Assumed: 2 churners, 8 stayers. In the real Waze dataset, the churn rate is ~18%, but we're keeping it simple.)*

```
User 1  → will CHURN (uninstall)
User 2  → will CHURN (uninstall)
User 3  → will stay
User 4  → will stay
User 5  → will stay
User 6  → will stay
User 7  → will stay
User 8  → will stay
User 9  → will stay
User 10 → will stay
```

---

## Step 2: The Model Makes Guesses

The model doesn't know the future. It looks at each user's driving behavior — sessions, activity days, km per drive, etc. — and guesses: "churn risk" or "not churn risk."

Let's say the model flags **3 people** as churn risk:

```
User 1  → model says: CHURN RISK ✓  (actually will churn — correct catch!)
User 2  → model says: CHURN RISK ✓  (actually will churn — correct catch!)
User 3  → model says: not churn  ✗  (actually will churn — missed!)
User 4  → model says: CHURN RISK ✗  (actually will stay  — false alarm!)
User 5  → model says: not churn  ✓  (actually will stay  — correct)
User 6  → model says: not churn  ✓  (actually will stay  — correct)
User 7  → model says: not churn  ✓  (actually will stay  — correct)
User 8  → model says: not churn  ✓  (actually will stay  — correct)
User 9  → model says: not churn  ✓  (actually will stay  — correct)
User 10 → model says: not churn  ✓  (actually will stay  — correct)
```

*(Assumed: these specific right/wrong guesses — just to have something concrete to calculate with.)*

---

## Step 3: Name the Four Outcomes

|  | Model says: CHURN | Model says: STAY |
|---|---|---|
| **Actually churns** | ✅ True Positive | ❌ False Negative |
| **Actually stays** | ❌ False Positive | ✅ True Negative |

In plain English:

| Name | What happened | Waze meaning |
|---|---|---|
| **True Positive** | Flagged AND actually churns | Caught a real churner — re-engagement nudge can prevent uninstall |
| **False Positive** | Flagged BUT actually stays | Sent a retention push to an active user who wasn't going to leave |
| **False Negative** | NOT flagged BUT actually churns | Missed a churner — they uninstall silently, no intervention possible |
| **True Negative** | NOT flagged AND actually stays | Correctly left a loyal active user alone |

From the example above:
- **Users 1, 2** → True Positives (flagged, actually churn)
- **User 4** → False Positive (flagged, actually stays — false alarm)
- **User 3** → False Negative (not flagged, actually churns — missed)

---

## Step 4: Precision

**Precision answers:** of everyone the model flagged, how many were actually churners?

The model flagged users 1, 2, and 4. That's 3 people flagged.
Of those 3, only 2 were real churners (users 1 and 2).

*(Calculated: 2 real churners out of 3 flagged.)*

```
Precision = real churners flagged ÷ everyone flagged
          = 2 ÷ 3
          = 67%
```

**Low precision** = lots of false alarms. You're sending retention nudges to many active users who had no intention of leaving.

---

## Step 5: Recall

**Recall answers:** of all the actual churners, how many did the model catch?

There were 2 actual churners: users 1 and 2. Wait — in our setup there were 2 churners but the model also missed user 3 (who actually churns). Let me re-check:

Actually in our example: Users 1, 2, and 3 all churn. User 3 is NOT flagged.

*(Corrected: 3 actual churners — users 1, 2, and 3. Model caught users 1 and 2, missed user 3.)*

```
Recall = real churners caught ÷ all actual churners
       = 2 ÷ 3
       = 67%
```

**Low recall** = lots of misses. Churning users are slipping through and uninstalling with no intervention.

---

## Step 6: The Tradeoff

You can't maximize both at the same time. Here's why:

If you lower the model's threshold (flag MORE users to catch more churners), **recall goes up** but **precision goes down** — you catch more real churners, but also flag more loyal active users as false alarms.

If you raise the threshold (only flag users the model is very confident about), **precision goes up** but **recall goes down** — fewer false alarms, but more real churners slip through and uninstall.

The right balance depends on the cost of each mistake — which brings us to why Waze's math is different from a subscription product.

---

## Step 7: Why the Cost Asymmetry Matters for Waze

In a subscription product like a music streaming service, the business sends a *discount offer* to flagged users. That costs real money — maybe $10 per user. So false positives are expensive.

In Waze, the retention intervention is a **push notification** — a feature highlight, a route tip, a "we miss you" message. Cost per send: essentially $0.

*(Assumed: push notification cost ≈ negligible direct cost. Re-acquisition cost of a lost user: ~$2–5 per user.)*

Scale up to **100 flagged users** and test different precision levels:

| Precision | Real churners in 100 flagged | Users retained (at 20% save rate) | Re-acquisition cost avoided | Push cost | Net benefit |
|---|---|---|---|---|---|
| 10% | 10 churners | 2 users saved | ~$10 | ~$0 | +$10 |
| 35% | 35 churners | 7 users saved | ~$35 | ~$0 | +$35 |
| 65% | 65 churners | 13 users saved | ~$65 | ~$0 | +$65 |

*(Calculated: retained users = churners in flagged group × 20% save rate. Re-acquisition cost = $5/user. Push cost ≈ $0.)*

The push is cheap — so why have any precision floor at all?

**Because over-notification causes its own churn.** If the model flags 80% of users as churn risk, Waze bombards active, happy users with retention messages. Research shows that unsolicited push notifications increase uninstall rates. The false positive becomes a self-fulfilling prophecy — you caused the churn you were trying to prevent.

The 35% precision floor is a product-safety constraint: enough signal that most flagged users are actually at risk, not just moderately less active than usual.

---

## Summary

| Term | One-line definition | Cost when it happens |
|---|---|---|
| **False Positive** | Flagged as churner, but was staying | Unwanted push → possible notification fatigue |
| **False Negative** | Not flagged, but actually churned | Uninstall with no intervention → re-acquisition cost (~$5) |
| **Precision** | % of flagged users who actually churn | Low precision = notification fatigue, potential secondary churn |
| **Recall** | % of actual churners you caught | Low recall = silent uninstalls, lost users |

**False negatives are more expensive than false positives** — which is why this model optimizes for recall (catching more at-risk users) with a precision floor (enough signal to avoid counter-productive over-notification).

The key difference from a subscription product: the *absolute cost* of both errors is smaller for Waze (no discounts, lower LTV per user). But the *relative* cost asymmetry is the same: missed churners are more expensive than false alarms.

---

*Part of the Waze Churn Prediction project — PMcurve Advanced AI for PMs | ML Foundations*
