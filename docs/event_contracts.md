# Event & Regime Contracts

## 1. Regime State Enum

Defines the volatility environment.

| State | Condition | Risk Scalar |
|---|---|---|
| `LOW_VOL` | ratio < 0.85 | 1.00 |
| `MID_VOL` | 0.85 ≤ ratio ≤ 1.25 | 0.80 |
| `HIGH_VOL` | ratio > 1.25 | 0.50 |
| `SHOCK_EVENT` | TR_current > 2.0 × ATR_fast | 0.00 |

*   **Hysteresis:** State changes require 3 consecutive confirming bars (except `SHOCK_EVENT` which is immediate).
*   **Shock Recovery:** 12 bars (60 min) cooldown before transition to `MID_VOL` or `HIGH_VOL`.

---

## 2. Macro State Enum

Defines the macroeconomic bias for Gold (XAU/USD).

| State | Condition | Effect |
|---|---|---|
| `MACRO_BULL_GOLD` | USD falling + Yields falling | Long bias only |
| `MACRO_BEAR_GOLD` | USD rising + Yields rising | Short bias only |
| `MACRO_NEUTRAL` | No strong impulse | Both directions allowed |
| `MACRO_EVENT_RISK` | High impact news or VIX spike | Trading restricted; Risk × 0.5 |

*   **Inputs:** USD Index (DXY) ROC(5), Real Yield (TIPS/TLT) ROC(5), Calendar Severity, VIX.

---

## 3. Execution Quality Enum

Defines the gate status for trade execution.

| State | Condition | Action |
|---|---|---|
| `EXECUTION_ALLOWED` | All checks pass | Proceed normally |
| `EXECUTION_DEGRADED` | Marginal spread/slippage | Reduce size 50% (`REDUCE_SIZE_50PCT`) |
| `EXECUTION_BLOCKED` | High spread, stale quote, or error | Reject trade; Log reason |

---

## 4. Swing Direction Enum

| Value | Meaning |
|---|---|
| `+1` | Bullish |
| `-1` | Bearish |
| `0` | Neutral |

---

## 5. DispatcherPermissions Schema

Output from **Strategy Dispatcher** to **Strategy Engines**.

```json
{
  "allow_swing_rebalance": "bool",
  "allow_orb": "bool",
  "allow_news": "bool",
  "allow_scalp": "bool",
  "direction_constraint": "int (+1 | -1 | 0)",
  "macro_bias": "MACRO_STATE (Enum)",
  "regime": "REGIME_STATE (Enum)",
  "risk_scalar": "float [0.0 - 1.0]",
  "blackout_active": "bool",
  "blackout_reason": "string | null"
}
```

*   **Validation:**
    *   `direction_constraint` must be -1, 0, or 1.
    *   `risk_scalar` must be between 0.0 and 1.0 inclusive.
    *   `macro_bias` and `regime` must be valid Enum values.
