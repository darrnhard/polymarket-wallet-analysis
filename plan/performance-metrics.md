# Performance Metrics — Implementation Plan

## Status: Blocked on market resolution data

## Problem
Data API activity logs only contain REDEEM events when wallets manually
claim winnings. Auto-redeemed or unclaimed markets produce no REDEEM
event, making WIN/LOSS classification from activity data alone unreliable.
Current resolve logic returns ~99% OPEN which is incorrect.

## Solution: Fetch market resolution from Gamma API

### API details
- Endpoint:   GET https://gamma-api.polymarket.com/markets
- Auth:       None required
- Rate limit: 300 requests / 10 seconds
- Key fields: conditionId, closed, resolutionTime, winner (winning outcome)

### Implementation steps
1. Extract all unique conditionIds across all three wallets
   (~55k rn1, ~40k sovereign, ~85k swisstony — many overlap)
2. Deduplicate → estimate ~100k unique markets total
3. Batch fetch at 300/10s with tenacity retry logic
4. Save as data/markets.parquet
5. Join to activity data on conditionId
6. Rebuild resolve logic:
   - market.closed = True + wallet held winning outcome → WIN
   - market.closed = True + wallet held losing outcome  → LOSS
   - market.closed = False                              → OPEN

### Metrics unlocked after this
- Win rate per wallet
- Realized PnL (closed markets only)
- Cumulative PnL curve over time
- Sortino ratio (no annualization — binary markets)
- Calmar ratio
- Average return per market
