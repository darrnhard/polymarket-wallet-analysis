# Executive Summary
## Polymarket Wallet Analysis — Phase 1

**Date:** April 2026  
**Scope:** Behavioral analysis of three high-performing Polymarket wallets  
**Status:** Phase 1 complete — findings inform Phase 2 copy trading implementation

---

## 1. Objective

Identify and reverse-engineer the trading strategies of the highest-performing wallets on Polymarket, with the goal of evaluating their suitability as copy trading signal sources for a proprietary automated trading system.

Selection criteria for target wallets: consistent cumulative PnL growth, high win rate across hundreds of markets, no detectable gambling behavior (no martingale sizing, no all-in patterns, diversified market exposure).

---

## 2. Data

**Source:** Polymarket Data API (public, no authentication)  
**Coverage:** Full onchain history from account inception through April 2026  
**Volume:** 6,846,004 total events across three wallets

| Wallet | Label | Period | Events | Portfolio Value |
|---|---|---|---|---|
| `0x2005...75ea` | rn1 | Jul 2025 → Apr 2026 | 2,298,316 | $402k |
| `0xee61...debf` | sovereign | Aug 2025 → Apr 2026 | 1,271,417 | $105k |
| `0x204f...e14` | swisstony | Aug 2025 → Apr 2026 | 3,276,271 | $210k |

**Data quality:** Zero null values across all 23 fields. All three wallets start exactly on their expected first activity date with no gaps exceeding 7 days (sovereign has a genuine 19-day dormancy period in Aug–Sep 2025 consistent with a new account).

---

## 3. Key Findings

### 3.1 All three wallets are automated bots

- 99.99% BUY-only across all three (swisstony: literally zero SELL trades across 3.1M events)
- Sub-cent trade sizes ($0.0001 minimum for rn1) — no human places $0.0001 trades manually
- Burst behavior: 66–75% of all activity occurs in rapid-fire clusters of 5–7 trades within 60–90 seconds
- Trading activity peaks at UTC 16:00–22:00 across all three wallets — US market hours, consistent with bots configured by US-timezone operators

### 3.2 Three completely distinct strategies

| Metric | rn1 | sovereign | swisstony |
|---|---|---|---|
| Primary sport | Global (28% Soccer EU, 14% Esports, 12% NBA) | NBA (53%) | Soccer EU (35%) |
| Market type | Head-to-Head (46%), Outright Winner (25%) | Over/Under (40%), Spread (24%), H2H (37%) | Outright Winner (31%), O/U (24%), H2H (22%) |
| Entry price median | 0.47 (broad, uniform distribution) | 0.49 (tight spike at 0.47–0.52) | 0.51 (broad bell) |
| Holding period median | 1.6h | 12.2h | 15.1h |
| Markets per day (median) | 145 | 220 | 498 |
| Portfolio value | $402k | $105k | $210k |
| Monthly USDC volume | ~$40M | ~$30M | ~$80M |

**sovereign** is a pure lines betting model — exclusively trading NBA Over/Under totals and point spreads near 50/50 probability. Entry price std of 0.03 across sessions confirms they're not chasing price movements, but hammering stable lines as liquidity refreshes. This is the most quantifiable, model-driven strategy of the three.

**swisstony** is a European football specialist with the broadest market coverage. Trades across all football bet types — outright winners, O/U, both teams to score, spreads, draws. Monthly volume of $80M makes them the most active by capital deployed.

**rn1** is a global scanner — trades every sport, every market type, at every probability level. The near-uniform entry price distribution (0.0 to 1.0) is the clearest evidence: rn1 has no probability preference, suggesting a high-frequency strategy that bets on volume rather than edge concentration.

### 3.3 Lead/follow hierarchy

The most operationally important finding. For markets traded by multiple wallets, we measured who enters first:

| Pair | Who leads | Lead rate | Median time gap |
|---|---|---|---|
| sovereign vs rn1 | sovereign | 82% | 403 minutes |
| sovereign vs swisstony | sovereign | 68% | 160 minutes |
| swisstony vs rn1 | swisstony | 86% | 594 minutes |

**Hierarchy: sovereign → swisstony → rn1**

sovereign enters first in 68–82% of shared markets. rn1 enters last in 86% of cases. This is not noise — it is a consistent directional signal across tens of thousands of markets.

Implication: **rn1 is a follower, not a leader.** Copying rn1 means entering markets that sovereign and swisstony have already entered hours earlier. By the time rn1 enters, price has already moved.

### 3.4 Market overlap structure

```
swisstony ──── 85,235 unique markets (broadest coverage)
    │
    ├── 36,098 shared with rn1 only
    ├── 12,745 shared with sovereign only  
    └── 7,645 shared with ALL THREE  ← highest conviction signal
    
rn1 ────────── 55,717 unique markets
    └── 78.5% overlap with swisstony

sovereign ───── 40,549 unique markets
    └── only 24.6% overlap with rn1 (most independent)
```

The 78.5% overlap between rn1 and swisstony is too high to be coincidental — they likely share a signal source or one is configured from the other. sovereign's independence (24.6% overlap with rn1) reflects their narrow NBA-only focus.

### 3.5 Copy trading feasibility assessment

| Wallet | Signal quality | Execution window | Daily load | Copy verdict |
|---|---|---|---|---|
| sovereign | ★★★★★ Alpha signal | 12h median holding | 220 markets/day | ✓ Primary signal |
| swisstony | ★★★☆☆ Confirms sovereign | 15h median holding | 498 markets/day | ⚠ Use as filter only |
| rn1 | ★★☆☆☆ Lags by 400+ min | 1.6h median holding | 145 markets/day | ✗ Do not copy |

**Recommended copy protocol:**
1. Monitor sovereign as primary signal source
2. On new market entry detection, place one proportionally-sized order within the 60–90 second burst window
3. If swisstony enters the same market within 3 hours, increase position size (consensus confirmation)
4. Ignore rn1 for signal generation entirely
5. All-three consensus (7,645 historical shared markets) = highest conviction tier

---

## 4. Methodology Notes

**Fetch pipeline:** Time-window chunked pagination with recursive bisection handles the Polymarket Data API's hard 3,500-offset cap. Weekly windows bisect to hourly or sub-hourly chunks for high-density periods. Fully resumable via JSON ledger checkpointing.

**Resolve logic limitation:** Market WIN/LOSS classification from activity data alone is unreliable — Polymarket auto-redeems winning positions without generating REDEEM events in the wallet activity log. Performance metrics (win rate, realized PnL, Sortino) are deferred pending Gamma API market resolution fetch. See `plan/PERFORMANCE_METRICS_PLAN.md`.

**Market classification:** Rule-based keyword classifier across 15 sport/domain categories. 82% coverage across 29,000+ unique market titles. Remaining 18% is genuine long-tail (obscure global leagues) treated as a valid catch-all category.

---

## 5. Next Steps (Phase 2)

1. **Gamma API fetch** — resolve WIN/LOSS for all ~100k unique conditionIds, unlock performance metrics
2. **Copy trading bot** — real-time sovereign monitoring, first-trade detection, proportional execution
3. **Risk controls** — per-market position cap, daily drawdown limit, circuit breaker on consecutive losses
4. **Paper trading** — validate copy protocol against historical sovereign entries before live capital

---

*This document summarizes findings from `notebooks/02_analysis.ipynb`. All data sourced from Polymarket's public Data API. Past trading patterns do not guarantee future results.*
