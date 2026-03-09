# Signal & Risk Contracts

## 1. SignalIntent Schema

**From:** Strategy Engine  
**To:** Risk Engine

| Field | Type | Description |
|---|---|---|
| `strategy_name` | string | `"SWING"`, `"ORB"`, `"NEWS"`, `"SCALP"` |
| `direction` | int | `+1` (Long), `-1` (Short) |
| `score` | int | 0-5 (Must be ≥ 4 to pass) |
| `entry_type` | string | `"MARKET"`, `"LIMIT"`, `"STOP_LIMIT"` |
| `entry_trigger` | float | Price trigger (Must be > 0) |
| `sl_distance` | float | Stop Loss distance in price units (> 0) |
| `tp_plan.tp1_distance` | float | TP1 distance in price units (> 0) |
| `tp_plan.tp1_size_pct` | float | Fraction closed at TP1 (0.0-1.0) |
| `tp_plan.tp2_distance` | float \| null | TP2 distance or null (Trail only) |
| `tp_plan.trail_atr_mult` | float | Trailing stop multiplier (N × ATR) |
| `timeout_plan.max_bars` | int | Hard exit after N bars |
| `timeout_plan.mandatory_exit_utc` | string \| null | HH:MM or null |
| `regime_context` | string | `REGIME_STATE` (Must match current) |
| `macro_context` | string | `MACRO_STATE` (Must match current) |
| `execution_constraints.max_spread_bp` | float | Per-strategy max spread (> 0) |
| `execution_constraints.max_slippage_bp` | float | Per-strategy max slippage (> 0) |
| `execution_constraints.min_quote_fresh_ms` | int | Min quote age in ms (> 0) |

---

## 2. RiskDecision Schema

**From:** Risk Engine  
**To:** Portfolio Coordinator → Execution Quality Gate

| Field | Type | Description |
|---|---|---|
| `approved` | bool | `true` if trade passes all risk checks |
| `rejection_reason` | string \| null | Populated if `approved = false` |
| `position_size` | float | Lot size after modifiers |
| `risk_fraction_used` | float | Actual risk fraction used |
| `stop_price` | float | Absolute Stop Loss price |
| `take_profit_plan` | object | Forwarded from SignalIntent |
| `circuit_breaker_state.daily_loss_pct` | float | Current daily P&L % |
| `circuit_breaker_state.weekly_loss_pct` | float | Current weekly P&L % |
| `circuit_breaker_state.consecutive_losses` | int | Sequential loss count |
| `circuit_breaker_state.breaker_active` | bool | `true` if any CB triggered |
| `portfolio_heat_snapshot.current_heat_pct` | float | Sum of open position risks |
| `portfolio_heat_snapshot.heat_remaining` | float | 5.0% - current_heat |
| `portfolio_heat_snapshot.open_positions` | int | Count of open positions |
| `strategy_viability.rolling_er` | float | Expectancy R (30 trades) |
| `strategy_viability.viable` | bool | `false` if rolling ER ≤ 0 |

---

## 3. ExecutionDecision Schema

**From:** Execution Quality Gate  
**To:** Execution Engine

| Field | Type | Description |
|---|---|---|
| `allowed_state` | string | `"ALLOWED"`, `"DEGRADED"`, `"BLOCKED"` |
| `blocked_reason` | string \| null | Reason if BLOCKED |
| `spread_snapshot` | float | Live spread in basis points |
| `spread_vs_median` | float | Live / 100-tick median ratio |
| `quote_freshness_ms` | int | Age of quote in ms |
| `expected_slippage_bp` | float | Estimated slippage |
| `actual_slippage_bp` | float \| null | Post-trade actual slippage |
| `routing_mode` | string | `"LIMIT"`, `"MARKET_FALLBACK"` |
| `order_intent_id` | string | UUID linking to SignalIntent |
| `degraded_action` | string \| null | `"REDUCE_SIZE_50PCT"` if DEGRADED |
