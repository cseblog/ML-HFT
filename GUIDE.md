# Beginner's Guide to This Project

A plain-English walkthrough of every notebook/script, for someone new to ML.

## The whole project as one picture

```
 RAW EXCHANGE FEED                          (data/CN_Futures_2014.01.02.csv)
 every order add/cancel/trade message
            │
            ▼
 ┌─────────────────────────┐
 │ or_transfomer.py         │  rebuild the order book from messages
 │ data_process.ipynb       │  → data/order_book_3_2014_1_2.csv
 └─────────────────────────┘
            │
            ├───────────────► data_visualization.ipynb   (just LOOK at the data, plots)
            │
            ▼
 ┌─────────────────────────┐
 │ HFT_factors.ipynb        │  INVENT + test the signals & the label (research lab)
 │ feature_engineering.ipynb│  apply them at scale → processed_data/*_UP.csv / *_DOWN.csv
 └─────────────────────────┘
            │
            ▼
 ┌─────────────────────────┐
 │ model_fitting.ipynb      │  train ML on features → predict buy/flat → measure accuracy & PnL
 └─────────────────────────┘
```

Read it left→right = raw market data slowly turns into a trading decision.

---

## 1. `or_transfomer.py` — rebuild the order book

The exchange sends a stream of tiny events, not a neat table:

```
t=0.001  ADD   BUY  10 @ 100.0
t=0.002  ADD   SELL  8 @ 100.2
t=0.005  CANCEL BUY  4 @ 100.0
t=0.007  TRADE      5 @ 100.2
```

This script **replays** them to keep a running picture = the **order book**.

```
   messages (diary of changes)          →     order book snapshot (current state)
   ADD BUY 10@100.0                            ask1 100.2   8
   ADD SELL 8@100.2          replay            ───────────────
   CANCEL BUY 4@100.0    ──────────────►       bid1 100.0   6   (10 added − 4 cancelled)
```

Key functions: `first_order_create()` builds the first book; the loop applies later
messages and writes each snapshot. (Demo processes only 100 rows.)

**Output:** `order_book_*.csv`.

---

## 2. `data_process.ipynb` — same job, notebook version

Same reconstruction logic, interactive. Produces the full `order_book_3_2014_1_2.csv`.
The order book per tick is a ladder:

```
        PRICE    QTY
 ask3  100.3     40
 ask2  100.2     25
 ask1  100.1     10     ← best ask
 ─────────────────────  ← spread = ask1 − bid1
 bid1  100.0     15     ← best bid
 bid2   99.9     30
 bid3   99.8     50
```

Three levels each side. Everything later is computed from these numbers.

---

## 3. `data_visualization.ipynb` — look before you model

No new data. Just plots to build intuition and catch problems (EDA).

```
 best bid / ask over the day        spread distribution
 price                              count
  │      ___/‾‾\__                   │    ▁▃█▇▅▂
  │   __/        \___                │   ▁█████▇▃▁
  │__/               \_              │  ▁████████▂
  └────────────────────► time       └──────────────► spread size
```

Also converts timestamps to "seconds since 09:00" so morning `[0…9000]` = 09:00–11:30
and afternoon `[14400…25200]` = 13:00–16:00.

---

## 4. `HFT_factors.ipynb` — research lab: invent the signals & the label

The most important notebook. Designs the model's inputs (features) and answer (label).

### Three signal families

**(a) Depth Ratio** — who has more size?
```
        bid_qty
ratio = ───────     bigger bids → buyers pushing → price likely UP
        ask_qty
```

**(b) Order Book Imbalance (OBI)** — same idea, squashed to −1…+1:
```
       ask_qty − bid_qty
OBI =  ─────────────────     (project's sign convention)
       ask_qty + bid_qty
```

**(c) Rise Ratio** — momentum = slope of the best-ask price recently.
```
ask price
  │        ●        slope > 0 → rising
  │     ●           slope < 0 → falling
  │  ●
  └──────────► time
```

Then **weighted multi-level** versions combine levels 1/2/3 with weights like `910`
(0.9·L1 + 0.1·L2 + 0·L3), `820`, `550`… → about **64 features per second**.

### The label (the answer the model learns)

```
              now
               │   look forward "traded_time" (e.g. 15 min)
   ────────────┼──────────────────────────────►  time
               │   find the MINIMUM ask in this window
               ▼
   if  current_bid_price  >  min(future_ask) :
            label = 1   →  BUY  (you'd profit)
   else:
            label = 0   →  STAY FLAT
```

Built by `traded_label_micsecond(...)`.

> ⚠️ The label legally uses the future (to know the right answer for training).
> The *features* must NOT — that would be look-ahead bias (cheating).

---

## 5. `feature_engineering.ipynb` — apply the recipes at scale

Runs the signal recipes over the whole day, morning + afternoon, saves a tidy table.

```
 data/order_book_3_2014_1_2.csv
            │   compute all 64 signals per second + attach the buy/flat label
            ▼
 processed_data/order_book_3_2014_new_1_2_UP.csv     ← morning ("UP")
 processed_data/order_book_3_2014_new_1_2_DOWN.csv   ← afternoon ("DOWN")
```

Column meanings in the saved table:

```
 col '0'            = label (0/1)        ← what we predict
 col '1'…'64'       = the 64 features    ← what we feed the model
 col '65','66','67' = prices/spread      ← used ONLY for profit calc, dropped before training
```

Written by `train_test_to_csv()`. **These files only exist after you run this notebook**
(skipping it is why model_fitting throws `FileNotFoundError`).

---

## 6. `model_fitting.ipynb` — the actual machine learning

A binary classification problem: features → predict 0 or 1.

Models: RandomForest, ExtraTrees, AdaBoost, GradientBoosting, SVM.

**Rolling window** (never train on the future):

```
 time ───────────────────────────────────────────────►
 [■■■■■■■■■■ train 30 min ■■■■■■■■■■][▦ test 10s]
        slide →
            [■■■■■■■■■■ train 30 min ■■■■■■■■■■][▦ test 10s]
                slide →
                    [■■■■■■■■■■ train 30 min ■■■■■■■■■■][▦ test 10s]
```

Each window: `GridSearchCV` (5-fold) tunes every model → pick best by CV accuracy →
predict next 10s. Then measure Accuracy, F1, CV mean accuracy, and a PnL (equity) curve
via `equity_curve_with_long_at_close()`.

```
 equity
   │        ___/‾
   │    __/
   │  _/
   └──────────────► rolling windows
```

---

## Cheat-sheet

| Notebook | One line | Makes |
|----------|----------|-------|
| `or_transfomer.py` / `data_process` | messages → order book | `order_book_*.csv` |
| `data_visualization` | look at the data | plots only |
| `HFT_factors` | invent signals + label (research) | plots, logic |
| `feature_engineering` | apply at scale | `processed_data/*_UP/DOWN.csv` |
| `model_fitting` | train, predict, score, PnL | accuracy + equity curve |

**Run order:** `data_process` → `feature_engineering` → `model_fitting`.

---

## Appendix: one signal end-to-end — OBI

**Formula** (project convention):
```
       ask_qty1 − bid_qty1
OBI =  ───────────────────
       ask_qty1 + bid_qty1
```

**Real code** (`HFT_factors.ipynb`, level-1 plot cell):
```python
# Depth ratio (level 1)
ask_quantity_1 / bid_quantity_1
# OBI (level 1)
(ask_quantity_1 - bid_quantity_1) / (ask_quantity_1 + bid_quantity_1)
```

**Weighted multi-level version** (`feature_engineering.ipynb`):
```python
def weight_pecentage(w1, w2, w3, ask_quantity_1, ask_quantity_2, ask_quantity_3,
                                  bid_quantity_1, bid_quantity_2, bid_quantity_3):
    Weight_Ask = (w1 * ask_quantity_1 + w2 * ask_quantity_2 + w3 * ask_quantity_3)
    Weight_Bid = (w1 * bid_quantity_1 + w2 * bid_quantity_2 + w3 * bid_quantity_3)
    W_AB  = Weight_Ask / Weight_Bid                                 # weighted depth ratio
    W_A_B = (Weight_Ask - Weight_Bid) / (Weight_Ask + Weight_Bid)   # weighted OBI
    return W_AB, W_A_B
```

**How to read it:** OBI > 0 → more size resting on the ask side (sell pressure);
OBI < 0 → more size on the bid side (buy pressure); OBI ≈ 0 → balanced. Each weight
tuple (`100`, `910`, `820`, …) is just a different blend of levels 1/2/3, giving the
model many views of the same imbalance idea.
