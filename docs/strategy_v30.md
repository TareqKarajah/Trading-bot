# Strategy v3.0 Master Reference

## 1. Module Pipeline & Responsibilities

The system is organized as a pipeline of loosely coupled modules, each with a single responsibility.

| Module | Single Responsibility | Output | Must NOT Do |
|---|---|---|---|
| **Data Ingestion** | Normalise OHLCV; detect gaps; maintain buffers | OHLCV events per timeframe | Strategy logic, risk logic |
| **Macro Bias Engine** | Classify gold macro environment | `MACRO_STATE` + confidence [0–1] | Trade directly; be a signal source |
| **Volatility Regime Engine** | Classify volatility state with hysteresis | `REGIME_STATE` + `risk_scalar` | Dispatch strategy permissions |
| **Strategy Dispatcher** | Map (regime, macro, time, session) → permissions | `DispatcherPermissions` object | Emit orders or signals |
| **Strategy Engines** | Detect setup; score it; package intent | `SignalIntent` object | Size position; check risk; execute |
| **Risk Engine** | Transform intent into sized, stopped decision | `RiskDecision` object | Execute orders; manage portfolio |
| **Portfolio Coordinator** | Resolve conflicts; cap heat; weight quality | Updated `RiskDecision` or rejection | Generate signals; execute |
| **Execution Quality Gate** | Check real-world execution feasibility | `ExecutionDecision` object | Trade; size; generate signals |
| **Execution Engine** | Submit and manage orders | Order state transitions + audit log | Generate signals; size; risk-check |
| **Monitoring & Audit** | Observe system health; emit alerts | Metrics + alert events | Modify system behaviour |

---

## 2. Strategy Summaries

### 2.1 Swing Strategy — TSMOM Multi-Horizon
*   **Concept:** Time Series Momentum across 1M, 3M, 6M, 12M horizons.
*   **Signal:** Inverse-volatility weighted score of momentum signs.
*   **Brake:** Activates if short-term momentum diverges from trend in high vol.
*   **Horizon Weights:** 1M=0.10, 3M=0.20, 6M=0.30, 12M=0.40.
*   **Rebalance:** Every Monday 00:05 UTC.

### 2.2 ORB Strategy — Opening Range Breakout
*   **Concept:** Breakout from London Opening Range (07:00–08:00 UTC).
*   **Logic:** Long if `Close > OR_High + Buffer`; Short if `Close < OR_Low - Buffer`.
*   **Buffer:** Adaptive based on OR width vs median.
*   **Constraints:** One trade per session; no re-entry.

### 2.3 News Breakout Strategy
*   **Concept:** Volatility expansion post-event (NFP, FOMC, CPI).
*   **Tiers:** T1 (Always trade), T2 (Spread check), T3 (High vol check).
*   **Entry:** Breakout from pre-news range (5 bars M5).
*   **Expiry:** Strict time limit (1-3 mins) to enter.

### 2.4 Scalping Strategy
*   **Concept:** Mean reversion in low volatility regimes.
*   **Logic:** Bollinger Band breakout + RSI overbought/sold + Trend filter.
*   **Gate:** Strictly `LOW_VOL` regime only.
*   **Time:** 08:00–17:00 UTC only.

---

## 3. Strategy Dispatcher Priority

| Priority | Condition | Scalp | ORB | News | Swing | Direction |
|---|---|---|---|---|---|---|
| 1 | `SHOCK_EVENT` or approved news active | ✗ | ✗ | ✗ | ✓ | Monitor (From macro) |
| 2 | Blackout (±10 min around HIGH/CRITICAL news) | ✗ | ✗ | ✗ | Monitor | Monitor (From macro) |
| 3 | London open 07:00–08:00 UTC; no shock; no blackout | ✗ | ✓ | Standby | ✓ | From `swing_dir` |
| 4 | `LOW_VOL` + liquid session (07:00–17:00); no event | ✓ | ✗ | ✗ | ✓ | From `swing_dir` |
| 5 | Any other time | ✗ | ✗ | ✗ | ✓ | From `swing_dir` |

---

## 4. Direction Constraint Rules

Derived from `swing_dir` (TSMOM output) and `macro_state`:

1.  `swing_dir = sign(confidence_score)`
2.  **Rule Map:**
    *   If `MACRO_BULL_GOLD`: `direction_constraint = max(swing_dir, 0)` (Long or Neutral)
    *   If `MACRO_BEAR_GOLD`: `direction_constraint = min(swing_dir, 0)` (Short or Neutral)
    *   If `MACRO_NEUTRAL`: `direction_constraint = swing_dir`
    *   If `MACRO_EVENT_RISK`: `direction_constraint = 0` (Neutral/News only)

---

## 5. Net Exposure Rule

The total exposure across all strategies is capped:

`|pos_swing + pos_orb + pos_scalp| ≤ 1.0` (in risk units)

---

## 6. Risk Budget Allocation

| Strategy | Budget | Risk/Trade |
|---|---|---|
| **Swing TSMOM** | 60% | ~1.0% vol-target |
| **ORB** | 25% | 0.50% |
| **Scalping** | 15% | 0.25% |
| **News Breakout** | From ORB budget | 0.50% |

---

## 7. Numeric Threshold Appendix (Frozen)

### 7.1 Regime Engine
*   **ATR_fast period:** 14 bars M5
*   **ATR_slow period:** 200 bars M5
*   **LOW_VOL upper bound:** ratio < 0.85
*   **HIGH_VOL lower bound:** ratio > 1.25
*   **Hysteresis confirmation:** 3 consecutive bars
*   **Shock candle threshold:** TR > 2.0 × ATR_fast
*   **Shock cooldown:** 12 bars (60 min)
*   **LOW_VOL risk_scalar:** 1.00
*   **MID_VOL risk_scalar:** 0.80
*   **HIGH_VOL risk_scalar:** 0.50
*   **SHOCK risk_scalar:** 0.00

### 7.2 Macro Bias Engine
*   **USD impulse window:** 5-day ROC
*   **MACRO_BULL_GOLD threshold:** USD ROC < −0.005 AND yield ROC < −0.003
*   **MACRO_BEAR_GOLD threshold:** USD ROC > +0.005 AND yield ROC > +0.003
*   **Risk-off VIX threshold:** > 25
*   **Risk-off ATR fallback:** ATR_daily > 2.0 × ATR_baseline_252d
*   **MACRO_EVENT_RISK sizing modifier:** 0.50

### 7.3 Swing TSMOM
*   **Horizon weights:** 1M=0.10, 3M=0.20, 6M=0.30, 12M=0.40
*   **Bullish threshold:** confidence > +0.30
*   **Bearish threshold:** confidence < −0.30
*   **Neutral zone:** −0.30 to +0.30
*   **Turning-point brake 1M threshold:** `mom_1m < −0.50 × StdDev(mom_1m, 52w)`
*   **Turning-point brake ATR threshold:** ATR_fast/ATR_slow > 1.15
*   **Turning-point brake size reduction:** 0.40
*   **Rebalance:** Every Monday 00:05 UTC
*   **Chandelier stop multiplier:** 3.0 × ATR(20, Daily)
*   **Vol target sigma:** 1.0% daily equity risk
*   **Max size cap:** 1.5 × base_lots

### 7.4 ORB
*   **Opening range window:** 07:00–08:00 UTC
*   **Quality band lower:** `max(0.25 × ATR_daily, 0.70 × OR_median_20)`
*   **Quality band upper:** `min(1.80 × ATR_daily, 2.00 × OR_median_20)`
*   **Adaptive buf base:** 0.15 × ATR(14, M15)
*   **Adaptive buf clamp:** [0.10 × ATR, 0.25 × ATR]
*   **Volume confirmation:** > 1.20 × VMA(20, M15)
*   **SL cap:** 1.50 × ATR(14, M15)
*   **Trailing stop:** 2.0 × ATR(14, M15)
*   **Mandatory exit:** 20:45 UTC (19:30 UTC Fridays)
*   **Retest tolerance:** 0.5 × buf_final

### 7.5 News Breakout
*   **Pre-news range window:** 5 M5 bars before event
*   **buf_news:** 0.20 × ATR(14, M5)
*   **Max spread T1:** 3.0 bp
*   **Max spread T2:** 2.5 bp
*   **Max expected slippage:** 5.0 bp
*   **Quote freshness limit:** 500 ms
*   **T1 entry window:** 3 min post-release
*   **T2 entry window:** 2 min post-release
*   **T3 entry window:** 1 min post-release
*   **TP target:** 2.0 × SL_distance
*   **Max chase limit:** 1.5 × SL_distance from trigger
*   **Max per day:** 2 trades

### 7.6 Scalping
*   **Regime gate:** LOW_VOL strictly
*   **BB parameters:** SMA(20), 2.0σ
*   **RSI period / levels:** 14 / < 30 long; > 70 short
*   **EMA trend filter:** EMA(200, M5)
*   **Volume confirmation:** > 1.10 × VMA(20, M5)
*   **Trend-day filter:** `|Close − EMA(200)| < 2.50 × ATR(14)`
*   **Max spread ceiling:** 2.0 bp
*   **SL distance:** 1.0 × ATR(14, M5)
*   **TP1 distance:** 0.5 × ATR(14, M5)
*   **Trailing multiplier:** 1.0 × ATR (post-TP1)
*   **Time stop:** 6 bars = 30 min
*   **Daily consecutive loss halt:** 5 losses
*   **Min viable win rate:** 63% at 0.65R avg win
