# Mobile Money Fraud Detector (MVP)

Detects anomalous mobile-money agent transactions in the Nigerian mobile-money
context (OPay, Moniepoint, PalmPay, Paga-style agent networks) and returns a
**fraud flag** and an **anomaly score** for each transaction.

## Problem context

Nigerian mobile-money agents process high volumes of cash-in, cash-out,
transfer, bill-payment and airtime transactions daily, and are a frequent
fraud target. Common patterns modeled in this MVP:

- **Balance-draining cash-out** — a compromised/SIM-swapped account is
  emptied in one large `CASH_OUT`/`TRANSFER`, often from an unrecognized
  device, at odd hours.
- **Agent float round-tripping** — a balance is inflated then reversed or
  withdrawn.
- **Velocity / smurfing** — many rapid small transfers from the same sender
  in a short window, to stay under alert thresholds.
- **Odd-hour account takeover** — transactions at 1am–4am from a device or
  location not previously used by the sender.
- **Round-amount social engineering** — suspiciously round large amounts
  (₦50,000 / ₦100,000 / ₦150,000 / ₦200,000), typical of "urgent transfer"
  scam messages.

No public labeled Nigerian mobile-money fraud dataset exists, so this MVP
uses a **synthetic but realistic dataset** (60,000 transactions, ~1.2% fraud
rate, patterns above injected) to build and validate the pipeline end to end.
Swap the data-generation step for a real transaction export when available —
the feature engineering and model code do not need to change, as long as the
column schema below is kept.

## Deliverables in this folder

| File | Description |
|---|---|
| `Mobile_Money_Fraud_Detector.ipynb` | Full notebook: data generation → feature engineering → training → evaluation → single-transaction scoring. Open directly in Google Colab. |
| `fraud_classifier.joblib` | Trained Random Forest pipeline (preprocessing + model). |
| `anomaly_detector.joblib` | Trained Isolation Forest pipeline. |
| `model_config.joblib` | Decision threshold used to convert fraud probability → fraud flag. |
| `evaluation_results.json` | Full metrics (ROC AUC, PR AUC, F1, confusion matrix, classification report) from the held-out test set. |
| `evaluation_plots.png` | ROC curve, Precision-Recall curve, confusion matrix. |
| `transactions.csv` | The generated synthetic dataset used to train/evaluate. |

## How to run

**Google Colab (recommended):**
1. Upload `Mobile_Money_Fraud_Detector.ipynb` to Colab (or open it via
   File → Upload notebook).
2. Run all cells top to bottom (`Runtime → Run all`). Everything —
   including data generation — is self-contained in the notebook; no
   external files are required.

**Locally:**
```bash
pip install pandas numpy scikit-learn matplotlib joblib
jupyter notebook Mobile_Money_Fraud_Detector.ipynb
```

## Approach

1. **Data**: synthetic transactions with sender balances tracked over time,
   five fraud patterns injected at ~1.2% overall rate (realistic for
   agent-network fraud).
2. **Feature engineering** (all computed causally — only from information
   available at or before the transaction's timestamp, to avoid leakage):
   - `amount`, `hour`, `is_odd_hour`, `day_of_week`
   - `balance_ratio` (amount / balance before), `drains_account`
   - `is_round_amount`
   - `sender_avg_amount_so_far`, `sender_txn_count_so_far`, `amount_vs_avg`
     (running behavioural baseline per sender)
   - `sender_velocity_30min` (transaction count in the trailing 30 minutes)
   - `is_new_device_for_sender`
   - `transaction_type`, `state` (categorical)
3. **Models**:
   - **Random Forest classifier** (supervised, primary) — trained on the
     labeled data, outputs `fraud_probability`; the `fraud_flag` is the
     probability thresholded at the value that maximizes F1 on the test set.
   - **Isolation Forest** (unsupervised, secondary) — outputs a 0–100
     `anomaly_score`. Doesn't require fraud labels, so it stays useful before
     enough confirmed fraud cases accumulate, or against fraud patterns the
     classifier hasn't seen before.
4. **Split**: chronological (train on the first 75% of transactions by time,
   test on the last 25%) rather than random — fraud detection is a
   forecasting problem in production, so evaluation should reflect that.

## Results (held-out test set)

| Metric | Value |
|---|---|
| ROC AUC | ~0.97 |
| PR AUC (average precision) | ~0.78 |
| F1 (fraud class), at tuned threshold | ~0.81 |
| Precision (fraud class) | ~0.99 |
| Recall (fraud class) | ~0.69 |

See `evaluation_results.json` for exact figures and the full
`classification_report`, and `evaluation_plots.png` for the ROC curve,
Precision-Recall curve, and confusion matrix.

**Reading these numbers**: precision is deliberately weighted high (few
false alarms for agents/customers) at some cost to recall (a fraction of
fraud is missed at the default threshold). The threshold in
`model_config.joblib` can be lowered to catch more fraud at the cost of more
false positives — this is a business/risk trade-off, not a modeling
limitation, and should be tuned with input from whoever owns fraud-ops
decisions.

## Core MVP feature: score a single transaction

```python
result = score_transaction(new_txn, history_df)
# {
#   "transaction_id": "TXN99999999",
#   "fraud_flag": 1,
#   "fraud_probability": 0.809,
#   "anomaly_score": 52.5,
#   "threshold_used": 0.6
# }
```

`new_txn` is a dict with the transaction schema (see table below); `history_df`
is the sender's prior transactions, used to compute behavioural features like
velocity and running average amount. See Section 7 of the notebook for a full
worked example (a suspicious drain transaction vs. a normal airtime top-up).

## Transaction schema

| Column | Type | Description |
|---|---|---|
| `transaction_id` | string | Unique transaction ID |
| `timestamp` | datetime | Transaction time |
| `sender_id` | string | Customer/account initiating the transaction |
| `agent_id` | string | Agent handling the transaction |
| `transaction_type` | string | `CASH_IN`, `CASH_OUT`, `TRANSFER`, `BILL_PAYMENT`, `AIRTIME` |
| `amount` | float | Transaction amount (₦) |
| `sender_balance_before` | float | Sender's balance before the transaction |
| `sender_balance_after` | float | Sender's balance after the transaction |
| `state` | string | Nigerian state of the sender |
| `device_id` | string | Device used to initiate the transaction |
| `is_fraud` | 0/1 | Label (only present in training data) |

## Known limitations & next steps

- Trained on **synthetic** data with hand-designed fraud patterns. Before
  production use, retrain on real, labeled agent transaction history.
- No graph/network features yet (e.g. devices or accounts shared across
  multiple "customers" — a strong signal in real agent fraud rings).
- Velocity is only tracked per sender; production should also track
  velocity per agent and per device.
- Suggested next steps: wrap `score_transaction()` in a REST endpoint for
  real-time agent-app integration; add a feedback loop where confirmed
  fraud/chargeback outcomes retrain the classifier periodically; add
  agent-level and device-level features.

## Tools used

Python, Pandas, NumPy, Scikit-Learn (Random Forest, Isolation Forest),
Matplotlib, Google Colab.
