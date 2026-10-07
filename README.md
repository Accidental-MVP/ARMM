# ARMM — Adaptive Regime Market Maker

A market maker that decides what kind of market it is in before it decides how to quote.
Built solo in 24 hours for the **National Bank Challenge**, where it placed **2nd**.

> **Credit first.** The exchange simulator (`app/`), the market scenarios, `manual_trader.py`,
> `API_REFERENCE.md` and the original project skeleton were provided by the competition
> organisers. My work is the strategy in `student_algorithm.py` — roughly 1,100 of its
> ~1,260 lines. This README describes that strategy, not the harness.

---

## The idea

A fixed spread is correct in exactly one market regime and wrong in every other. A quiet
book, a trending book and a dislocation each punish the same quoting policy differently, so
the policy has to know which one it is looking at.

ARMM classifies the book into three states on every snapshot and converts that state into a
**target inventory**, which the execution layer then works toward:

| State | Meaning | Response |
|---|---|---|
| `BAND` | Mean-reverting inside a rolling range | Quote both sides, lean on reversion |
| `DRIFT` | Persistent directional movement | Skew with the trend, reduce adverse fills |
| `EVENT` | Jump or dislocation (flash-type move) | Cut exposure, widen, wait out the cooldown |

## How it works

**Features** (`FeatureEngine`) — the band comes from **rolling quantiles** (5th/95th) rather
than min/max, so a single print cannot distort the range. Volatility is a robust statistic:
`min(mean, median, EWMA)` of absolute mid changes, floored so it never collapses to zero.
Trend is an EMA over mid deltas, not raw tick noise.

**State detection** (`StateDetector`) — the part that took the longest. A naive classifier
flips between states on noise and the strategy spends the round whipsawing itself, so:

- `DRIFT` requires **EMA trend persistence**, not a single directional tick
- `EVENT` fires only on a genuine dislocation — a jump **and** spread widening together
- `EVENT` carries a hold period and a **cooldown**, so one shock cannot retrigger repeatedly
- **Hysteresis** throughout; `BAND` is the default it falls back to

**Inventory as a control variable** (`TargetInventoryPolicy`, `TargetShaper`) — inventory is
not a number you report at the end, it is the thing you steer. State maps to a target
position, bounded by per-state caps (`EVENT` gets a tighter cap than `BAND`), then
`TargetShaper` **slew-rate-limits** movement toward that target so the strategy cannot chase
its own signal. This was the single change that most improved behaviour under the stressed
and flash-crash scenarios.

**Execution and quoting** (`ExecutionEngine`, `QuoteEngine`) — lot rounding and price
rounding to exchange rules, inventory limits enforced before an order is constructed rather
than after, **self-cross prevention**, and cancellation of contra orders before placing a
quote that would trade against them. Quotes are only replaced when the new price differs
enough to be worth the message.

**Per-scenario tuning** — `hft_dominated` runs different `EVENT` thresholds from the other
scenarios, because what counts as a dislocation depends on the baseline.

## Running it

Requires the organisers' simulator in `app/`.

```bash
pip install -r requirements.txt
python student_algorithm.py --name <team> --password <password> --scenario normal_market
```

Scenarios: `normal_market`, `stressed_market`, `flash_crash`, `hft_dominated`,
`mini_flash_crash`.

## What I would change

The thresholds are hand-tuned per scenario, which works for a 24-hour competition and would
not survive contact with a real book. The honest next step is fitting them, with a walk-forward
split so the regime classifier is evaluated on data it was not tuned against — right now there
is nothing separating "the classifier works" from "the constants were fitted to five scenarios."

---

Built by [Uday Parmar](https://uday-parmar.vercel.app) · [More work](https://uday-parmar.vercel.app)
