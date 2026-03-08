# myfinance2 — Repository Overview

> Quantitative finance platform for automated daily analysis of **Borsa Italiana** (Italian stock market) equities, running end-to-end via GitHub Actions.

---

## Table of Contents

1. [Project Summary](#1-project-summary)
2. [Repository Layout](#2-repository-layout)
3. [End-to-End Data Pipeline](#3-end-to-end-data-pipeline)
4. [Scripts Reference](#4-scripts-reference)
5. [Signal Generation](#5-signal-generation)
6. [Stop-Loss and Risk](#6-stop-loss-and-risk)
7. [Trading Report and Dashboard](#7-trading-report-and-dashboard)
8. [CI/CD Workflows](#8-cicd-workflows)
9. [Configuration Reference](#9-configuration-reference)
10. [Dependencies](#10-dependencies)

---

## 1. Project Summary

| Property | Value |
|---|---|
| **Market** | Borsa Italiana (Milan, `.MI` tickers) |
| **Benchmark** | `FTSEMIB.MI` (FTSE MIB Index) |
| **Data source** | Yahoo Finance via `yfinance` |
| **Data format** | Parquet (long/tidy format) |
| **Signal types** | Breakout, Turtle Trader, Triple MA Crossover, Floor/Ceiling |
| **Automation** | GitHub Actions — runs Mon–Fri after market close (21:00 UTC) |
| **Language** | Python 3.11 |

**Core idea:** every day after Borsa Italiana closes, the pipeline automatically downloads new price data, computes trading signals on relative prices (stock vs FTSE MIB), calculates ATR-based stop-losses, and produces an actionable trading dashboard.

All signals operate on **relative prices** — the stock price divided by the rebased benchmark — so they measure alpha (stock-specific outperformance) rather than raw price movement.

---

## 2. Repository Layout

```
myfinance2/
│
├── analyze_stock.py              # Main analysis entry point
├── get_daily_ohlc_data.py        # Download today's OHLC bar
├── get_historical_ohlc_data.py   # Bulk backfill (2016–present)
├── trading_report.py             # Generate dashboard + text report
├── generate_symbol_notebooks.py  # Per-symbol Jupyter notebooks
├── pipeline.py                   # Stateless pipeline stage functions
├── config.json                   # Central parameter store
│
├── algoshort-0.1.1-py3-none-any.whl  # Private package (install manually)
│
├── data/
│   ├── ticker/it/ticker.xlsx         # Input: list of tickers (ticker column)
│   ├── ohlc/
│   │   ├── today/it/ohlc_data.parquet       # Latest daily bar
│   │   └── historical/it/ohlc_data.parquet  # Full history (appended daily)
│   └── results/it/
│       ├── analysis_results.parquet   # Full analysis (~500 MB, NOT in git)
│       ├── cumul_snapshot.xlsx        # Terminal cumulative returns
│       ├── trading_dashboard.xlsx     # Actionable dashboard (in git)
│       └── trading_report.txt         # Text report (in git)
│
├── notebooks/
│   ├── symbol_analysis_template.ipynb  # Template for per-symbol analysis
│   └── it/<SYMBOL>.ipynb               # Generated notebooks (artifact only)
│
└── .github/workflows/
    ├── download_daily_ohlc.yml         # Workflow 1: daily download
    ├── analyze_and_report.yml          # Workflow 2: analysis + report
    └── generate_symbol_notebooks.yml  # Workflow 3: notebook generation
```

---

## 3. End-to-End Data Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│  INPUT                                                              │
│  data/ticker/it/ticker.xlsx  →  [AVIO.MI, A2A.MI, ...]            │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 1 — get_daily_ohlc_data.py                                    │
│  Download today's OHLC bar from Yahoo Finance                       │
│  → data/ohlc/today/it/ohlc_data.parquet                            │
└──────────────────────┬──────────────────────────────────────────────┘
                       │  CI appends to historical
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 2 — CI append step                                            │
│  Concat today's bar with cumulative history                         │
│  → data/ohlc/historical/it/ohlc_data.parquet  (2016–present)       │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 3 — analyze_stock.py + pipeline.py                            │
│                                                                     │
│  Load config.json                                                   │
│  Load historical OHLC                                               │
│  For each symbol:                                                   │
│    1. Normalize vs FTSEMIB.MI  → ropen, rhigh, rlow, rclose        │
│    2. Generate signals         → rbo_*, rtt_*, rsma_*, rema_*, rrg │
│    3. Calculate returns        → <signal>_cumul, <signal>_chg1D    │
│    4. Calculate stop-losses    → <signal>_stop_loss                 │
│  → data/results/it/analysis_results.parquet  (~500 MB)             │
│  → data/results/it/cumul_snapshot.xlsx                              │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 4 — trading_report.py                                         │
│  For each signal × ticker:                                          │
│    - Detect current action (ENTER / EXIT / HOLD / FLIP)            │
│    - Compute risk_pct = |price - stop_loss| / price                 │
│  → data/results/it/trading_dashboard.xlsx  (in git)                │
│  → data/results/it/trading_report.txt      (in git)                │
└──────────────────────┬──────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 5 — generate_symbol_notebooks.py  (optional)                 │
│  Run symbol_analysis_template.ipynb for each ticker via papermill  │
│  → notebooks/it/<SYMBOL>.ipynb  (uploaded as GitHub artifact)      │
└─────────────────────────────────────────────────────────────────────┘
```

### Data schema (all Parquet files)

Long/tidy format — one row per `(symbol, date)`.

| Column | Type | Description |
|---|---|---|
| `symbol` | string | Yahoo Finance ticker, e.g. `AVIO.MI` |
| `date` | datetime | Bar date |
| `open` | float | Opening price |
| `high` | float | Daily high |
| `low` | float | Daily low |
| `close` | float | Closing price |
| `volume` | int | Trading volume |
| `fx` | float | FX adjustment (always `1.0` for `.MI` stocks) |

After analysis, additional columns are added per signal (see §5).

---

## 4. Scripts Reference

### `get_daily_ohlc_data.py`

Downloads the current day's OHLC bar for all tickers in `ticker.xlsx`.

- Uses `algoshort.YFinanceDataHandler` with chunked multi-threaded downloads (`chunk_size=20`, `threads=True`)
- `use_cache=False` — always fetches fresh data
- Output: `data/ohlc/today/it/ohlc_data.parquet`

### `get_historical_ohlc_data.py`

One-time (or periodic reset) bulk download from `2016-01-01` to present.

- Same handler as above, but with `start='2016-01-01'`
- After first pass: detects symbols with zero rows and retries them individually
- Output: `data/ohlc/historical/it/ohlc_data.parquet`

### `analyze_stock.py` + `pipeline.py`

Main analysis pipeline. `analyze_stock.py` is the entry point; `pipeline.py` contains all the pure transformation functions.

**Execution order:**

```python
cfg = load_config("config.json")
ohlc_data, symbols = load_data(historical_parquet, benchmark)
dfs = build_symbol_dfs(ohlc_data, symbols)        # split by symbol
dfs = compute_relative_prices(dfs, benchmark_df)  # normalize vs FTSEMIB.MI
dfs, signals = generate_all_signals(dfs, ...)     # Breakout, Turtle, MA, FC
dfs = calculate_returns(dfs, signals)             # cumulative returns
dfs = calculate_stop_losses(dfs, signals, atr_window=14, atr_multiplier=2.0)
save_results(dfs, output_path)                    # write parquet + Excel
```

**Design principle:** each function in `pipeline.py` is a stateless `DataFrame → DataFrame` transformation — independently testable and reorderable.

### `trading_report.py`

Reads `analysis_results.parquet` and produces the trading dashboard.

- Detects signal columns automatically: any column `X` where `X_stop_loss` also exists
- Calls `algoshort.trading_summary.get_multi_symbol_summary()` for each signal
- Outputs **two sections** to stdout (captured to `trading_report.txt`):
  1. **Actionable Trades** — Enter/Exit events in the last 4 trading days
  2. **Latest Status** — per-signal detail for active signals on the latest bar
- Also saves `trading_dashboard.xlsx` with full per-(signal × ticker) flat table

### `generate_symbol_notebooks.py`

Generates one Jupyter notebook per symbol using `papermill`.

- Template: `notebooks/symbol_analysis_template.ipynb`
- Injects `SYMBOL` parameter into each execution
- Output: `notebooks/it/<SYMBOL>.ipynb`
- Graceful failure: errors per symbol are logged and skipped; batch continues

---

## 5. Signal Generation

### 5.1 Relative Price Normalization

All signals run on **relative prices**, not raw prices. This removes the systematic benchmark component and isolates alpha.

```
bmfx    = benchmark_close / benchmark_close[0]   # rebase to 1.0 at first date

ropen  = open  / bmfx
rhigh  = high  / bmfx
rlow   = low   / bmfx
rclose = close / bmfx
```

**Example:** if a stock doubled while the benchmark stayed flat, `rclose` doubles. If both moved identically, `rclose` stays at 1.0.

### 5.2 Breakout Signal (`rbo_<window>`)

**Logic:**

```
rhi_N = rolling_max(rhigh, N)   # N-bar channel high
rlo_N = rolling_min(rlow,  N)   # N-bar channel low

IF rhigh == rhi_N  →  signal = +1   (new N-bar high → long)
IF rlow  == rlo_N  →  signal = -1   (new N-bar low  → short)
ELSE               →  forward-fill previous signal
```

Signal persists until the opposite channel extreme is breached.

**Active parameter:** `bo_window = 150`
**Output columns:** `rhi_150`, `rlo_150`, `rbo_150`

### 5.3 Turtle Trader Signal (`rtt_<fast><slow>`)

Dual-window breakout combining a direction signal (slow) with a trailing stop (fast).

```
slow_regime = breakout(rhigh, rlow, window=50)   # direction
fast_regime = breakout(rhigh, rlow, window=20)   # trailing stop

rtt_5020 = slow × fast  (element-wise, then re-mapped)

→ +1 if slow=+1 AND fast=+1  (long, trailing stop not hit)
→ -1 if slow=-1 AND fast=-1  (short, trailing stop not hit)
→  0 otherwise               (position closed by trailing stop)
```

**Why this matters:** the fast window acts as a trailing stop. When the 20-bar breakout reverses, the position closes even if the 50-bar direction signal hasn't flipped — this limits drawdown.

**Active parameters:** `fast_window = 20`, `slow_window = 50`
**Output columns:** `rbo_20`, `rbo_50`, `rtt_5020`
(Note: `rbo_20` and `rbo_50` also appear in the dashboard as standalone signals.)

### 5.4 Triple MA Crossover (`rsma_*`, `rema_*`)

Three moving averages (50, 100, 150 periods) must all align for a signal.

```
sm = sign(MA_50  − MA_100)   # short vs medium: +1 bullish, -1 bearish
ml = sign(MA_100 − MA_150)   # medium vs long:  +1 bullish, -1 bearish

final = sm × ml

→ +1 if sm=+1 AND ml=+1  (all MAs: 50 > 100 > 150, fully bullish)
→ -1 if sm=-1 AND ml=-1  (all MAs: 50 < 100 < 150, fully bearish)
→  0 if mixed            (conflicting crossovers → no position)
```

Run for both `sma` and `ema`, producing 6 signal columns:

| Column | Description |
|---|---|
| `rsma_50100` | Relative SMA: 50 vs 100 crossover only |
| `rsma_100150` | Relative SMA: 100 vs 150 crossover only |
| `rsma_50100150` | Relative SMA: all three aligned |
| `rema_50100` | Relative EMA: 50 vs 100 crossover only |
| `rema_100150` | Relative EMA: 100 vs 150 crossover only |
| `rema_50100150` | Relative EMA: all three aligned |

### 5.5 Floor/Ceiling Regime (`rrg`)

Identifies support/resistance regimes using swing-high/swing-low detection with volatility bands.
Computed by `algoshort.RegimeFC`.

| Parameter | Value | Meaning |
|---|---|---|
| `vlty_n` | 63 | Volatility lookback (one quarter) |
| `dist_pct` | 0.05 | Distance threshold from swing to current price (5%) |
| `retrace_pct` | 0.05 | Retracement % to confirm regime change |
| `threshold` | 1.5 | ATR multiple for level validation |

### 5.6 Signal Column Naming Convention

```
r        = relative (uses rclose/rhigh/rlow instead of close/high/low)
bo_N     = breakout, N-bar channel
tt_FasSl = turtle, fast=Fa, slow=Sl
sma_ABCD = SMA crossover, windows A/B/C/D
ema_ABCD = EMA crossover, windows A/B/C/D
rrg      = relative floor/ceiling regime

Examples:
  rbo_150     → relative breakout, 150-bar channel
  rtt_5020    → relative turtle, fast=20, slow=50
  rsma_50100150 → relative triple SMA, windows 50/100/150
```

### 5.7 Columns added per signal after full pipeline

```
<signal>               → ±1 / 0 (raw signal)
<signal>_chg1D         → daily price change × lagged signal
<signal>_chg1D_fx      → same (legacy duplicate)
<signal>_PL_cum        → cumulative P&L from chg1D
<signal>_returns       → pct_change × lagged signal
<signal>_log_returns   → log(1 + pct_change) × lagged signal
<signal>_cumul         → exp(cumsum(log_returns)) − 1
<signal>_stop_loss     → ATR-based stop-loss level
```

**Lagged signal:** returns use `signal.shift(1)` — today's signal is acted on at tomorrow's open, preventing look-ahead bias.

---

## 6. Stop-Loss and Risk

### ATR Stop-Loss

**True Range (TR):**
```
TR = max(
    high − low,
    |high − close_prev|,
    |low  − close_prev|
)
```

**ATR:**
```
ATR(14) = rolling_mean(TR, window=14)
```

**Stop-Loss levels:**
```
Long  position:  stop_loss = close − (ATR × 2.0)
Short position:  stop_loss = close + (ATR × 2.0)
Flat:            stop_loss = NaN
```

**NaN warm-up handling:** ATR requires 14 bars before producing a valid value. The first 14 rows of `<signal>_stop_loss` would be NaN. These are patched with `ffill().bfill()` so position sizing never receives NaN.

### Risk Percent (`risk_pct`)

Computed at report time by `algoshort.trading_summary`:

```
risk_pct = |price − stop_loss| / price × 100
```

- Only meaningful on **Enter** signals (sizing into a new position)
- `NaN` on Exit signals (closing a position, no forward risk)
- Represents: "if stop-loss is hit immediately, what % of capital is lost on this position"

**Verification example from 2026-03-06:**
```
CED.MI: price=1.860, stop=1.7957 → |1.860 − 1.7957| / 1.860 = 3.46% ✓
ENV.MI: price=4.140, stop=3.8571 → |4.140 − 3.8571| / 4.140 = 6.83% ✓
```

---

## 7. Trading Report and Dashboard

### `trading_report.txt` — two sections

**Section 1: Actionable Trades (last 4 trading days)**

Filtered to `Enter` and `Exit` actions only. Columns:

| Column | Description |
|---|---|
| `last_date` | Date the action was triggered |
| `signal` | Strategy name |
| `ticker` | Yahoo Finance symbol |
| `price` | Close price on action date |
| `action` | `ENTER LONG`, `ENTER SHORT`, `EXIT LONG`, `EXIT SHORT` |
| `stop_loss` | ATR stop-loss level at entry |
| `risk_pct` | % distance from price to stop-loss |

**Section 2: Latest Status for Active Signals**

Per-signal detail view using `algoshort.print_multi_symbol_summary()` for tickers with Enter/Exit on the latest bar only.

---

### `trading_dashboard.xlsx` — full flat table

**2,650 rows = 10 signals × 265 tickers**. One row per `(signal, ticker)` showing the latest bar state.

| Column | Type | Description |
|---|---|---|
| `signal` | string | Signal strategy name |
| `ticker` | string | Yahoo Finance symbol |
| `last_date` | date | Date of last available bar |
| `price` | float | Close price on `last_date` |
| `position` | string | `LONG`, `SHORT`, or `FLAT` |
| `action` | string | What happened on last bar (see below) |
| `signal_changed` | bool | `True` if signal flipped on `last_date` |
| `stop_loss` | float | ATR-based stop-loss level |
| `risk_pct` | float | `\|price − stop_loss\| / price × 100` |

**Action taxonomy:**

| Action | Meaning |
|---|---|
| `ENTER LONG` | New long position opened |
| `ENTER SHORT` | New short position opened |
| `EXIT LONG` | Long closed → now flat |
| `EXIT SHORT` | Short closed → now flat |
| `FLIP: SHORT → LONG` | Reversed directly from short to long |
| `FLIP: LONG → SHORT` | Reversed directly from long to short |
| `HOLD LONG` | Existing long, no change |
| `HOLD SHORT` | Existing short, no change |
| `STAY FLAT` | No signal, no position |

**report.txt vs dashboard.xlsx difference:**

The text report shows `EXIT SHORT` / `ENTER LONG` as two separate events (from `rtt_5020`), while the dashboard shows `FLIP: SHORT → LONG` for the same move under `rbo_20`/`rbo_50`. This is because the two signals have different internal state machines — the Turtle signal passes through 0 (flat) between positions, while Breakout transitions directly.

---

## 8. CI/CD Workflows

Three workflows run in sequence, each triggering the next on success.

```
Mon–Fri 21:00 UTC
        │
        ▼
┌───────────────────────────────────┐
│  download_daily_ohlc.yml          │
│  Runs: get_daily_ohlc_data.py     │
│  Commits to git:                  │
│    data/ohlc/today/...parquet     │
│    data/ohlc/historical/...parquet│
└──────────────┬────────────────────┘
               │ on success
               ▼
┌───────────────────────────────────┐
│  analyze_and_report.yml           │
│  Timeout: 90 min                  │
│  Runs: analyze_stock.py           │
│        trading_report.py          │
│  Commits to git:                  │
│    trading_dashboard.xlsx         │
│    trading_report.txt             │
│  Uploads artifact (30 days):      │
│    analysis_results.parquet       │
└──────────────┬────────────────────┘
               │ on success
               ▼
┌───────────────────────────────────┐
│  generate_symbol_notebooks.yml    │
│  Timeout: 180 min                 │
│  Downloads artifact from step 2   │
│  Runs: generate_symbol_notebooks  │
│  Uploads artifact (30 days):      │
│    notebooks/it/*.ipynb           │
└───────────────────────────────────┘
```

**Why `analysis_results.parquet` is an artifact and not committed:**
At ~500 MB it would bloat the git history unacceptably. GitHub Actions artifacts are retained for 30 days, sufficient for the notebook generation workflow that follows.

**`[skip ci]` tag:** The automated git commits from CI include `[skip ci]` to prevent infinite workflow loops.

**Manual triggers:** all three workflows support `workflow_dispatch` for on-demand runs. The notebook workflow accepts an optional `analyze_run_id` input to re-use a prior analysis artifact.

---

## 9. Configuration Reference

All parameters live in `config.json`. The file is the canonical reference; `pipeline.py` loads it via `load_config()` and `build_search_spaces()`.

### `regimes.breakout`
| Key | Value | Description |
|---|---|---|
| `bo_window` | `150` | Rolling channel lookback for breakout detection |
| `relative` | `true` | Use relative prices (vs benchmark) |

### `regimes.turtle`
| Key | Value | Description |
|---|---|---|
| `fast_window` | `20` | Trailing stop lookback |
| `slow_window` | `50` | Direction signal lookback |
| `relative` | `true` | Use relative prices |

### `regimes.ma_crossover`
| Key | Value | Description |
|---|---|---|
| `short_window` | `50` | Short MA period |
| `medium_window` | `100` | Medium MA period |
| `long_window` | `150` | Long MA period |
| `ma_type` | `["sma","ema"]` | Both simple and exponential MAs generated |
| `relative` | `true` | Use relative prices |

### `regimes.floor_ceiling`
| Key | Value | Description |
|---|---|---|
| `vlty_n` | `63` | Volatility window (~1 quarter) |
| `dist_pct` | `0.05` | Price-to-level distance threshold (5%) |
| `retrace_pct` | `0.05` | Retracement to confirm regime change (5%) |
| `threshold` | `1.5` | ATR multiple for level validation |

### `stop_loss`
| Key | Value | Description |
|---|---|---|
| `atr_window` | `14` | ATR rolling window |
| `atr_multiplier` | `2.0` | Stop distance = ATR × this value |
| `swing_window` | `50` | Swing high/low lookback (available but not currently used) |
| `retracement_level` | `0.618` | Fibonacci retracement (available but not currently used) |
| `min_distance` | `0.5` | Minimum stop distance in % (available but not currently used) |
| `max_distance` | `4.0` | Maximum stop distance in % (available but not currently used) |

### `returns`
| Key | Value | Description |
|---|---|---|
| `fast` | `20` | Short-term return window |
| `relative` | `false` | Use absolute prices for returns |

### `metrics`
| Key | Value | Description |
|---|---|---|
| `risk_window` | `252` | Rolling risk metric window (1 trading year) |
| `percentile` | `0.05` | VaR percentile (5th = 95% confidence) |
| `limit` | `5` | Max signals per metric output |

### `position_sizing`
| Key | Value | Description |
|---|---|---|
| `starting_capital` | `100000` | Portfolio starting capital (€) |
| `lot` | `1` | Minimum lot size |
| `equal_weight` | `0.05` | 5% of capital per position |
| `amortized_root` | `2` | Concave sizing root (square root position sizing) |

### `benchmark`
| Key | Value | Description |
|---|---|---|
| `benchmark` | `"FTSEMIB.MI"` | Reference index for relative price normalization |

---

## 10. Dependencies

### Private package (must be installed manually)

```bash
pip install algoshort-0.1.1-py3-none-any.whl
```

The `algoshort` wheel is **not on PyPI**. It must be installed from the local `.whl` file included in the repo root. It provides all core quantitative logic:

| Module | Purpose |
|---|---|
| `YFinanceDataHandler` | Bulk Yahoo Finance downloads with caching and chunking |
| `OHLCProcessor` | Relative price normalization vs benchmark |
| `RegimeBO` | Breakout and Turtle Trader signal generation |
| `TripleMACrossoverRegime` | Triple MA crossover signal generation |
| `RegimeFC` | Floor/Ceiling regime detection |
| `ReturnsCalculator` | Cumulative returns per signal |
| `StopLossCalculator` | ATR-based stop-loss computation |
| `PositionSizing` | Equal/constant/concave/convex position sizing |
| `trading_summary` | Action detection and report formatting |

### Public packages

```bash
pip install yfinance pandas openpyxl pyarrow joblib tqdm papermill ipykernel matplotlib
```

| Package | Purpose |
|---|---|
| `yfinance` | Yahoo Finance data download (used internally by `algoshort`) |
| `pandas` | Data manipulation and Parquet I/O |
| `openpyxl` | Excel read/write (ticker list, dashboard, snapshot) |
| `pyarrow` | Parquet serialization engine |
| `joblib` | Parallel processing |
| `tqdm` | Progress bars |
| `papermill` | Parametrized Jupyter notebook execution |
| `ipykernel` | Jupyter kernel for papermill |
| `matplotlib` | Plots inside symbol notebooks |

### Full local setup

```bash
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
pip install yfinance openpyxl pyarrow joblib
pip install algoshort-0.1.1-py3-none-any.whl
```
