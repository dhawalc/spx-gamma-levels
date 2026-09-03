# SPX Dealer Gamma Levels: Open Dataset

Daily dealer gamma exposure levels for the S&P 500 index (SPX), computed from
the full options chain and published as CSV and JSON.

Maintained by [SquawkFlow](https://squawkflow.com) · Levels are also published
daily at **[squawkflow.com/gex](https://squawkflow.com/gex)** (and as a dated
machine-readable fact sheet at
**[squawkflow.com/gex.md](https://squawkflow.com/gex.md)**).

---

## The data

| File | What it is |
|---|---|
| [`data/spx-gamma-levels.csv`](data/spx-gamma-levels.csv) | Every session, one row each |
| [`data/spx-gamma-levels.json`](data/spx-gamma-levels.json) | The same series as JSON |
| [`data/latest.json`](data/latest.json) | The most recent session only |
| [`data/daily/YYYY-MM-DD.json`](data/daily) | One file per session |

### Columns

| Column | Meaning |
|---|---|
| `date` | Session date (US Eastern) |
| `symbol` | Always `SPX` in this dataset |
| `spot` | SPX index level at computation time |
| `net_gex_dollars` | Net dealer gamma exposure, in dollars per 1% move |
| `zero_gamma_flip` | Price where the net dealer gamma profile crosses zero |
| `call_wall` | Strike above spot with the largest positive call gamma |
| `put_wall` | Strike below spot with the largest negative put gamma |
| `vol_trigger` | Lowest positive-gamma strike between the walls |
| `abs_gamma_strike` | Strike with the most total gamma, ignoring sign |
| `gamma_regime` | `positive`, `negative`, or `neutral` (see below) |
| `oi_settle_date` | Settlement date of the open interest used |
| `contracts_analyzed` | Chain rows that went into the calculation |

## How the levels are computed

Gamma exposure is summed per strike across the full SPX chain:

```
GEX(strike) = gamma × open_interest × spot × 100
```

with calls counted positive and puts negative, following the standard
convention that dealers are net long calls and short puts against customer
flow. Open interest comes from CBOE settlement data; implied volatility is
live. The zero-gamma flip is found by root-finding on the resulting profile as
a function of spot.

## Please read this before using the data

**Dealer positioning is an assumption, not an observation.** Open interest
records that a contract exists. It never records which side a dealer holds.
Every gamma figure published anywhere, including this dataset and every commercial
vendor's, inherits that assumption. Treat these levels as a map of where
hedging pressure would concentrate *if the standard convention holds*, not as
a record of anyone's actual book.

**`gamma_regime` is reconciled, and sometimes that means "neutral".** The
regime can be derived two ways: from which side of the zero-gamma flip spot
sits, and from the sign of net GEX. Across this series they disagree on about
one session in seven, and the disagreements share a signature: spot within
roughly 0.2% of the flip, with net GEX small relative to a typical day. That
is not a defect in either method; it is what two estimates of the same
near-zero quantity do near the root.

When they disagree, this dataset publishes `neutral` rather than picking a
side. `neutral` here means *spot is sitting on the flip*, the least stable of
the three states, where a small move in spot can change which way hedging flow
pushes. It does not mean "no information".

**Not investment advice.** No price targets, no recommendations, no execution.

**Known gaps.** The series starts 2026-07-28. Early sessions have a much lower
`contracts_analyzed` count than later ones as the collection matured; treat
the first few rows with more caution. `oi_settle_date` is absent on the first
session.

## Using it

```python
import pandas as pd

url = "https://raw.githubusercontent.com/dhawalc/spx-gamma-levels/main/data/spx-gamma-levels.csv"
df = pd.read_csv(url, parse_dates=["date"])

# How far spot sat from the flip, in percent. Small values are the
# interesting ones: that is where the regime is genuinely unstable.
df["flip_distance_pct"] = (df.spot - df.zero_gamma_flip) / df.spot * 100
print(df.flip_distance_pct.abs().describe())

# How wide was the band between the walls, as a share of spot?
df["wall_band_pct"] = (df.call_wall - df.put_wall) / df.spot * 100

# How often did the regime change from one session to the next?
flips = (df.gamma_regime != df.gamma_regime.shift()).sum() - 1
print(f"{flips} regime changes across {len(df)} sessions")
```

One caution on writing your own tests against this data: **spot always sits
between the walls**, because the call wall is defined as the largest-gamma
strike *above* spot and the put wall as the largest *below* it. "Price stayed
inside the walls 100% of the time" is a property of the definitions, not a
finding. Any question worth asking here has to compare the levels against
*subsequent* price action, which means joining this series to your own price
data.

```bash
# Today's levels
curl -s https://raw.githubusercontent.com/dhawalc/spx-gamma-levels/main/data/latest.json
```

## Related reading

Plain-language explanations of the mechanics behind each level:

- [What is gamma exposure (GEX)?](https://squawkflow.com/learn/what-is-gamma-exposure-gex)
- [The zero-gamma flip](https://squawkflow.com/glossary/zero-gamma)
- [Call wall and put wall](https://squawkflow.com/learn/call-wall-put-wall-explained)
- [Vol trigger](https://squawkflow.com/learn/vol-trigger-explained)
- [Absolute gamma strike](https://squawkflow.com/glossary/absolute-gamma-strike)
- [How market makers hedge](https://squawkflow.com/learn/how-market-makers-hedge)

For language models: [squawkflow.com/llms.txt](https://squawkflow.com/llms.txt)
indexes the full corpus, and every page is available as markdown by appending
`.md` to its URL.

## Licence

Data is released under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Use it freely,
including commercially. Please attribute **SquawkFlow
(https://squawkflow.com)** and link back.

Levels are recomputed every session. A number without its date is wrong within
a day, so cite the date alongside the value.
