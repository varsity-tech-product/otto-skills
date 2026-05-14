---
name: quantai-service
description: >
  Crypto single-factor quant research service skill. Load this skill when users ask to
  write a factor, research a factor, auto-discover/mine factors, submit a factor, or run
  factor backtests. The Agent writes factor plugin code and interacts with the service
  over HTTP. The server handles all data processing and computation; the Agent does not
  need any local market data.
---

# QuantAI Service — Agent Runbook

## Service Endpoint

```
BASE_URL = http://quantai-alb-396163355.ap-southeast-1.elb.amazonaws.com
```

> Before starting, run `curl ${BASE_URL}/health` to confirm the service is online. If the connection fails, notify the user immediately.

---

## ⛔ Forbidden Actions

1. **Do not run any quant-factor-loop scripts locally** (`run_workflow.py`, `step*.py`, etc.). All computation runs on the server.
2. **Do not download or store parquet data files**. Do not attempt to access data files on the server.
3. **Do not modify any file under `/home/ec2-user/quant-factor-loop/`**. That directory belongs to the server internals only.
4. **Do not wait forever when polling**: max 30 minutes per Job; on timeout, notify the user and stop.
5. **Do not skip retest**: when Step 4C C# compilation fails, fix `strategy.cs` and retest. Do not declare failure directly.
6. **Do not wait for parameter search**: as soon as Step 4C completes, present the default-parameter results to the user and end this round. Steps 5–16 are the server-side async parameter-search pipeline — irrelevant to the user, do not poll.

---

## Agent Local Paths

Record these paths into working variables at the start of each new task:

```
./quant_agent/
├── factor_registry.jsonl         ← factor archive index (factor_type + formula + job_id)
└── jobs/
    └── {job_id}/
        ├── plugin.py                  ← uploaded factor plugin (saved after phase 2)
        ├── strategy.cs                ← C# strategy from Step 4a (download when strategy_cs_ready=true)
        ├── factor_card_default.json   ← Step 4C default-parameter factor card (download after Step 4C)
        ├── factor_card_default.txt    ← human-readable txt version of the same card (optional)
        └── step4c/
            ├── equity_curves.png      ← Step 4C default-parameter equity curve
            ├── ts_profile_4panel.png  ← TS 4-in-1 time-series profile (new)
            ├── trade_log.csv          ← Step 4C default-parameter trade log
            ├── group_return_plot.png  ← CS grouped cumulative return plot (may not exist)
            ├── cs_profile_4panel.png  ← CS 4-in-1 cross-section profile (may not exist)
            └── cs_nav_curves.png      ← CS NAV curves: long / short / long-short (may not exist)
```

> **Note**: the server runs two cloud backtests internally — Step 4C (default params) and Step 11 (tuned params). The Agent only downloads and presents the **default-parameter** version (`default_` prefix files) to the user. The tuned version stays on the server.

---

## Internal Server Pipeline (Reference Only — Agent Doesn't Operate It)

After job submission, the server runs the following automatically. **The Agent only polls and waits**:

```
Step 1-3   load config, compute forward returns, set exit rule
Step 4A    generate C# strategy code (strategy.cs)
Step 4B    compute raw signal (Python research mirror)
Step 4L    future-data leakage check (custom factors only, 5 random timestamps)
Step 4C    default-parameter cloud backtest  ← user-facing results come from here
Step 5-11  Agent does not need to care
```

> Agent intervention cases:
> - C# compile failure (`failed_step="4c"`) → fix strategy.cs → retest
> - Future-data leakage (`failed_step="4l"`) → Python plugin `build_signal` contains a future function; rewrite the factor and submit a **new job** (retest is not allowed)
>
> Once Step 4C finishes, the Agent can fetch results and present them to the user. **No need to wait for Step 5 or beyond.**

---

## Agent Workflow (Must Follow Order)

### Phase -1: Choose Work Mode

At each session start, **the first thing** is to ask the user which work mode to use:

```
This session, choose:
A) Auto-Discovery Mode — AI autonomously picks tasks from the server's published task pool and designs the implementation path on its own
B) Free Mode           — you directly describe the factor you want to research

(Enter A / B, or directly describe a factor idea)
```

**Position-strategy mode is no longer asked** — every factor automatically submits two jobs (sigmoid_continuous + quantile_discrete), and both modes' results are presented side by side.

> Save the work mode to variable `WORK_MODE` (value `A` / `B`); later phases depend on this variable.

#### A. Auto-Discovery Mode

1. Fetch the task list:

```bash
curl -s ${BASE_URL}/tasks
```

Example output:
```json
[
  {"task_id": "task_momentum_001", "title": "Short-Term Price Momentum", "category": "momentum"},
  {"task_id": "task_volume_001",   "title": "Aggressive Volume Imbalance", "category": "volume"}
]
```

2. **AI autonomously picks one task**: prefer a category not yet (or rarely) studied based on `factor_registry.jsonl`. Do not ask the user. After picking, immediately tell the user which task was chosen (show `task_id` and `title`), then continue.

3. Fetch the full description of the chosen task:

```bash
TASK_ID="task_momentum_001"
curl -s ${BASE_URL}/tasks/${TASK_ID}
```

4. Read `description` and `hints`, **AI autonomously decides the technical path** (no need to ask the user about implementation details), and proceeds directly to **phase 0b** to extract task info. When entering phase 0b, use the `fwd_period` value from the task JSON (default 7).

> Note: tasks are always in `open` state, multiple AIs can research the same task in parallel — the more diverse the results the better. **There is no "claimed" state.**

#### B. Free Mode

The user directly describes the factor logic; proceed to **phase 0b** (existing flow, unchanged).

---

### Phase 0: Confirm Task

**0a. Dedup Check — Inspect Already-Researched Factors**

Before writing any code, run dedup against `factor_registry.jsonl` (auto-initialize on first run):

```bash
REGISTRY=./quant_agent/factor_registry.jsonl
mkdir -p ./quant_agent

# First-run init: build registry from historical jobs
# Dedup rule: same factor_type+formula within the same minute and <= 10 seconds apart
# is treated as a single SIG/QD dual-submit round; keep only one entry.
if [ ! -s "$REGISTRY" ]; then
  TMP=$(mktemp)
  for f in ./quant_agent/jobs/job_*/plugin.py; do
    [ -f "$f" ] || continue
    job_id=$(basename "$(dirname "$f")")
    factor_type=$(sed -n 's/^FACTOR_TYPE[[:space:]]*=[[:space:]]*"\([^"]*\)".*/\1/p' "$f" | head -n 1)
    formula=$(sed -n 's/^[[:space:]]*"__FACTOR_FORMULA__":[[:space:]]*"\(.*\)",[[:space:]]*$/\1/p' "$f" | head -n 1)
    [ -n "$factor_type" ] || continue
    ts=$(printf '%s' "$job_id" | sed -n 's/^job_\([0-9]\{8\}\)_\([0-9]\{6\}\)_.*/\1\2/p')
    [ -n "$ts" ] || continue
    minute=${ts%??}
    second=${ts#????????????}
    printf '%s\t%s\t%s\t%s\t%s\n' "$factor_type" "$formula" "$minute" "$second" "$job_id" >> "$TMP"
  done

  sort -t $'\t' -k1,1 -k2,2 -k3,3 -k4,4n "$TMP" | awk -F '\t' '
  function esc(s){gsub(/\\/,"\\\\",s); gsub(/"/,"\\\"",s); gsub(/\r/," ",s); gsub(/\n/," ",s); return s}
  function emit(ft, fm, jb){print "{\"factor_type\":\"" esc(ft) "\",\"formula\":\"" esc(fm) "\",\"job_id\":\"" jb "\"}"}
  {
    key=$1 "\t" $2 "\t" $3
    sec=$4+0
    job=$5
    if (!(key in has)) {
      has[key]=1; psec[key]=sec; pjob[key]=job; pft[key]=$1; pfm[key]=$2
      next
    }
    if ((sec - psec[key]) <= 10) {
      emit(pft[key], pfm[key], pjob[key])
      delete has[key]; delete psec[key]; delete pjob[key]; delete pft[key]; delete pfm[key]
    } else {
      emit(pft[key], pfm[key], pjob[key])
      has[key]=1; psec[key]=sec; pjob[key]=job; pft[key]=$1; pfm[key]=$2
    }
  }
  END{
    for (k in has) emit(pft[k], pfm[k], pjob[k])
  }' > "$REGISTRY"

  rm -f "$TMP"
fi

# Read archive (Agent decides whether the new factor is "essentially duplicate", not limited to factor_type)
cat "$REGISTRY"
```

Example single-line record:

```
{"factor_type":"rsi_oversold_bounce","formula":"RSI < oversold -> long; RSI > overbought -> short","job_id":"job_20260312_153001_f4a2c1"}
```

- If the new logic is **essentially identical** to a historical record (only params differ), tell the user the factor already exists, show the historical `job_id`, and ask whether to rerun with new params or proceed as new.
- If the logic differs substantively (e.g. RSI oversold vs RSI divergence), continue normally.

**0b. Extract Task Info**

From the user's description, extract:

| Info | Description | Example |
|------|------|------|
| Factor logic | Describe in plain language how the signal is generated | "Long when RSI is below 30" |
| Core params | Window, thresholds, etc. | `rsi_period=14, oversold=30` |
| `factor_type` | Factor type identifier (snake_case, globally unique) | `rsi_oversold_bounce` |
| `factor_name` | Factor name (includes major param values) | `rsi_14_ob30` |

If the user's description is unclear, follow up on these items first; confirm before writing code.

---

### Phase 1: Write the Factor Plugin plugin.py

The plugin file contains two parts that **must both be implemented and have completely identical logic**:

1. **`FACTOR_SECTIONS`**: C# code fragments — the server uses them to generate `strategy.cs`
2. **`build_signal()`**: Python function — the server uses it for hyperparameter grid search

#### Plugin File — Full Template

```python
import pandas as pd
import numpy as np
from typing import Any, Dict

FACTOR_TYPE = "<factor_type>"   # must match the factor_type passed at submission

FACTOR_DEFAULT_PARAMS = {
    "param1": <default_int>,    # all hyperparams and their defaults; keys in snake_case
}

FACTOR_SECTIONS = {
    # ── Comment fields (human-readable) ─────────────────────────────────
    "__FACTOR_DESCRIPTION__": "Factor description in English",
    "__FACTOR_FORMULA__":     "Signal formula (used in comments)",
    "__FACTOR_TYPE__":        "<factor_type>",

    # ── C# class field declarations (every line must end with \n) ───────
    "__FACTOR_PARAM_FIELDS__": (
        "        private int _param1;\n"
        # one field per line, note the 8-space indent
    ),

    # ── C# constructor init (every line must end with \n) ───────────────
    "__FACTOR_INIT__": (
        '            _param1 = GetIntParameter("param1", <default>);\n'
        # keys use kebab-case ("param-one"); the corresponding Python key uses snake_case ("param_one")
    ),

    # ── Init log (every line must end with \n) ──────────────────────────
    "__FACTOR_LOG__": (
        '            Log($"[INIT] param1={_param1}");\n'
    ),

    # ── Sliding window size (valid C# integer expression, no quotes) ────
    "__PRICE_WINDOW_EXPR__": "_param1 + 1",

    # ── Extra-column buffer declarations (line ends with \n; "" if only close is used) ──
    # FactorCsvBar field types (determine whether casts are needed at Enqueue):
    #   decimal : Open / High / Low / Close / Volume
    #             → must write (double)bar.Volume etc. at Enqueue, otherwise CS1503 compile error
    #   double  (base): TakerBuyVolume / TakerSellVolume / TakerBuyQuoteVolume /
    #                  TakerSellQuoteVolume / TakerBuyTrades / TakerSellTrades / QuoteVolume
    #   double  (extra futures / on-chain columns):
    #     OpenInterestOpen / OpenInterestHigh / OpenInterestLow / OpenInterestClose
    #     FundingRateOpen / FundingRateHigh / FundingRateLow / FundingRateClose
    #     GlobalAccountLongPercent / GlobalAccountShortPercent / GlobalAccountLongShortRatio
    #     TopPositionLongPercent / TopPositionShortPercent / TopPositionLongShortRatio
    #     TopAccountLongPercent / TopAccountShortPercent / TopAccountLongShortRatio
    #     LiquidationLongUsd / LiquidationShortUsd
    #     BinancePremiumIndexOpen / BinancePremiumIndexHigh / BinancePremiumIndexLow / BinancePremiumIndexClose
    #     → All double, Enqueue directly, no cast needed.
    #     → Older coins / early dates may be NaN (structural data gaps).
    #       The strategy template auto-injects a NaN scan over every array
    #       declared in __EXTRA_BUF_TOARRAY__ before your compute body runs,
    #       so a NaN value transparently causes the symbol to skip that day.
    #       You do NOT need to write `if (double.IsNaN(x)) return false;` yourself.
    "__EXTRA_BUF_FIELDS__": "",   # e.g. '        private readonly Queue<double> _volBuf = new Queue<double>();\n'

    # ── Per-bar enqueue for extra columns (line ends with \n; "" if unused) ──
    # decimal field example: '            _volBuf.Enqueue((double)bar.Volume);\n'
    # double  field example: '            _takerBuyBuf.Enqueue(bar.TakerBuyVolume);\n'
    "__EXTRA_BUF_ENQUEUE__": "",

    # ── Dequeue when over window (line ends with \n; "" if unused) ───────
    "__EXTRA_BUF_DEQUEUE__": "",  # e.g. '            if (_volBuf.Count > requiredBars) _volBuf.Dequeue();\n'

    # ── Convert extra columns to arrays for compute body (line ends with \n; "" if unused) ──
    "__EXTRA_BUF_TOARRAY__": "",  # e.g. '            var volumes = _volBuf.ToArray();\n'

    # ── C# signal compute body ──────────────────────────────────────────
    # Always available: prices[] (close, oldest to newest)
    # If __EXTRA_BUF_TOARRAY__ is declared, those arrays are also available here
    # Must assign rawSignal (positive = long, negative = short) and return true
    # Return false when data is insufficient (do not throw)
    "__FACTOR_COMPUTE_BODY__": """
            // C# compute logic
            var n = prices.Length;
            if (n < _param1) return false;
            // ... compute ...
            rawSignal = <signal_value>;
            return true;
""",
}


def build_signal(
    close: pd.DataFrame,
    params: Dict[str, Any],
    # Declare the columns this factor uses (the framework injects them, equal status to close):
    # ── base kline (11) ──
    # open, high, low, volume,
    # taker_buy_volume, taker_sell_volume,
    # taker_buy_quote_volume, taker_sell_quote_volume,
    # taker_buy_trades, taker_sell_trades,
    # quote_volume  ← computed column = taker_buy_quote_volume + taker_sell_quote_volume
    # ── futures / on-chain (23) ──
    # open_interest_open / open_interest_high / open_interest_low / open_interest_close
    # funding_rate_open / funding_rate_high / funding_rate_low / funding_rate_close
    # global_account_long_percent / global_account_short_percent / global_account_long_short_ratio
    # top_position_long_percent / top_position_short_percent / top_position_long_short_ratio
    # top_account_long_percent / top_account_short_percent / top_account_long_short_ratio
    # liquidation_long_usd / liquidation_short_usd
    # binance_premium_index_open / binance_premium_index_high / binance_premium_index_low / binance_premium_index_close
    # NOTE: older coins / early dates can be NaN. rolling/sum auto-skips NaN,
    #       but for cross-section ops (mean, std, rank) call .dropna() / .notna() first
    #       so a few NaN cells don't contaminate the whole column.
    **_kwargs,
) -> pd.DataFrame:
    """
    close  : pd.DataFrame, index = UTC DatetimeIndex, columns = symbol codes
    params : dict, keys aligned with FACTOR_DEFAULT_PARAMS
    return : DataFrame of the same shape as close; positive = long, negative = short, NaN = no signal
    Logic must be exactly identical to FACTOR_SECTIONS.__FACTOR_COMPUTE_BODY__
    """
    param1 = int(params.get("param1", <default>))
    # ... Python implementation ...
    return signal.reindex_like(close)
```

#### Python build_signal Constraints (violations cause Step 4L future-data check to fail)

| Constraint | Reason |
|------|------|
| Forbid `shift(-n)` | Negative shift reads future prices — most common leakage source |
| Forbid `pct_change(periods=-n)` | Negative period, same as above |
| Forbid `fillna(method='bfill')` | Backward fill uses future values |
| Forbid full-series stats normalization | `(x - x.mean()) / x.std()` over the whole column contains future data |
| Forbid `.rolling(n).mean().shift(-k)` | Rolling then negative shift |
| `shift` must use a positive integer | `shift(1)` looks back into history — safe |

#### C# Code Constraints (violations cause Step 4C compile failure)

| Constraint | Reason |
|------|------|
| `__PRICE_WINDOW_EXPR__` | Must be a pure C# integer expression, no quotes, e.g. `_window + 1` |
| `rawSignal` | Must be assigned in `__FACTOR_COMPUTE_BODY__` |
| Insufficient data | `return false`; do not `throw` or `return true` without assignment |
| Types | Use `double` for all calculations; never `decimal` or `float` |
| Forbidden API calls | `Securities[].GetLastData()`, `Portfolio`, `Order`, `SetHoldings` |
| Parameter keys | `GetIntParameter("param-name", default)` uses kebab-case |
| Line endings | `__FACTOR_PARAM_FIELDS__` / `__FACTOR_INIT__` / `__FACTOR_LOG__` / `__EXTRA_BUF_FIELDS__` / `__EXTRA_BUF_ENQUEUE__` / `__EXTRA_BUF_DEQUEUE__` / `__EXTRA_BUF_TOARRAY__` — every line ends with `\n` |
| When no extra columns | `__EXTRA_BUF_FIELDS__` / `__EXTRA_BUF_ENQUEUE__` / `__EXTRA_BUF_DEQUEUE__` / `__EXTRA_BUF_TOARRAY__` are empty strings `""` |
| Extra-column array length | Extra-column buffers use the same `requiredBars` window size as close |
| **decimal → double cast** | `Open`/`High`/`Low`/`Close`/`Volume` are `decimal`; at Enqueue you must write `(double)bar.Volume` or you get `CS1503`. `TakerBuy*/TakerSell*/QuoteVolume` and all futures/on-chain columns (`OpenInterest*` / `FundingRate*` / `GlobalAccount*` / `TopPosition*` / `TopAccount*` / `Liquidation*` / `BinancePremiumIndex*`) are already `double` — no cast needed |
| **NaN handling for futures/on-chain columns** | Auto-injected by the strategy template: a NaN scan over every array declared in `__EXTRA_BUF_TOARRAY__` is prepended to `Compute()` and returns `false` if any cell is NaN. You do NOT need a manual `if (double.IsNaN(x)) return false;` in your compute body — write your factor logic assuming all values are finite |

#### Reference Implementations (full runnable examples)

<details>
<summary>Example: RSI Oversold-Bounce Factor (rsi_oversold_bounce)</summary>

```python
import pandas as pd
import numpy as np
from typing import Any, Dict

FACTOR_TYPE = "rsi_oversold_bounce"

FACTOR_DEFAULT_PARAMS = {
    "rsi_period": 14,
    "oversold":   30,
    "overbought": 70,
}

FACTOR_SECTIONS = {
    "__FACTOR_DESCRIPTION__": "RSI oversold bounce: long when RSI < oversold, short when RSI > overbought",
    "__FACTOR_FORMULA__":     "RSI < oversold → +(oversold-RSI)/oversold; RSI > overbought → -(RSI-overbought)/(100-overbought)",
    "__FACTOR_TYPE__":        "rsi_oversold_bounce",
    "__FACTOR_PARAM_FIELDS__": (
        "        private int _rsiPeriod;\n"
        "        private double _oversold;\n"
        "        private double _overbought;\n"
        "        private double _prevGainEma;\n"
        "        private double _prevLossEma;\n"
        "        private bool _rsiInitialized;\n"
    ),
    "__FACTOR_INIT__": (
        '            _rsiPeriod = GetIntParameter("rsi-period", 14);\n'
        '            _oversold = GetDoubleParameter("oversold", 30.0);\n'
        '            _overbought = GetDoubleParameter("overbought", 70.0);\n'
        '            _prevGainEma = 0.0;\n'
        '            _prevLossEma = 0.0;\n'
        '            _rsiInitialized = false;\n'
    ),
    "__FACTOR_LOG__": (
        '            Log($"[INIT] rsi_period={_rsiPeriod} oversold={_oversold} overbought={_overbought}");\n'
    ),
    "__PRICE_WINDOW_EXPR__": "_rsiPeriod + 1",
    "__EXTRA_BUF_FIELDS__":   "",
    "__EXTRA_BUF_ENQUEUE__":  "",
    "__EXTRA_BUF_DEQUEUE__":  "",
    "__EXTRA_BUF_TOARRAY__":  "",
    "__FACTOR_COMPUTE_BODY__": """
            var n = prices.Length;
            if (n < _rsiPeriod + 1) return false;

            if (!_rsiInitialized)
            {
                double sumGain = 0.0, sumLoss = 0.0;
                for (int i = 1; i < n; i++)
                {
                    var change = prices[i] - prices[i - 1];
                    if (change > 0) sumGain += change;
                    else sumLoss += Math.Abs(change);
                }
                _prevGainEma = sumGain / _rsiPeriod;
                _prevLossEma = sumLoss / _rsiPeriod;
                _rsiInitialized = true;
            }
            else
            {
                var change = prices[n - 1] - prices[n - 2];
                var gain = change > 0 ? change : 0.0;
                var loss = change < 0 ? Math.Abs(change) : 0.0;
                _prevGainEma = (_prevGainEma * (_rsiPeriod - 1) + gain) / _rsiPeriod;
                _prevLossEma = (_prevLossEma * (_rsiPeriod - 1) + loss) / _rsiPeriod;
            }

            double rsi;
            if (_prevLossEma < 1e-12)
                rsi = 100.0;
            else
            {
                var rs = _prevGainEma / _prevLossEma;
                rsi = 100.0 - 100.0 / (1.0 + rs);
            }

            if (rsi < _oversold)
                rawSignal = (_oversold - rsi) / _oversold;
            else if (rsi > _overbought)
                rawSignal = -(rsi - _overbought) / (100.0 - _overbought);
            else
                rawSignal = 0.0;

            return true;
""",
}


def _compute_rsi_wilder(close: pd.DataFrame, period: int) -> pd.DataFrame:
    delta = close.diff()
    gain  = delta.clip(lower=0.0)
    loss  = (-delta).clip(lower=0.0)
    avg_gain = gain.ewm(com=period - 1, min_periods=period, adjust=False).mean()
    avg_loss = loss.ewm(com=period - 1, min_periods=period, adjust=False).mean()
    rs  = avg_gain / avg_loss.replace(0, np.nan)
    rsi = 100.0 - 100.0 / (1.0 + rs)
    rsi.iloc[:period] = np.nan
    return rsi


def build_signal(close: pd.DataFrame, params: Dict[str, Any], **_) -> pd.DataFrame:
    rsi_period = int(params.get("rsi_period", 14))
    oversold   = float(params.get("oversold",   30.0))
    overbought = float(params.get("overbought", 70.0))

    rsi    = _compute_rsi_wilder(close, rsi_period)
    signal = pd.DataFrame(0.0, index=close.index, columns=close.columns)
    signal[rsi < oversold]   = (oversold   - rsi[rsi < oversold])   / oversold
    signal[rsi > overbought] = -(rsi[rsi > overbought] - overbought) / (100.0 - overbought)
    signal[rsi.isna()]       = np.nan
    return signal.reindex_like(close)
```

</details>

Once the plugin is written, save it to a temporary path for submission, then archive it under the job_id after submitting.

<details>
<summary>Example: Aggressive Money-Flow Factor (taker_buy_ratio_momentum) — uses taker extra columns</summary>

```python
import pandas as pd
import numpy as np
from typing import Any, Dict

FACTOR_TYPE = "taker_buy_ratio_momentum"

FACTOR_DEFAULT_PARAMS = {
    "window": 20,
}

FACTOR_SECTIONS = {
    "__FACTOR_DESCRIPTION__": "Aggressive money flow: rolling mean of taker active-buy share, deviation from neutral 0.5",
    "__FACTOR_FORMULA__":     "buy_ratio = taker_buy_vol / (buy+sell); signal = rolling_mean(buy_ratio, w) - 0.5",
    "__FACTOR_TYPE__":        "taker_buy_ratio_momentum",
    "__FACTOR_PARAM_FIELDS__": (
        "        private int _window;\n"
    ),
    "__FACTOR_INIT__": (
        '            _window = GetIntParameter("window", 20);\n'
    ),
    "__FACTOR_LOG__": (
        '            Log($"[INIT] window={_window}");\n'
    ),
    "__PRICE_WINDOW_EXPR__": "_window",
    # ── Extra columns: taker_buy_volume / taker_sell_volume ─────────────
    "__EXTRA_BUF_FIELDS__": (
        "        private readonly Queue<double> _takerBuyBuf  = new Queue<double>();\n"
        "        private readonly Queue<double> _takerSellBuf = new Queue<double>();\n"
    ),
    "__EXTRA_BUF_ENQUEUE__": (
        "            _takerBuyBuf.Enqueue(bar.TakerBuyVolume);\n"
        "            _takerSellBuf.Enqueue(bar.TakerSellVolume);\n"
    ),
    "__EXTRA_BUF_DEQUEUE__": (
        "            if (_takerBuyBuf.Count  > requiredBars) _takerBuyBuf.Dequeue();\n"
        "            if (_takerSellBuf.Count > requiredBars) _takerSellBuf.Dequeue();\n"
    ),
    "__EXTRA_BUF_TOARRAY__": (
        "            var takerBuys  = _takerBuyBuf.ToArray();\n"
        "            var takerSells = _takerSellBuf.ToArray();\n"
    ),
    "__FACTOR_COMPUTE_BODY__": """
            var n = prices.Length;
            if (n < _window) return false;

            double sumRatio = 0.0;
            for (int i = 0; i < n; i++)
            {
                var total = takerBuys[i] + takerSells[i];
                var ratio = total > 1e-12 ? takerBuys[i] / total : 0.5;
                sumRatio += ratio;
            }
            rawSignal = sumRatio / n - 0.5;
            return true;
""",
}


def build_signal(
    close:            pd.DataFrame,
    params:           Dict[str, Any],
    taker_buy_volume: pd.DataFrame,
    taker_sell_volume: pd.DataFrame,
    **_kwargs,
) -> pd.DataFrame:
    window = int(params.get("window", 20))
    total = taker_buy_volume + taker_sell_volume
    buy_ratio = taker_buy_volume / total.replace(0, float("nan"))
    signal = buy_ratio.rolling(window).mean() - 0.5
    return signal.reindex_like(close)
```

</details>

<details>
<summary>Example: Funding Rate Reversion Factor (funding_rate_reversion) — uses futures column</summary>

```python
import pandas as pd
import numpy as np
from typing import Any, Dict

FACTOR_TYPE = "funding_rate_reversion"

FACTOR_DEFAULT_PARAMS = {
    "window": 7,
}

FACTOR_SECTIONS = {
    "__FACTOR_DESCRIPTION__": "Rolling-mean funding rate reversion: high funding → crowded long → short next; low funding → crowded short → long next",
    "__FACTOR_FORMULA__":     "signal = -rolling_mean(funding_rate_close, w)",
    "__FACTOR_TYPE__":        "funding_rate_reversion",
    "__FACTOR_PARAM_FIELDS__": (
        "        private int _window;\n"
    ),
    "__FACTOR_INIT__": (
        '            _window = GetIntParameter("window", 7);\n'
    ),
    "__FACTOR_LOG__": (
        '            Log($"[INIT] window={_window}");\n'
    ),
    "__PRICE_WINDOW_EXPR__": "_window",
    # ── Extra column: funding_rate_close (double, no cast needed) ──
    "__EXTRA_BUF_FIELDS__": (
        "        private readonly Queue<double> _frBuf = new Queue<double>();\n"
    ),
    "__EXTRA_BUF_ENQUEUE__": (
        "            _frBuf.Enqueue(bar.FundingRateClose);\n"
    ),
    "__EXTRA_BUF_DEQUEUE__": (
        "            if (_frBuf.Count > requiredBars) _frBuf.Dequeue();\n"
    ),
    "__EXTRA_BUF_TOARRAY__": (
        "            var frs = _frBuf.ToArray();\n"
    ),
    # The strategy template auto-injects a NaN scan over frs[] before this body runs,
    # so we can compute the mean assuming all values are finite.
    "__FACTOR_COMPUTE_BODY__": """
            var n = prices.Length;
            if (n < _window) return false;

            double sum = 0.0;
            for (int i = 0; i < frs.Length; i++) sum += frs[i];
            var mean = sum / frs.Length;
            rawSignal = -mean;
            return true;
""",
}


def build_signal(
    close:               pd.DataFrame,
    params:              Dict[str, Any],
    funding_rate_close:  pd.DataFrame,
    **_kwargs,
) -> pd.DataFrame:
    window = int(params.get("window", 7))
    # rolling.mean() auto-skips NaN cells (min_periods=window ensures we never
    # output a value computed on a partially-NaN window).
    signal = -funding_rate_close.rolling(window, min_periods=window).mean()
    return signal.reindex_like(close)
```

</details>

---

### Phase 2: Submit Jobs (Dual Mode)

For every factor, **submit two jobs at once** — sigmoid_continuous and quantile_discrete:

```bash
# Use a process-unique temp path so concurrent Agents don't overwrite each other
PLUGIN_TMP="/tmp/plugin_${FACTOR_TYPE}_$$.py"

cat > ${PLUGIN_TMP} << 'PLUGIN_EOF'
<plugin content>
PLUGIN_EOF

# Job 1: sigmoid_continuous
curl -s -X POST ${BASE_URL}/jobs/submit \
  -F "factor_kind=custom" \
  -F "factor_type=<factor_type>" \
  -F "factor_name=<factor_name>" \
  -F "params=<JSON string, e.g. {\"rsi_period\":14}>" \
  -F "fwd_period=16" \
  -F "plugin=@${PLUGIN_TMP}"

# Job 2: quantile_discrete
curl -s -X POST ${BASE_URL}/jobs/submit \
  -F "factor_kind=custom" \
  -F "factor_type=<factor_type>" \
  -F "factor_name=<factor_name>" \
  -F "params=<JSON string>" \
  -F "fwd_period=16" \
  -F "position_mode=quantile_discrete" \
  -F "entry_q=20" \
  -F "plugin=@${PLUGIN_TMP}"
```

The two submissions return one `job_id` each; **save them as two shell variables**:

```bash
JOB_ID_SIG="job_20260312_153001_xxxxxx"   # sigmoid_continuous
JOB_ID_QD="job_20260312_153002_yyyyyy"    # quantile_discrete

mkdir -p ./quant_agent/jobs/${JOB_ID_SIG}
mkdir -p ./quant_agent/jobs/${JOB_ID_QD}
cp ${PLUGIN_TMP} ./quant_agent/jobs/${JOB_ID_SIG}/plugin.py
cp ${PLUGIN_TMP} ./quant_agent/jobs/${JOB_ID_QD}/plugin.py

# Archive: write only one record for SIG/QD (use JOB_ID_SIG as the job_id)
REGISTRY=./quant_agent/factor_registry.jsonl
[ -f "$REGISTRY" ] || : > "$REGISTRY"
FORMULA_TEXT=$(awk -F'"' '/__FACTOR_FORMULA__/ {print $4; exit}' "./quant_agent/jobs/${JOB_ID_SIG}/plugin.py")
FORMULA_ESC=$(printf '%s' "${FORMULA_TEXT}" | sed 's/\\/\\\\/g; s/"/\\"/g')
printf '{"factor_type":"%s","formula":"%s","job_id":"%s"}\n' \
  "${FACTOR_TYPE}" "${FORMULA_ESC}" "${JOB_ID_SIG}" >> "$REGISTRY"

rm -f ${PLUGIN_TMP}
```

> **builtin factors** (`momentum` / `trend` / `mean_revert`) do not need a plugin upload —
> use `factor_kind=builtin` and omit `-F "plugin=..."` (no plugin.py archiving needed).

---

### Phase 3: Poll and Wait (two jobs in parallel)

Every **15 seconds**, query both jobs' status, with a max wait of **30 minutes**:

```bash
curl -s ${BASE_URL}/jobs/${JOB_ID_SIG}/status
curl -s ${BASE_URL}/jobs/${JOB_ID_QD}/status
```

The two jobs are **handled independently**: one finishing does not affect the other's polling, and one failing does not affect the other.

#### Agent Behavior Cheat Sheet (judge per job)

| status | Agent action |
|--------|-----------|
| `queued` / `running` (`current_step` < `"5"`) | Keep waiting. Every 2–3 polls, report progress to user |
| `running` (`current_step` >= `"5"`) or `done` | **This job's Step 4C is done**, mark as ready to download |
| `failed` (`failed_step="4l"`) | Python plugin has future-data leakage; rewrite `build_signal` and submit **a new job** (retest forbidden). Both jobs share the same plugin and must be resubmitted together |
| `failed` (`failed_step="4c"`) | Go to **phase 3b** to fix this job's C# |
| `failed` (other steps) | Tell the user this job hit a server-internal error and cannot be fixed |
| `retesting` | Keep waiting |
| `retest_failed` | Inspect retest logs and fix strategy.cs again |

> **Key rule**: only enter phase 4 once **both jobs** have `current_step >= 5`. If one finishes first, keep waiting on the other; if one fully fails (non-C#-compile), still proceed to phase 4 with the surviving result and tell the user which mode failed.

#### strategy_cs_ready Flag

If a job's status returns `"strategy_cs_ready": true`, immediately download and archive that job's strategy.cs (one-time):

```bash
# Use the fetch_artifact helper from phase 4 (handles EFS→S3 fallback automatically)
mkdir -p ./quant_agent/jobs/${JOB_ID_SIG}
fetch_artifact ${JOB_ID_SIG} strategy.cs ./quant_agent/jobs/${JOB_ID_SIG}/strategy.cs

mkdir -p ./quant_agent/jobs/${JOB_ID_QD}
fetch_artifact ${JOB_ID_QD} strategy.cs ./quant_agent/jobs/${JOB_ID_QD}/strategy.cs
```

---

### Phase 3b: Fix C# Compile Errors and Retest

When a job has `status=failed` and `failed_step="4c"`, fix **that job**.
The two jobs use different C# templates (sigmoid vs quantile) — fix them separately.
Below, `${JOB_ID}` refers to the failing job_id.

**1. Inspect error logs**

```bash
curl -s "${BASE_URL}/jobs/${JOB_ID}/logs?tail=80"
```

**2. Download and fix strategy.cs**

```bash
curl -s ${BASE_URL}/jobs/${JOB_ID}/files/strategy.cs \
  -o ./quant_agent/jobs/${JOB_ID}/strategy.cs
```

Fix according to the log error info. Common error reference:

| Error | Cause | Fix |
|---------|------|---------|
| `CS0019: Operator '/' cannot be applied to 'double' and 'decimal'` | C# type mismatch | Add `(double)` cast before division |
| `CS0103: The name 'xxx' does not exist` | Variable name typo or wrong scope | Check declaration in `__FACTOR_PARAM_FIELDS__` |
| `CS0128: A local variable named 'xxx' is already defined` | Variable name conflicts with template framework | Rename in `__FACTOR_COMPUTE_BODY__` (**do not touch framework code**) |
| `CS1002: ; expected` | C# syntax error | Check end of every line in `__FACTOR_COMPUTE_BODY__` |
| `rawSignal` always 0 | Forgot to assign `rawSignal` before `return true` | Assign `rawSignal` on every code path |
| Runtime NullReference during backtest | Accessed an uninitialized field | Check whether `__FACTOR_INIT__` missed initializing some field |

**Fix principle**: only modify code inside `#region FactorComputeBody`; do not touch framework code.

**3. Submit retest**

```bash
curl -s -X POST ${BASE_URL}/jobs/${JOB_ID}/retest \
  -F "strategy_cs=@./quant_agent/jobs/${JOB_ID}/strategy.cs"
```

After receiving `{ "status": "retesting" }`, return to **phase 3** and continue polling. Once retest is submitted, the server resumes from the failure point and runs all subsequent steps automatically.

> If retest fails 3 times in a row, consider rewriting plugin.py and POST `/jobs/submit` again to start a fresh job.

---

### Phase 4: Fetch Results, Compare, and Move to Next Factor

> **⚠️ Key rule**: when both jobs reach `current_step >= 5`, Step 4C is done — **download files directly**.
> **Do not call the `/result` endpoint** — that endpoint requires the entire pipeline (Step 16D) to finish before returning data,
> while the user only needs to see the Step 4C default-parameter backtest, not later steps.

#### 4a. Download Artifacts (run for both jobs)

> **Note**: the server runs a daily EFS cleanup (max_age=1 day). For older jobs the
> EFS-side artifact may already be deleted, but the S3 archive remains. Use the
> `fetch_artifact` helper below: it tries the default `as=file` path first, and on
> 404 transparently falls back to `?as=url` to download from S3 via presigned URL.

```bash
# Helper: try EFS first, fall back to S3 presigned URL on 404
fetch_artifact() {
  local job_id="$1" name="$2" out="$3"
  local code
  code=$(curl -s -o "${out}" -w "%{http_code}" \
           "${BASE_URL}/jobs/${job_id}/files/${name}")
  if [ "${code}" = "200" ]; then return 0; fi
  if [ "${code}" = "404" ]; then
    # Fall back to S3 presigned URL (EFS may have been cleaned)
    local url
    url=$(curl -s "${BASE_URL}/jobs/${job_id}/files/${name}?as=url" \
           | python3 -c "import sys,json; print(json.load(sys.stdin).get('url',''))")
    if [ -n "${url}" ]; then
      code=$(curl -sL -o "${out}" -w "%{http_code}" "${url}")
      [ "${code}" = "200" ] && return 0
    fi
  fi
  rm -f "${out}"   # don't leave empty/partial files behind
  echo "skip ${name} (http=${code})"
  return 1
}

# Run for JOB_ID_SIG and JOB_ID_QD separately (JOB_ID below stands in for either)
for JOB_ID in ${JOB_ID_SIG} ${JOB_ID_QD}; do
  JOB_DIR=./quant_agent/jobs/${JOB_ID}
  mkdir -p ${JOB_DIR}/step4c

  fetch_artifact ${JOB_ID} default_factor_card.json      ${JOB_DIR}/factor_card_default.json
  fetch_artifact ${JOB_ID} default_factor_card.txt       ${JOB_DIR}/factor_card_default.txt
  fetch_artifact ${JOB_ID} default_equity_curves.png     ${JOB_DIR}/step4c/equity_curves.png
  fetch_artifact ${JOB_ID} default_ts_profile_4panel.png ${JOB_DIR}/step4c/ts_profile_4panel.png
  fetch_artifact ${JOB_ID} default_trade_log.csv         ${JOB_DIR}/step4c/trade_log.csv
  fetch_artifact ${JOB_ID} default_group_return_plot.png ${JOB_DIR}/step4c/group_return_plot.png
  fetch_artifact ${JOB_ID} default_cs_profile_4panel.png ${JOB_DIR}/step4c/cs_profile_4panel.png
  fetch_artifact ${JOB_ID} default_cs_nav_curves.png     ${JOB_DIR}/step4c/cs_nav_curves.png
done
```

> If a file is not yet generated (old job, or Step 12 not yet done), both paths return 404; ignore.
> If a file is missing only on EFS (cleaned up), the fallback transparently fetches it from S3.

#### 4b. Read Both factor_card_default.json Files and Present a Side-by-Side Comparison

Read SIG's and QD's `factor_card_default.json` separately, extract the key metrics, build a **comparison table**.
**You must report `ts_success`, `cs_success`, and `status` (Overall = TS OR CS) together** — never report only a single pass/fail.

**Sigmoid Continuous (SIG job) — fields to focus on:**

| JSON field | Purpose |
|-----------|------|
| `ts_success` | Whether TS succeeded (`true/false`) |
| `cs_success` | Whether CS succeeded (`true/false`) |
| `status` | Overall (`pass/fail`, rule = TS OR CS) |
| `ts_fail_reasons` | TS failure reason list (if any) |
| `cs_fail_reasons` | CS failure reason list (if any) |
| `median_sharpe` | Default-param median Sharpe |
| `icir` | IC information ratio |
| `median_annual_return` | Median annualized return |
| `median_max_drawdown` | Median max drawdown |
| `win_rate` | Win rate |
| `rank_icir` | RankICIR (cross-section predictive power) |
| `cs_branch.profile.monotonicity_score` | Group monotonicity score (if any) |

**Quantile Discrete (QD job) — fields to focus on:**

| JSON field | Meaning |
|-----------|------|
| `ts_success` | Whether TS succeeded (`true/false`) |
| `cs_success` | Whether CS succeeded (`true/false`) |
| `status` | Overall (`pass/fail`, rule = TS OR CS) |
| `ts_fail_reasons` | TS failure reason list (if any) |
| `cs_fail_reasons` | CS failure reason list (if any) |
| `median_sharpe` | C# backtest Sharpe (signal-quality reference) |
| `ts_branch.discrete_turnover` | Discrete state-switching frequency (per bar) |
| `ts_branch.median_hold_bars` | Median hold duration (in bars) |
| `ts_branch.metrics_pool.sharpe_pool` | Portfolio-level Sharpe (aggregated across symbols) |
| `ts_branch.metrics_pool.max_dd_pool` | Portfolio-level max drawdown |
| `rank_icir` | RankICIR (cross-section predictive power) |
| `direction_stability` | Rolling IC same-sign ratio (0–1) |

> **Important**: the QD job's primary results come from the Python-side discrete-position simulation in Step 8/10;
> the C# Lean cloud backtest (Step 4C)'s `median_sharpe` is computed under the Sigmoid convention and serves as a **signal-quality reference only**.

Also present **both jobs' charts**:
- `equity_curves.png`: TS time-series strategy equity curve (one for SIG, one for QD)
- `ts_profile_4panel.png`: TS 4-in-1 time-series profile — IC mean / win rate / Sharpe distribution / drawdown distribution (one for SIG, one for QD; **must be shown**)
- `group_return_plot.png`: CS cross-section grouped cumulative return (both jobs share the same CS data; show one is enough)
- `cs_profile_4panel.png`: CS 4-in-1 cross-section evaluation chart (both jobs share the same CS data; show one is enough)
- `cs_nav_curves.png`: CS NAV curves — long-only / short-only / long-short (both jobs share; show one is enough)

#### 4c. Comparison Summary and Move to Next Round

Use a **comparison table + a short summary** to summarize the core performance of the two modes:

```
| Metric             | Sigmoid Continuous | Quantile Discrete |
|--------------------|--------------------|-------------------|
| TS Success         | true / false       | true / false      |
| CS Success         | true / false       | true / false      |
| Overall (TS OR CS) | pass / fail        | pass / fail       |
| Median Sharpe      | x.xx               | x.xx (signal ref.)|
| QD Sharpe Pool     | -                  | x.xx              |
| Rank ICIR          | x.xx               | x.xx              |
| Win Rate           | xx%                | -                 |
| Monotonicity       | x.xx               | x.xx              |
| Hold Bars (QD)     | -                  | xx                |
| Turnover (QD)      | -                  | x.xxxx            |
| Dir Stability      | x.xx               | x.xx              |
```

Summary must include:
- Which mode performed better, plus relative pros and cons
- Explicitly explain whether TS succeeded, whether CS succeeded, and why Overall is the current value
- If a mode's Overall fails, identify the failure dimension (TS / CS) and give an improvement direction
- Highlight whether `rank_icir` and `direction_stability` support admission to the factor library

**After presenting results:**

- **WORK_MODE = A or B**: discuss the next factor with the user directly — this factor's full work is finished.

---

## Full Workflow Diagram

```
User: "Research a Bollinger band-width breakout factor"
        │
        ▼
[Phase -1] Choose work mode
           A: Auto-Discovery Mode
           B: Free Mode
        │
        ▼
[Phase 0] Confirm factor_type / factor_name / params
        │
        ▼
[Phase 1] Write plugin.py (C# fragments + Python build_signal)
        │
        ▼
[Phase 2] POST /jobs/submit × 2 (sigmoid + quantile) → JOB_ID_SIG + JOB_ID_QD
        │
        ▼
[Phase 3] Poll both jobs in parallel; wait for Step 4C done (current_step >= 5 or done)
        │
        ├─ strategy_cs_ready=true → download that job's strategy.cs
        ├─ failed (4c) → [Phase 3b] fix that job's C# → retest → back to polling
        │
        └─ Both jobs at current_step >= 5 or done (or one fully failed)
             │
             ▼
[Phase 4] Download both jobs' default_ files (incl. new ts_profile_4panel.png)
        → Compare and present factor cards + TS 4-in-1 profile
        → Discuss next factor
(Steps 5–16 run async on the server; the user-facing flow ends here)
```

---

## Other Endpoints

```bash
# View retest logs
curl -s "${BASE_URL}/jobs/${JOB_ID}/retest_logs?tail=100"

# Health check
curl -s ${BASE_URL}/health
```

---

## Cadence for Reporting Progress to the User

- Immediately after submission, report both job IDs (label them as "continuous mode" / "quantile mode" — do not explain the technical meaning)
- Every 2–3 poll intervals, tell the user the current state of both jobs (no need to update on every poll)
- When `current_step="4c"`, say "running backtest, takes ~3–5 minutes"
- On C# compile failure (Step 4C only), say "fixing the code and retrying" — do not throw an error
- **Once both jobs reach `current_step >= 5`, stop polling immediately and fetch results**
- After results are ready, use a comparison table to show both modes; emphasize "did the time-series test pass", "did the cross-section test pass", "overall conclusion", Sharpe, ICIR
- **Show only the default-parameter results**; after presenting, move directly to the next factor

### ⛔ Forbidden Wording (must not appear when speaking to the user)

The following are internal terms — they only live in the Agent's processing logic and **must not be spoken to the user**:

| Forbidden term | Replace with |
|-----------|-----------|
| `Step 4C` / `current_step` / `Step 4L` | "backtest stage" / "detection stage" |
| `strategy_cs_ready` | Don't mention to the user |
| `sigmoid_continuous` | "continuous mode", or omit and don't explain |
| `quantile_discrete` | "quantile mode", or omit and don't explain |
| `SIG job` / `QD job` | "continuous-mode task" / "quantile-mode task" |
| `C# compile failed` | "the strategy code hit a small issue" |
| `retest` / `polling` | "retest" / "waiting for results" |
| `plugin.py` / `build_signal` / `FACTOR_SECTIONS` | Don't mention to the user |
| `ts_success` / `cs_success` / `rank_icir` and other field names | Use plain English meaning, e.g. "time-series test passed" / "cross-section predictive power" |
