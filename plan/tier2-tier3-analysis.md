# Tier 2 & 3 Analysis — Deferred Reference

**Created:** 2026-04-24  
**Status:** Parked — revisit after copy trading bot is live  
**Note:** Performance metrics are tracked separately in PERFORMANCE_METRICS_PLAN.md

---

## Tier 2 — Important, Non-Blocking

### 2a. Kelly Fraction Analysis
Estimate each wallet's implied Kelly fraction from bet sizes relative
to portfolio value. Requires win rate from Gamma API fetch first.

Formula: f = (bp - q) / b
  where b = net odds, p = win probability, q = 1 - p

Useful for copy trading position sizing calibration — tells us
whether wallets are over/under betting relative to their actual edge.

### 2b. Trade Size vs Market Liquidity
Assess whether whale entries are in liquid or illiquid markets.
Large trades in thin markets may be manipulation rather than signal.

**Requires:** CLOB API order book depth
- Endpoint: GET https://clob.polymarket.com/book?token_id=<token_id>
- Auth: None required for read
- Fetch order book depth at time of entry for large trades (>$5k)
- Compute: position_size / available_liquidity ratio
- Flag markets where ratio > 0.1 (wallet absorbs >10% of book)

---

## Tier 3 — Nice to Have

### 3a. Drawdown Analysis
Peak-to-trough on cost basis deployed over time.
No additional data needed — purely from existing activity parquets.

Compute daily cumulative cost basis per wallet, find maximum
drawdown on that series, compare to portfolio value for context.

### 3b. Day-of-Week Sizing Patterns
Beyond trade count heatmap — do they size bigger on specific days?
No additional data needed — purely from existing activity parquets.

Hypothesis: sovereign sizes bigger on NBA game days (Tue/Thu/Sat).
Group trades by day of week, compare median usdcSize per day.

### 3c. Calibration Analysis
Are entry prices (implied probabilities) well-calibrated vs actual outcomes?
Requires Gamma API resolution data — blocked same as performance metrics.

Plot predicted probability (entry price) vs actual win rate per bucket.
A well-calibrated trader buying at 0.60 should win ~60% of the time.
Miscalibration reveals systematic biases in their models.

---

## When to Revisit
- [ ] After copy trading bot is live and generating real trades
- [ ] When tuning position sizing (Kelly fraction becomes relevant)
- [ ] After Gamma API fetch is implemented (unlocks 2a, 3c)
- [ ] After CLOB liquidity data is available (unlocks 2b)
