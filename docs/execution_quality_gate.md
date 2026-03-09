# Execution Quality Gate Contract

## 1. Quality State Machine

| State | Condition | Action |
|---|---|---|
| `EXECUTION_BLOCKED` | `spread_ratio > 3.0` | Reject trade; Log reason |
| | `quote_age_ms > 1000` | Reject trade; Log reason |
| | `api_error_rate_5min >= 0.05` | Reject trade; Log reason |
| | `session` outside liquid hours (07:00–20:00 UTC) | Reject trade; Log reason |
| `EXECUTION_DEGRADED` | `NOT BLOCKED` AND (`spread_ratio > 1.80` OR `expected_slippage_bp > signal.max_slippage_bp`) | Reduce size 50% (`REDUCE_SIZE_50PCT`) |
| `EXECUTION_ALLOWED` | All other conditions | Proceed normally |

---

## 2. Per-Strategy Constraints

| Strategy | max_spread_bp | max_slippage_bp | min_quote_fresh_ms | Session |
|---|---|---|---|---|
| **Swing** | 10.0 | 15.0 | 2000 | None |
| **ORB** | 3.0 | 5.0 | 500 | 07:00–20:00 UTC |
| **News T1/T2** | 3.0 / 2.5 | 5.0 | 500 | 07:00–21:00 UTC |
| **Scalping** | 2.0 | 3.0 | 300 | 08:00–17:00 UTC |

---

## 3. Numeric Thresholds (Frozen)

*   **Spread Ratio Calculation:** `Live_Spread / Rolling_Median_100_Tick`
*   **Quote Freshness Calculation:** `Current_Time_UTC - Quote_Time_UTC` (in ms)
*   **API Error Rate Calculation:** `Errors / Requests` over last 5 minutes
*   **Liquid Hours:** 07:00 UTC to 20:00 UTC (London/NY overlap & US session)
*   **Slippage Estimate:** `Spread/2 + (Order_Size / ADV) * Impact_Factor`
