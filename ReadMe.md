# Scalping Trading Bot

A machine-learning-driven forex scalping bot built on MetaTrader 5 (MT5), with a full **Put-Call Parity (PCP)** engine for arbitrage detection and synthetic position hedging.

[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.95+-green.svg)](https://fastapi.tiangolo.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PyPI version](https://img.shields.io/pypi/v/wifi-densepose.svg)](https://pypi.org/project/wifi-densepose/)
[![PyPI downloads](https://img.shields.io/pypi/dm/wifi-densepose.svg)](https://pypi.org/project/wifi-densepose/)
[![Test Coverage](https://img.shields.io/badge/coverage-100%25-brightgreen.svg)](https://github.com/ruvnet/wifi-densepose)
[![Docker](https://img.shields.io/badge/docker-ready-blue.svg)](https://hub.docker.com/r/ruvnet/wifi-densepose)

---

## Table of Contents

- [Overview](#overview)
- [Project Structure](#project-structure)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Workflow](#workflow)
  - [1. Generate Synthetic Data](#1-generate-synthetic-data)
  - [2. Train the Model](#2-train-the-model)
  - [3. Run Live Trading](#3-run-live-trading)
  - [4. Monitor Positions](#4-monitor-positions)
  - [5. Parity Check](#5-parity-check)
  - [6. Arbitrage Scan](#6-arbitrage-scan)
- [CLI Reference](#cli-reference)
- [Module Reference](#module-reference)
  - [main.py](#mainpy)
  - [src/live_bot.py](#srclive_botpy)
  - [src/pcp_hedge.py](#srcpcp_hedgepy)
  - [src/indicators.py](#srcindicatorspy)
  - [src/train.py](#srctrainpy)
  - [src/predict.py](#srcpredictpy)
  - [src/utils.py](#srcutilspy)
  - [src/monitor.py](#srcmonitorpy)
  - [src/check_stoploss.py](#srccheck_stoploss_py)
- [Put-Call Parity Engine](#put-call-parity-engine)
  - [The Formula](#the-formula)
  - [Arbitrage Detection](#arbitrage-detection)
  - [Synthetic Hedging](#synthetic-hedging)
  - [Combined Mode](#combined-mode)
  - [Transaction Cost Filtering](#transaction-cost-filtering)
  - [The Bond Leg](#the-bond-leg)
- [Model & Features](#model--features)
- [Risk Management](#risk-management)
- [Configuration Reference](#configuration-reference)
- [Broker Setup](#broker-setup)
- [Important Warnings](#important-warnings)

---

## Overview

This bot does three things:

1. **Scalp spot forex** — a Random Forest classifier reads 1-minute bars, computes technical indicators, and generates `buy`, `sell`, or `hold` signals. Orders are placed on MT5 with stop loss and take profit.

2. **Detect put-call parity violations** — every poll cycle, the bot evaluates `C + PV(X) = P + S` across configured symbols. When the equation breaks and the net profit after transaction costs exceeds a threshold, an arbitrage signal is raised. Trades can be executed automatically or logged for manual review.

3. **Hedge spot positions synthetically** — after every spot trade, the bot builds a synthetic counter-position from options fetched directly from MT5, protecting the open position against adverse moves.

---

## Project Structure

```
.
├── main.py                        # CLI entry point — all actions run from here
├── models/
│   ├── trained_scalping_model.pkl # Saved RandomForest model + feature names
│   ├── signal_label_encoder.pkl   # Encodes buy/sell/hold labels
│   └── symbol_label_encoder.pkl   # Encodes symbol strings
├── data/
│   ├── generate_synthetic.py      # Synthetic OHLCV data generator
│   └── scalping_large_dataset.csv # Training data (generated or real)
└── src/
    ├── live_bot.py                # Live trading loop
    ├── pcp_hedge.py               # Put-Call Parity engine (arb + hedge)
    ├── indicators.py              # Technical indicator computation
    ├── train.py                   # Model training pipeline
    ├── predict.py                 # Single-row prediction utility
    ├── preprocess.py              # Data loading and feature preparation
    ├── utils.py                   # MT5 connection, order execution, helpers
    ├── monitor.py                 # Real-time dashboard for open positions
    └── check_stoploss.py          # Diagnostic tool for stop loss coverage
```

---

## Requirements

- Python 3.8+
- MetaTrader 5 terminal installed and running (Windows)
- A broker account with MT5 access
- Options symbols available on your broker (required for PCP hedging/arbitrage)

**Python dependencies:**

```
MetaTrader5
pandas
numpy
scikit-learn
joblib
tabulate
```


## 🚀 Installation

### Prerequisites
- Python 3.8+
- MetaTrader 5 installed
- Demo or live trading account

### Step-by-Step Setup

1. **Clone and Setup Environment**
```bash
git clone <repository-url>
cd pesser
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
```

2. **Install Dependencies**
```bash
pip install -U pip
pip install pandas numpy scikit-learn joblib matplotlib 
pip install MetaTrader5 pyzmq python-dotenv tabulate
```
### Complete Workflow

1. **Generate Training Data**
```bash
python main.py gen-data --minutes 5000
```

2. **Train the Model**
```bash
python main.py train --csv data/scalping_large_dataset.csv
```

3. **Start Live Trading**
```bash
python main.py live --symbol EURUSD --lots 0.01 --sl 8 --tp 12 --mt5-path "C:/Program Files/MetaTrader 5/terminal64.exe"
```

4. **Monitor Performance** (in separate terminal)
```bash
python main.py monitor --symbol EURUSD --interval 5 --mt5-path "C:/Program Files/MetaTrader 5/terminal64.exe"
```

Ensure your MT5 terminal is open and logged into your broker account before running any command.

---

## Quick Start

```bash
# 1. Generate synthetic training data
python main.py gen-data --minutes 5000

# 2. Train the model
python main.py train

# 3. Run the live bot (hedge ON, arb detection ON, observe-only)
python main.py live --symbol EURUSD --lots 0.01

# 4. Open the monitor in a second terminal
python main.py monitor --symbol EURUSD
```

---

## Workflow

### 1. Generate Synthetic Data

Generates synthetic OHLCV data for model training when real historical data is unavailable.

```bash
python main.py gen-data --csv data/my_data.csv --minutes 5000
```

Output: a CSV with columns `timestamp, open, high, low, close, volume, symbol, signal`.

---

### 2. Train the Model

Loads the dataset, computes all indicators, trains a Random Forest classifier, and saves the model to `models/`.

```bash
python main.py train --csv data/scalping_large_dataset.csv
```

After training, the console prints:
- Train and test accuracy
- Full classification report (precision, recall, F1 per class)
- Signal distribution in the training set

Saved files:
- `models/trained_scalping_model.pkl` — model + feature names + training date
- `models/signal_label_encoder.pkl` — label encoder for `buy/sell/hold`
- `models/symbol_label_encoder.pkl` — label encoder for symbol names

---

### 3. Run Live Trading

Starts the live trading loop. On each poll cycle the bot:

1. Fetches the last `N` 1-minute bars from MT5
2. Computes indicators and builds the feature vector
3. Runs the classifier → `buy`, `sell`, or `hold`
4. If a trade signal fires and no position is already open, places a spot order with SL/TP
5. Runs `hedge_with_arb_check()` — detects arbitrage and builds the synthetic hedge for the new position

```bash
# Default: hedge ON, arb detection ON, auto-execute arb OFF
python main.py live --symbol EURUSD --lots 0.01 --sl 8 --tp 12

# Also auto-execute arbitrage trades when found
python main.py live --symbol EURUSD --auto-execute-arb

# Hedge only — no arb scan
python main.py live --symbol EURUSD --no-arb

# No hedge, no arb — plain signal bot
python main.py live --symbol EURUSD --no-hedge --no-arb

# Scan multiple symbols for arb while trading EURUSD
python main.py live --symbol EURUSD --arb-symbols EURUSD,GBPUSD,USDJPY
```

---

### 4. Monitor Positions

Opens a refreshing terminal dashboard showing account summary, all open positions with pip P/L, today's trade history, and performance stats.

```bash
# Monitor all symbols
python main.py monitor

# Monitor one symbol, refresh every 10 seconds
python main.py monitor --symbol EURUSD --interval 10
```

Dashboard sections:
- **Account Summary** — balance, equity, margin, free margin, floating P/L
- **Open Positions** — ticket, symbol, type, volume, open/current price, SL, TP, pips, profit
- **Today's Performance** — total trades, win rate, avg win/loss, profit factor, best/worst trade
- **Recent Trade History** — last 20 closed deals

---

### 5. Parity Check

Fetches live option prices for a symbol and prints the full put-call parity breakdown. No trades are placed.

```bash
python main.py parity-check --symbol EURUSD --option-days 30 --risk-free-rate 0.05
```

Example output:
```
=================================================================
  PUT-CALL PARITY  —  C + PV(X) = P + S
=================================================================
  Underlying  : EURUSD
  Spot   (S)  : 1.08250
  Strike (X)  : 1.08250
  PV(X)       : 1.07808  [X·e^(-rT), r=0.05, T=0.0822yr]
  Call   (C)  : 0.00310
  Put    (P)  : 0.00280
  LHS C+PV(X) : 1.08118
  RHS P+S     : 1.08530
  Difference  : -0.00412
  Parity holds: NO  ✗  (violation detected)
=================================================================
```

---

### 6. Arbitrage Scan

Scans a list of symbols for parity violations in a single pass. Prints all signals and flags which ones are executable after transaction costs. No trades are placed.

```bash
python main.py arb-scan \
  --arb-symbols EURUSD,GBPUSD,USDJPY,XAUUSD \
  --option-days 30 \
  --min-arb-profit 0.0002
```

---

## CLI Reference

All commands follow the pattern:

```bash
python main.py <action> [options]
```

### Actions

| Action | Description |
|---|---|
| `gen-data` | Generate synthetic OHLCV training data |
| `train` | Train the Random Forest model |
| `live` | Start live trading loop |
| `monitor` | Open real-time position dashboard |
| `parity-check` | Print parity breakdown for one symbol |
| `arb-scan` | Scan symbols for executable arbitrage |

### All Options

| Flag | Default | Description |
|---|---|---|
| `--symbol` | `EURUSD` | Primary trading symbol |
| `--lots` | `0.01` | Lot size for spot orders |
| `--sl` | `8` | Stop loss in pips |
| `--tp` | `12` | Take profit in pips |
| `--csv` | `data/scalping_large_dataset.csv` | Path to training CSV |
| `--minutes` | `3000` | Minutes of synthetic data to generate |
| `--mt5-path` | `None` | Path to MT5 terminal executable |
| `--interval` | `5` | Monitor refresh interval (seconds) |
| `--poll-interval` | `10` | Live bot polling interval (seconds) |
| `--no-hedge` | `False` | Disable synthetic hedge |
| `--no-arb` | `False` | Disable arbitrage detection |
| `--auto-execute-arb` | `False` | Auto-execute arb trades when found |
| `--hedge-ratio` | `1.0` | Fraction of spot position to hedge (0.0–1.0) |
| `--risk-free-rate` | `0.05` | Annual risk-free rate for PV(X) calculation |
| `--option-days` | `30` | Days to option expiry |
| `--option-expiry` | auto | Override expiry date `YYYYMMDD` |
| `--min-arb-profit` | `0.0003` | Min net profit to mark arb as executable |
| `--arb-symbols` | trading symbol | Comma-separated symbols to scan for arb |

---

## Module Reference

### `main.py`

CLI entry point. Parses arguments and dispatches to the appropriate function. Also contains `run_parity_check()` and `run_arb_scan()` which handle MT5 initialization and print formatted results.

---

### `src/live_bot.py`

#### `run_live(...)`

The main trading loop. Runs indefinitely until `KeyboardInterrupt`.

**Startup sequence:**
1. Connect to MT5
2. Load model, label encoder, symbol encoder
3. Resolve option expiry date (3rd Friday of target month)
4. Run startup arb scan or parity check (observe-only)

**Each poll cycle:**
1. Fetch last `window` 1-minute bars via `get_latest_ticks()`
2. Compute indicators via `add_all_indicators()`
3. Build feature vector, run `clf.predict()`
4. Skip if a position is already open for the symbol
5. Place spot order with SL/TP
6. Call `hedge_with_arb_check()` → detect arb + build hedge
7. Execute hedge legs via `execute_synthetic_hedge()`
8. Sleep `poll_interval` seconds

**Key parameters:**

| Parameter | Description |
|---|---|
| `use_pcp_hedge` | Master switch for hedging |
| `use_arb_detection` | Enable per-cycle arb scan |
| `auto_execute_arb` | Fire arb trades automatically |
| `hedge_ratio` | `1.0` = full hedge, `0.5` = half hedge |
| `arb_scan_symbols` | Extra symbols scanned each cycle |

---

### `src/pcp_hedge.py`

The Put-Call Parity engine. Contains all data classes, parity math, arbitrage detection, synthetic hedge construction, and order execution for options.

#### Data Classes

**`OptionContract`**

Represents a single call or put fetched from MT5.

| Field | Type | Description |
|---|---|---|
| `symbol` | `str` | Full MT5 option symbol string |
| `option_type` | `str` | `'call'` or `'put'` |
| `strike` | `float` | Strike price |
| `expiry` | `str` | Expiry date `YYYYMMDD` |
| `bid` | `float` | MT5 bid price |
| `ask` | `float` | MT5 ask price |
| `.mid` | `float` | `(bid + ask) / 2` |
| `.spread` | `float` | `ask - bid` |

**`ParityResult`**

Output of `compute_parity()`.

| Field | Description |
|---|---|
| `lhs` | `C + PV(X)` |
| `rhs` | `P + S` |
| `parity_diff` | `lhs - rhs` (`+` = LHS expensive, `−` = LHS cheap) |
| `parity_holds` | `True` if `\|diff\| ≤ tolerance` |
| `pv_strike` | `X * e^(-rT)` |

**`ArbitrageSignal`**

Output of `detect_arbitrage()`.

| Field | Description |
|---|---|
| `direction` | `'buy_lhs'` or `'sell_lhs'` |
| `legs` | List of `(symbol, action, reason)` tuples |
| `gross_profit` | `\|parity_diff\|` before costs |
| `transaction_cost_estimate` | Estimated half-spread costs |
| `net_profit` | `gross - costs` |
| `is_executable` | `True` if `net_profit ≥ min_profit_threshold` |

**`SyntheticHedge`**

Output of `build_synthetic_hedge()`.

| Field | Description |
|---|---|
| `direction` | `'synthetic_long'` or `'synthetic_short'` |
| `call_action` | `'buy'` or `'sell'` |
| `put_action` | `'buy'` or `'sell'` |
| `net_cost` | Option premium paid minus received |
| `hedge_ratio` | Lots for the option legs |

#### Public Functions

| Function | Description |
|---|---|
| `compute_parity(spot, call, put, r, T)` | Evaluate both sides of `C + PV(X) = P + S` |
| `detect_arbitrage(underlying, expiry, ...)` | Find parity violations and build 4-leg arb signal |
| `execute_arbitrage(signal, lots)` | Fire tradeable arb legs on MT5 |
| `scan_arbitrage_opportunities(symbols, ...)` | Batch scan across multiple underlyings |
| `build_synthetic_hedge(direction, ...)` | Build option legs to hedge a spot position |
| `execute_synthetic_hedge(hedge, lots)` | Place both option legs on MT5 |
| `hedge_with_arb_check(direction, ...)` | Combined: detect arb + build hedge atomically |
| `check_parity_for_symbol(underlying, ...)` | Standalone parity check, no trades |
| `find_atm_options(underlying, expiry)` | Fetch nearest ATM call + put from MT5 |
| `build_option_symbol(underlying, type, K, expiry)` | Build broker-formatted option symbol string |

---

### `src/indicators.py`

#### `add_all_indicators(df) → DataFrame`

Computes all features used by the model. Operates per-symbol via `groupby`. Input DataFrame must have columns `timestamp, open, high, low, close, volume, symbol`.

| Feature | Description |
|---|---|
| `hl_range` | `high - low` |
| `oc_change` | `close - open` |
| `return` | Percentage change in close |
| `ema_5` | 5-period exponential moving average |
| `ema_20` | 20-period exponential moving average |
| `sma_5` | 5-period simple moving average |
| `sma_20` | 20-period simple moving average |
| `rsi` | 14-period RSI (Wilder's smoothing, fills NaN with 50) |
| `atr` | 14-period Average True Range |

---

### `src/train.py`

#### `train_model(csv_path, model_out_path=None)`

Full training pipeline:

1. Load dataset via `load_dataset()`
2. Encode symbols via `prepare_features()` → adds `symbol_enc`
3. Compute indicators via `add_all_indicators()`
4. Train `RandomForestClassifier` with 80/20 stratified split
5. Save model dict (model + feature names + training date) and both encoders

**Classifier hyperparameters:**

| Param | Value |
|---|---|
| `n_estimators` | 100 |
| `max_depth` | 10 |
| `min_samples_split` | 20 |
| `min_samples_leaf` | 10 |
| `max_features` | `'sqrt'` |

---

### `src/predict.py`

#### `predict_from_row(row_dict) → str`

Single-row inference. Loads model and encoders, constructs a DataFrame, runs `add_all_indicators()`, and returns a `'buy'`, `'sell'`, or `'hold'` string.

Useful for testing predictions outside the live loop:

```python
from src.predict import predict_from_row

signal = predict_from_row({
    "timestamp": "2025-06-01 10:00:00",
    "symbol": "EURUSD",
    "open": 1.08200,
    "high": 1.08350,
    "low": 1.08180,
    "close": 1.08310,
    "volume": 1200,
})
print(signal)  # 'buy', 'sell', or 'hold'
```

---

### `src/utils.py`

MT5 connectivity and order execution layer.

#### Connection

| Function | Description |
|---|---|
| `connect_mt5(login, password, server, path)` | Initialize MT5, optionally log in |
| `disconnect_mt5()` | Shutdown MT5 connection |

#### Data Retrieval

| Function | Description |
|---|---|
| `get_latest_ticks(symbol, n)` | Fetch last `n` 1-minute bars as DataFrame |
| `get_account_info()` | Return MT5 account info object |
| `get_open_positions(symbol)` | Return list of open positions |

#### Order Execution

| Function | Description |
|---|---|
| `place_order_market_improved(symbol, type, lots, sl_pips, tp_pips, ...)` | Primary execution with retry logic and filling mode fallback |
| `place_order_with_slippage_check(symbol, type, lots, ...)` | Rejects fills with slippage above `max_slippage_pips` |
| `close_position(position, ...)` | Close a specific open position with retry |
| `place_order(...)` | Legacy wrapper — redirects to `place_order_market_improved` |

#### Stop Loss Management

| Function | Description |
|---|---|
| `add_stop_loss_to_position(ticket, sl_pips)` | Add SL to a single position |
| `add_stop_loss_to_all_positions(sl_pips)` | Add SL to every position missing one |
| `check_and_fix_positions()` | Audit all positions and auto-fix missing SLs |

**Retry and filling logic in `place_order_market_improved`:**
- Up to `max_retries` attempts
- Attempts 1–2 use `ORDER_FILLING_FOK`, attempts 3+ fall back to `ORDER_FILLING_RETURN`
- Deviation increases by 10 points per attempt
- Handles `REQUOTE`, `PRICE_OFF`, `INVALID_FILL` retcodes gracefully

---

### `src/monitor.py`

#### `TradingMonitor`

Real-time dashboard class.

```python
monitor = TradingMonitor(symbol="EURUSD")
monitor.run_monitor(refresh_interval=5)
```

| Method | Description |
|---|---|
| `get_account_summary()` | Balance, equity, margin, free margin, floating P/L |
| `get_open_positions()` | DataFrame of open positions with pip calculation |
| `get_today_history()` | Last 20 closed deals today |
| `get_performance_stats()` | Win rate, profit factor, best/worst trade |
| `display_dashboard()` | Render full dashboard to terminal |
| `run_monitor(refresh_interval)` | Loop with screen clear and refresh |

Can also be run standalone:

```bash
python src/monitor.py --symbol EURUSD --interval 5
```

---

### `src/check_stoploss.py`

#### `check_stop_loss_status()`

Diagnostic tool. Scans all open positions, prints their SL status, calculates pip distance to SL, and automatically calls `add_stop_loss_to_all_positions()` if any position is missing a stop loss. Verifies the fix afterward.

#### `test_stop_loss_calculation()`

Runs test cases for BUY/SELL SL price calculation across EURUSD and USDJPY to verify pip_size logic is correct for your broker's digit convention.

Run directly:

```bash
python src/check_stoploss.py
```

---

## Put-Call Parity Engine

### The Formula

```
C + PV(X) = P + S
```

Where:
- **C** = Call option price (mid of bid/ask)
- **PV(X)** = Present value of strike: `X × e^(−rT)`
- **P** = Put option price (mid of bid/ask)
- **S** = Spot price (mid of bid/ask)
- **X** = Strike price (ATM = nearest standard increment to spot)
- **r** = Annual risk-free rate (default `0.05`)
- **T** = Time to expiry in years (`days / 365`)

In efficient markets this equation holds exactly. When it doesn't, a riskless profit can be constructed by simultaneously buying the cheap side and selling the expensive side.

---

### Arbitrage Detection

`detect_arbitrage()` computes both sides and evaluates the signed difference:

```
parity_diff = (C + PV(X)) − (P + S)
```

**Violation A — LHS cheap** (`parity_diff < 0`):

The call + bond package is underpriced relative to the put + spot package.

| Leg | Action | Rationale |
|---|---|---|
| Call | Buy | Long the underpriced derivative |
| Bond | Invest PV(X) | Holds X at expiry to exercise the call |
| Put | Sell | Short the overpriced synthetic leg |
| Spot | Short | Short the overpriced cash leg |

**Violation B — LHS expensive** (`parity_diff > 0`):

The call + bond package is overpriced.

| Leg | Action | Rationale |
|---|---|---|
| Call | Sell | Short the overpriced derivative |
| Bond | Borrow PV(X) | Fund the spot purchase |
| Put | Buy | Long the underpriced synthetic leg |
| Spot | Buy | Long the underpriced cash leg |

At expiry both sides of the equation converge and the difference is realised as profit.

---

### Synthetic Hedging

After every spot trade, `build_synthetic_hedge()` constructs a counter-position from options that replicates the opposite exposure, neutralising directional risk.

**Spot BUY → Synthetic Short:**

```
Synthetic Short = Buy Put + Sell Call
```

This replicates `P + S` on the hedge side, offsetting the long spot.

**Spot SELL → Synthetic Long:**

```
Synthetic Long = Buy Call + Sell Put
```

This replicates `C + PV(X)` on the hedge side, offsetting the short spot.

The `hedge_ratio` parameter controls how much of the position is hedged:
- `1.0` = full hedge (default)
- `0.5` = half hedge
- `0.0` = no hedge (same as `--no-hedge`)

---

### Combined Mode

`hedge_with_arb_check()` is the recommended entry point when running the full bot. It runs both operations atomically in a defined order:

```
Step 1: detect_arbitrage()
          ↓ if executable and auto_execute_arb=True
        execute_arbitrage()

Step 2: build_synthetic_hedge()
          ↓ always runs
        (caller calls execute_synthetic_hedge())
```

This allows the bot to simultaneously capture any mispricing profit **and** protect the spot position in the same call.

---

### Transaction Cost Filtering

Raw parity violations are common due to bid-ask spreads. The engine estimates costs before flagging a signal as executable:

```
transaction_cost = (call.spread/2 + put.spread/2 + spot.spread/2) × 2
```

The `× 2` accounts for both entry and exit. A signal is only marked `is_executable = True` when:

```
net_profit = |parity_diff| − transaction_cost ≥ min_arb_profit
```

Default `min_arb_profit` is `0.0003`. Lower this to surface more signals; raise it to filter out marginal ones.

---

### The Bond Leg

The arbitrage trade requires investing or borrowing `PV(X)` at the risk-free rate. This leg **cannot be placed on MT5 directly**. In `execute_arbitrage()`, the bond leg is logged as `manual_required` and skipped. In practice this means:

- For `invest`: place a risk-free deposit, T-bill, or equivalent through your broker's money market desk
- For `borrow`: arrange a margin loan or repo agreement at the risk-free rate

All option and spot legs are executed on MT5 automatically. The bond leg is the only manual step.

---

## Model & Features

The classifier is a `RandomForestClassifier` that predicts one of three classes: `buy`, `sell`, `hold`.

**Feature vector (15 features):**

| Feature | Type | Description |
|---|---|---|
| `open` | Price | Bar open |
| `high` | Price | Bar high |
| `low` | Price | Bar low |
| `close` | Price | Bar close |
| `volume` | Count | Tick volume |
| `hl_range` | Derived | `high - low` |
| `oc_change` | Derived | `close - open` |
| `return` | Derived | Percentage change in close |
| `ema_5` | Indicator | 5-period EMA |
| `ema_20` | Indicator | 20-period EMA |
| `sma_5` | Indicator | 5-period SMA |
| `sma_20` | Indicator | 20-period SMA |
| `rsi` | Indicator | 14-period RSI |
| `atr` | Indicator | 14-period ATR |
| `symbol_enc` | Encoded | Integer-encoded symbol name |

The model object saved to disk is a dictionary containing the classifier, feature names in order, feature count, and training timestamp. This guarantees the live bot always aligns its feature vector to what the model was trained on.

---

## Risk Management

The bot has several layers of risk protection:

**Stop Loss on every trade.** Every spot order is placed with a stop loss (`--sl` pips, default 8). The `check_stoploss.py` diagnostic and `check_and_fix_positions()` utility automatically detect and fix any positions missing a stop loss.

**One position per symbol.** The live loop checks `mt5.positions_get(symbol=symbol)` before every trade. If a position is already open it skips that cycle, preventing pyramid entries.

**Slippage control.** `place_order_with_slippage_check()` measures the difference between the price at signal time and the executed price. If slippage exceeds `max_slippage_pips`, the position is closed immediately.

**Retry with escalating deviation.** `place_order_market_improved()` retries up to 5 times, increasing the allowed deviation by 10 points per attempt and switching from `FOK` to `RETURN` filling after the second attempt.

**Partial hedge alerting.** If the call leg of a synthetic hedge executes but the put leg fails, the bot logs a `PARTIAL HEDGE` warning and leaves the call leg open for manual resolution rather than silently abandoning it.

**Partial arbitrage alerting.** If any leg of an arb trade fails, `execute_arbitrage()` logs all successfully opened legs and flags the outcome as `PARTIAL` so orphaned positions can be identified and closed.

---

## Configuration Reference

The most important parameters to tune for your setup:

| Parameter | Where | Notes |
|---|---|---|
| `--sl` | CLI | Default 8 pips. Tighten for scalping, loosen for volatile sessions. |
| `--tp` | CLI | Default 12 pips. Aim for TP/SL ratio ≥ 1.5. |
| `--poll-interval` | CLI | Default 10s. Lower = more responsive, higher CPU/API load. |
| `--hedge-ratio` | CLI | `1.0` = full hedge. Reduce to lower option premium cost. |
| `--min-arb-profit` | CLI | Default `0.0003`. Lower to surface more signals on tight spreads. |
| `--risk-free-rate` | CLI | Match current overnight/SOFR rate for your currency pair. |
| `--option-days` | CLI | Match the tenor of options your broker offers. |
| `build_option_symbol()` | `pcp_hedge.py` | **Must be edited** to match your broker's symbol format. |
| `tolerance` in `compute_parity()` | `pcp_hedge.py` | Default `0.0005`. Tighten for liquid instruments. |

---

## Broker Setup

Before running the bot with PCP features:

1. **Confirm options are available.** Not all MT5 brokers offer forex options. Check your Market Watch for option symbols.

2. **Identify your broker's option symbol format.** Open `src/pcp_hedge.py` and edit `build_option_symbol()` to match. Common formats:
   ```
   EURUSD-C-1.1000-20250620   (default in this codebase)
   EURUSD.C.11000.20250620
   #EURUSD_C_1100_062025
   ```

3. **Add option symbols to Market Watch.** MT5 requires symbols to be visible in Market Watch before they can be traded. The bot calls `mt5.symbol_select(symbol, True)` automatically, but some brokers require manual addition.

4. **Verify option tick data.** Run `parity-check` before going live to confirm the bot can fetch bid/ask for your option symbols.

5. **Strike increments.** The bot auto-rounds spot to the nearest standard strike. Defaults are `0.0050` for forex majors, `0.50` for JPY pairs, `5.0` for XAUUSD. Edit `_round_to_nearest_strike()` if your broker uses different increments.

---

## 🚀 Future Enhancements

### Short-term Goals
- [ ] Add more technical indicators (MACD, Bollinger Bands)
- [ ] Implement portfolio diversification
- [ ] Add trading session filters
- [ ] Enhance risk management with trailing stops

### Medium-term Goals
- [ ] Deep learning models (LSTM, Transformer)
- [ ] Multi-timeframe analysis
- [ ] News sentiment integration
- [ ] Backtesting framework

### Long-term Vision
- [ ] Cloud deployment and scaling
- [ ] Web-based dashboard
- [ ] Mobile notifications
- [ ] Multi-broker support


## Important Warnings

**This bot trades real money.** Always test on a demo account first. Verify every component — signal generation, order execution, hedging, and arb leg placement — in a demo environment before enabling on a live account.

**Arbitrage in forex options is rare and fleeting.** Real violations large enough to survive transaction costs are uncommon in liquid markets. The arb detection is most useful as a monitoring tool and to ensure the hedge is being built at fair prices.

**The bond leg is manual.** The `invest`/`borrow` leg of every arbitrage trade must be handled outside MT5. Running arb trades without the bond leg converts a risk-free arbitrage into a directional position. Only enable `--auto-execute-arb` if you have a process for managing the bond leg.

**Partial hedges leave open exposure.** If one leg of a synthetic hedge fails, the other remains open. Monitor the logs for `PARTIAL HEDGE` warnings and close orphaned option legs promptly.

**Option liquidity.** Forex options, especially OTC, can have wide spreads and low liquidity. Always check that `min-arb-profit` is set above the typical spread cost for your broker before enabling auto-execution.