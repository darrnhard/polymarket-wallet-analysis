# Analysis

Analysis notebook structure
Section 1 — Data quality & basic shape
Null counts, dtypes, value distributions. Make sure the data is trustworthy before drawing any conclusions.
Section 2 — Event type deep dive
Distribution of TRADE/REDEEM/MERGE/REWARD/MAKER_REBATE per wallet. Understanding the activity composition before doing any PnL work.
Section 3 — Resolve logic
Per-market PnL reconstruction. Classify every market each wallet touched as WIN / LOSS / OPEN. This is the foundation for all performance metrics.
Section 4 — Performance metrics
Win rate, total PnL, realized vs unrealized, Sortino, Calmar, cumulative PnL curve over time.
Section 5 — Behavioral analysis
Trading frequency, time-of-day patterns, position sizing, holding periods, market category preferences.
Section 6 — Strategy reverse engineering
What kinds of markets do they bet on, at what implied probabilities, how do they size, do they average in or out, any detectable signals in their entry timing.
Section 7 — Cross-wallet comparison
Do they trade the same markets? Do they copy each other? Who leads, who follows?

## Import


```python
import pandas as pd
import numpy as np
import json
from datetime import datetime, timezone
from collections import Counter
import matplotlib.pyplot as plt
import matplotlib.ticker as mticker
import seaborn as sns
import plotly.express as px
import plotly.graph_objects as go
from plotly.subplots import make_subplots

# Disable Arrow-backed strings — same fix as fetch notebook
pd.options.future.infer_string = False

# Seaborn style for all static plots
sns.set_theme(style="darkgrid", palette="muted")

# Display settings — show all columns, readable floats
pd.set_option("display.max_columns", None)
pd.set_option("display.max_colwidth", 60)
pd.set_option("display.float_format", "{:.4f}".format)
```


```python
WALLETS   = ["rn1", "sovereign", "swisstony"]
DATA_DIR  = "data"


# Color map — consistent wallet colors across all plots
WALLET_COLORS = {
    "rn1":       "#4C72B0",  # blue
    "sovereign": "#DD8452",  # orange
    "swisstony": "#55A868",  # green
}
```


```python
dfs = {}

for name in WALLETS:
    path = f"{DATA_DIR}/activity_{name}.parquet"
    df   = pd.read_parquet(path, engine="fastparquet")
    df["wallet"]   = name                                          # add wallet label
    df["datetime"] = pd.to_datetime(df["timestamp"], unit="s", utc=True)  # human readable time
    dfs[name] = df
    print(f"{name}: {len(df):,} rows | {df['datetime'].min().date()} → {df['datetime'].max().date()}")

# Combined dataframe for cross-wallet analysis
activity = pd.concat(dfs.values(), ignore_index=True)
print(f"\nCombined: {len(activity):,} total rows")
print(f"Columns: {list(activity.columns)}")
```

    rn1: 2,298,316 rows | 2025-07-09 → 2026-04-22
    sovereign: 1,271,417 rows | 2025-08-07 → 2026-04-22
    swisstony: 3,276,271 rows | 2025-08-09 → 2026-04-22
    
    Combined: 6,846,004 total rows
    Columns: ['proxyWallet', 'timestamp', 'conditionId', 'type', 'size', 'usdcSize', 'transactionHash', 'price', 'asset', 'side', 'outcomeIndex', 'title', 'slug', 'icon', 'eventSlug', 'outcome', 'name', 'pseudonym', 'bio', 'profileImage', 'profileImageOptimized', 'wallet', 'datetime']


## Data Quality and Basic Shape

### Null counts and dtypes


```python
print("=" * 50)
print("DTYPES")
print("=" * 50)
print(activity.dtypes)

print("\n" + "=" * 50)
print("NULL COUNTS (combined)")
print("=" * 50)
nulls = activity.isnull().sum()
nulls_pct = (nulls / len(activity) * 100).round(2)
null_summary = pd.DataFrame({"null_count": nulls, "null_pct": nulls_pct})
print(null_summary[null_summary["null_count"] > 0])
```

    ==================================================
    DTYPES
    ==================================================
    proxyWallet                          object
    timestamp                             int64
    conditionId                          object
    type                                 object
    size                                float64
    usdcSize                            float64
    transactionHash                      object
    price                               float64
    asset                                object
    side                                 object
    outcomeIndex                          int64
    title                                object
    slug                                 object
    icon                                 object
    eventSlug                            object
    outcome                              object
    name                                 object
    pseudonym                            object
    bio                                  object
    profileImage                         object
    profileImageOptimized                object
    wallet                               object
    datetime                 datetime64[s, UTC]
    dtype: object
    
    ==================================================
    NULL COUNTS (combined)
    ==================================================
    Empty DataFrame
    Columns: [null_count, null_pct]
    Index: []


## Value distributions on key fields


```python
print("=" * 50)
print("EVENT TYPE DISTRIBUTION PER WALLET")
print("=" * 50)
for name in WALLETS:
    dist = dfs[name]["type"].value_counts()
    total = len(dfs[name])
    print(f"\n{name} ({total:,} total):")
    for event_type, count in dist.items():
        pct = count / total * 100
        print(f"  {event_type:<20} {count:>10,}  ({pct:.2f}%)")

print("\n" + "=" * 50)
print("SIDE DISTRIBUTION (BUY vs SELL) — TRADES ONLY")
print("=" * 50)
for name in WALLETS:
    trades = dfs[name][dfs[name]["type"] == "TRADE"]
    dist   = trades["side"].value_counts()
    total  = len(trades)
    print(f"\n{name} ({total:,} trades):")
    for side, count in dist.items():
        pct = count / total * 100
        print(f"  {side:<10} {count:>10,}  ({pct:.2f}%)")

print("\n" + "=" * 50)
print("USDC SIZE DISTRIBUTION (TRADES ONLY)")
print("=" * 50)
for name in WALLETS:
    trades = dfs[name][dfs[name]["type"] == "TRADE"]
    stats  = trades["usdcSize"].describe(percentiles=[.25, .5, .75, .90, .95, .99])
    print(f"\n{name}:")
    print(stats.to_string())
```

    ==================================================
    EVENT TYPE DISTRIBUTION PER WALLET
    ==================================================
    
    rn1 (2,298,316 total):
      TRADE                 2,287,110  (99.51%)
      REDEEM                    6,623  (0.29%)
      MERGE                     4,457  (0.19%)
      REWARD                       83  (0.00%)
      MAKER_REBATE                 42  (0.00%)
      CONVERSION                    1  (0.00%)
    
    sovereign (1,271,417 total):
      TRADE                 1,194,961  (93.99%)
      REDEEM                   40,207  (3.16%)
      MERGE                    36,126  (2.84%)
      REWARD                       74  (0.01%)
      MAKER_REBATE                 49  (0.00%)
    
    swisstony (3,276,271 total):
      TRADE                 3,132,655  (95.62%)
      REDEEM                  141,001  (4.30%)
      MERGE                     2,481  (0.08%)
      REWARD                       84  (0.00%)
      MAKER_REBATE                 50  (0.00%)
    
    ==================================================
    SIDE DISTRIBUTION (BUY vs SELL) — TRADES ONLY
    ==================================================
    
    rn1 (2,287,110 trades):
      BUY         2,286,867  (99.99%)
      SELL              243  (0.01%)
    
    sovereign (1,194,961 trades):
      BUY         1,194,949  (100.00%)
      SELL               12  (0.00%)
    
    swisstony (3,132,655 trades):
      BUY         3,132,655  (100.00%)
    
    ==================================================
    USDC SIZE DISTRIBUTION (TRADES ONLY)
    ==================================================
    
    rn1:
    count   2287110.0000
    mean         79.0953
    std         367.7390
    min           0.0000
    25%           1.5000
    50%           6.8000
    75%          42.0000
    90%         186.0000
    95%         380.0000
    99%         970.8593
    max       74549.7169
    
    sovereign:
    count   1194961.0000
    mean        141.8711
    std         992.1106
    min           0.0003
    25%           1.6500
    50%           5.6667
    75%          45.6167
    90%         295.0010
    95%         768.6274
    99%        2619.9996
    max      711838.9416
    
    swisstony:
    count   3132655.0000
    mean        109.4579
    std         582.0882
    min           0.0001
    25%           3.0500
    50%          11.8553
    75%          54.4465
    90%         215.5600
    95%         451.4364
    99%        1589.5968
    max      175590.7200



```python
fig, axes = plt.subplots(1, 3, figsize=(16, 5))
fig.suptitle("Trade Size Distribution (USDC) — Log Scale", fontsize=14, fontweight="bold")

for ax, name in zip(axes, WALLETS):
    trades = dfs[name][dfs[name]["type"] == "TRADE"]["usdcSize"]
    color  = WALLET_COLORS[name]

    # Log scale because distribution is heavily right-skewed
    ax.hist(
        trades[trades > 0],   # exclude zero-size trades
        bins=100,
        color=color,
        alpha=0.85,
        log=True,             # y-axis log scale
    )
    ax.set_xscale("log")      # x-axis log scale too

    # Mark median and 95th percentile
    median = trades.median()
    p95    = trades.quantile(0.95)
    ax.axvline(median, color="white",  linestyle="--", linewidth=1.5, label=f"Median ${median:.1f}")
    ax.axvline(p95,    color="yellow", linestyle="--", linewidth=1.5, label=f"P95 ${p95:.1f}")

    ax.set_title(name, fontweight="bold", color=color)
    ax.set_xlabel("Trade Size (USDC, log scale)")
    ax.set_ylabel("Count (log scale)")
    ax.legend(fontsize=8)

plt.tight_layout()
plt.savefig("plots/trade_size_distribution.png", dpi=150, bbox_inches="tight")
plt.show()
```


    
![png](02_analysis_files/02_analysis_11_0.png)
    



```python
fig, axes = plt.subplots(1, 3, figsize=(16, 5))
fig.suptitle("Trade Size Distribution (USDC)", fontsize=14, fontweight="bold")

for ax, name in zip(axes, WALLETS):
    trades = dfs[name][dfs[name]["type"] == "TRADE"]["usdcSize"]
    color  = WALLET_COLORS[name]

    # Cap at 99th percentile so the long tail doesn't flatten the chart
    p75    = trades.quantile(0.75)
    trades_capped = trades[trades <= p75]

    ax.hist(
        trades_capped,
        bins=80,
        color=color,
        alpha=0.85,
        edgecolor="none",
    )

    # Mark median and mean
    median = trades.median()
    mean   = trades.mean()
    ax.axvline(median, color="white",  linestyle="--", linewidth=1.5, label=f"Median ${median:.1f}")
    ax.axvline(mean,   color="yellow", linestyle="--", linewidth=1.5, label=f"Mean ${mean:.1f}")

    ax.set_title(name, fontweight="bold", color=color)
    ax.set_xlabel("Trade Size (USDC)")
    ax.set_ylabel("Count")
    ax.legend(fontsize=8)
    ax.xaxis.set_major_formatter(mticker.FuncFormatter(lambda x, _: f"${x:,.0f}"))

plt.tight_layout()
plt.savefig("plots/trade_size_distribution.png", dpi=150, bbox_inches="tight")
plt.show()
```


    
![png](02_analysis_files/02_analysis_12_0.png)
    


## Monthly trading volume


```python
trades_only = activity[activity["type"] == "TRADE"].copy()
trades_only["month"] = trades_only["datetime"].dt.to_period("M").dt.to_timestamp()

monthly = (
    trades_only
    .groupby(["wallet", "month"])
    .agg(
        trade_count=("transactionHash", "count"),
        total_usdc =("usdcSize", "sum"),
    )
    .reset_index()
)

# Plot 1 — trade count over time
fig = px.line(
    monthly,
    x="month",
    y="trade_count",
    color="wallet",
    color_discrete_map=WALLET_COLORS,
    markers=True,
    title="Monthly Trade Count per Wallet",
    labels={"month": "Month", "trade_count": "Number of Trades", "wallet": "Wallet"},
)
fig.update_layout(hovermode="x unified")
fig.show()

# Plot 2 — total USDC volume over time
fig2 = px.line(
    monthly,
    x="month",
    y="total_usdc",
    color="wallet",
    color_discrete_map=WALLET_COLORS,
    markers=True,
    title="Monthly USDC Volume per Wallet",
    labels={"month": "Month", "total_usdc": "Total USDC", "wallet": "Wallet"},
)
fig2.update_layout(hovermode="x unified", yaxis_tickprefix="$")
fig2.show()
```

    /tmp/ipykernel_485214/1786070467.py:2: UserWarning: Converting to PeriodArray/Index representation will drop timezone information.
      trades_only["month"] = trades_only["datetime"].dt.to_period("M").dt.to_timestamp()






## Hourly activity heatmap


```python
trades_only = activity[activity["type"] == "TRADE"].copy()
trades_only["hour"]    = trades_only["datetime"].dt.hour
trades_only["weekday"] = trades_only["datetime"].dt.day_name()

WEEKDAY_ORDER = ["Monday","Tuesday","Wednesday","Thursday","Friday","Saturday","Sunday"]

fig, axes = plt.subplots(1, 3, figsize=(18, 5))
fig.suptitle("Trading Activity Heatmap (Hour of Day vs Day of Week, UTC)", 
             fontsize=14, fontweight="bold")

for ax, name in zip(axes, WALLETS):
    pivot = (
        trades_only[trades_only["wallet"] == name]
        .groupby(["weekday", "hour"])
        .size()
        .unstack(fill_value=0)
        .reindex(WEEKDAY_ORDER)
    )

    sns.heatmap(
        pivot,
        ax=ax,
        cmap="YlOrRd",
        cbar=True,
        linewidths=0,
        xticklabels=4,  # show every 4th hour label
    )
    ax.set_title(name, fontweight="bold", color=WALLET_COLORS[name])
    ax.set_xlabel("Hour (UTC)")
    ax.set_ylabel("Day of Week")

plt.tight_layout()
plt.savefig("plots/hourly_heatmap.png", dpi=150, bbox_inches="tight")
plt.show()
```


    
![png](02_analysis_files/02_analysis_16_0.png)
    


## PNL calculation and logic

### Resolve logic


```python
def resolve_markets(df):
    """
    Reconstruct per-market PnL and resolution status for a single wallet.

    Key design decisions:
    - OPEN/CLOSED classification uses asset-level net_tokens, not conditionId-level.
      This handles wallets that buy both sides of a market — losing tokens expire
      worthless and should not keep a market classified as OPEN.
    - realized_pnl is computed at conditionId level (full market economics).
    - REWARD and MAKER_REBATE excluded — platform incentives, not trading outcomes.
    - realized_pnl is only meaningful for WIN/LOSS markets, not OPEN.
    """
    trading = df[~df["type"].isin(["REWARD", "MAKER_REBATE"])].copy()

    # --- Asset-level net tokens (per outcome, not per market) ---
    # Used only for OPEN/CLOSED detection
    asset_tokens_in = (
        trading[
            ((trading["type"] == "TRADE") & (trading["side"] == "BUY")) |
            (trading["type"] == "SPLIT")
        ]
        .groupby(["conditionId", "asset"])["size"].sum()
    )

    asset_tokens_out = (
        trading[
            ((trading["type"] == "TRADE") & (trading["side"] == "SELL")) |
            (trading["type"].isin(["REDEEM", "MERGE"]))
        ]
        .groupby(["conditionId", "asset"])["size"].sum()
    )

    asset_net = (asset_tokens_in - asset_tokens_out).fillna(asset_tokens_in).fillna(0)

    # A market is OPEN if ANY of its assets still has tokens > 0.01
    open_conditions = set(
        asset_net[asset_net > 0.01]
        .reset_index()["conditionId"]
        .unique()
    )

    # --- conditionId-level cash flows (for PnL) ---
    cash_out = (
        trading[
            ((trading["type"] == "TRADE") & (trading["side"] == "BUY")) |
            (trading["type"] == "SPLIT")
        ]
        .groupby("conditionId")["usdcSize"].sum()
        .rename("cash_out")
    )

    cash_in = (
        trading[
            ((trading["type"] == "TRADE") & (trading["side"] == "SELL")) |
            (trading["type"].isin(["REDEEM", "MERGE"]))
        ]
        .groupby("conditionId")["usdcSize"].sum()
        .rename("cash_in")
    )

    has_redeem = (
        trading[trading["type"] == "REDEEM"]
        .groupby("conditionId")["type"].count()
        .gt(0)
        .rename("has_redeem")
    )

    # --- Market metadata ---
    meta = (
        trading.sort_values("timestamp")
        .groupby("conditionId")
        .agg(
            title      =("title",           "last"),
            outcome    =("outcome",         "last"),
            first_trade=("datetime",        "first"),
            last_trade =("datetime",        "last"),
            wallet     =("wallet",          "first"),
            trade_count=("transactionHash", "count"),
        )
    )

    resolved = (
        meta
        .join(cash_out,   how="left")
        .join(cash_in,    how="left")
        .join(has_redeem, how="left")
        .fillna(0)
    )

    resolved["has_redeem"]   = resolved["has_redeem"].astype(bool)
    resolved["realized_pnl"] = resolved["cash_in"] - resolved["cash_out"]

    # Classify using asset-level open detection
    def classify(row):
        if row.name in open_conditions:
            return "OPEN"
        elif row["has_redeem"]:
            return "WIN"
        else:
            return "LOSS"

    resolved["status"] = resolved.apply(classify, axis=1)

    return resolved.reset_index()
```


```python
# Run for all wallets
markets = {}

for name in WALLETS:
    result        = resolve_markets(dfs[name])
    markets[name] = result
    status_counts = result["status"].value_counts()
    total         = len(result)

    closed = result[result["status"].isin(["WIN", "LOSS"])]

    print(f"\n{name} — {total:,} unique markets:")
    for status, count in status_counts.items():
        pct = count / total * 100
        print(f"  {status:<6} {count:>6,}  ({pct:.1f}%)")
    print(f"  Realized PnL (closed only): ${closed['realized_pnl'].sum():,.2f}")
    print(f"  Cost basis  (open only):    ${result[result['status']=='OPEN']['cash_out'].sum():,.2f}")

markets_all = pd.concat(markets.values(), ignore_index=True)
```

    
    rn1 — 55,716 unique markets:
      OPEN   55,690  (100.0%)
      LOSS       22  (0.0%)
      WIN         4  (0.0%)
      Realized PnL (closed only): $109.41
      Cost basis  (open only):    $179,848,374.14
    
    sovereign — 40,549 unique markets:
      OPEN   40,542  (100.0%)
      LOSS        6  (0.0%)
      WIN         1  (0.0%)
      Realized PnL (closed only): $298.99
      Cost basis  (open only):    $169,516,778.78
    
    swisstony — 85,235 unique markets:
      OPEN   85,231  (100.0%)
      WIN         4  (0.0%)
      Realized PnL (closed only): $158.00
      Cost basis  (open only):    $342,893,721.99


## Behavioral Analysis

### Market category analysis


```python
# Extract category from slug — the first segment before the first hyphen-separated
# keyword gives us a rough category signal. But title keywords are more reliable.

# Let's look at the raw slugs and titles first to understand the structure
print("=== Sample slugs per wallet ===")
for name in WALLETS:
    trades = dfs[name][dfs[name]["type"] == "TRADE"].copy()
    print(f"\n{name} — 20 most traded markets by trade count:")
    top = (
        trades.groupby(["conditionId", "title", "slug"])
        .size()
        .reset_index(name="trade_count")
        .sort_values("trade_count", ascending=False)
        .head(20)
    )
    for _, row in top.iterrows():
        print(f"  {row['trade_count']:>6,}  {row['title'][:60]}")
```

    === Sample slugs per wallet ===
    
    rn1 — 20 most traded markets by trade count:
       2,323  CA Mineiro vs. São Paulo FC: O/U 3.5
       2,172  Will Fluminense FC win on 2026-03-18?
       2,075  Will Kocaelispor vs. Konyaspor end in a draw?
       1,740  Credit One Charleston Open: Jessica Pegula vs Diana Shnaider
       1,730  SE Palmeiras vs. Grêmio FBPA: O/U 2.5
       1,617  Will Wuhan San Zhen FC win on 2026-03-13?
       1,614  Will Gaziantep FK vs. Kayserispor end in a draw?
       1,573  Dundee United FC vs. Aberdeen FC: O/U 1.5
       1,509  Will CS Independiente Rivadavia win on 2026-04-02?
       1,499  Kilmarnock FC vs. St Mirren FC: O/U 1.5
       1,485  Will Real Madrid CF win on 2026-03-22?
       1,460  Dubrovnik: Polona Hercog vs Tamara Korpatsch
       1,450  Will Paris Saint-Germain FC win on 2026-03-17?
       1,449  Trail Blazers vs. Nuggets
       1,358  Will Club Atlético de Madrid win on 2026-03-18?
       1,356  Santa Clara Broncos vs. Kentucky Wildcats
       1,355  Colorado Rockies vs. Houston Astros
       1,351  Will Real Madrid CF win on 2026-03-17?
       1,339  Australian Open Men's: Carlos Alcaraz vs Alexander Zverev
       1,332  Colorado Rockies vs. San Diego Padres
    
    sovereign — 20 most traded markets by trade count:
       3,518  Kings vs. Celtics: O/U 217.5
       3,378  Spread: Clippers (-1.5)
       2,850  Mavericks vs. Rockets: O/U 221.5
       2,539  Warriors vs. Suns: O/U 215.5
       2,444  Clippers vs. Nuggets: O/U 217.5
       2,107  Nets vs. Magic: O/U 214.5
       1,867  Pelicans vs. 76ers: O/U 232.5
       1,807  Clippers vs. Nuggets: O/U 219.5
       1,684  Clippers vs. Kings: O/U 222.5
       1,568  Spread: Timberwolves (-8.5)
       1,504  Spread: Trail Blazers (-6.5)
       1,361  Pistons vs. Suns: O/U 215.5
       1,313  Heat vs. Bulls: O/U 243.5
       1,226  Spurs vs. Hornets: O/U 226.5
       1,199  Magic vs. Thunder: O/U 218.5
       1,194  Hawks vs. Pacers: O/U 232.5
       1,158  Pelicans vs. 76ers: O/U 231.5
       1,152  Spread: Rockets (-4.5)
       1,133  The Citadel Bulldogs vs. Samford Bulldogs: O/U 143.5
       1,127  Celtics vs. Mavericks: O/U 221.5
    
    swisstony — 20 most traded markets by trade count:
       2,708  Manchester City vs. Liverpool: Both Teams to Score
       2,224  Rayo Vallecano vs. Real Madrid: O/U 2.5
       1,823  Pharco FC vs. ZED FC: O/U 2.5
       1,736  Will FC Barcelona win on 2026-02-12?
       1,690  Sport Lisboa e Benfica vs. Real Madrid CF: Both Teams to Sco
       1,388  Spread: UD Las Palmas (-2.5)
       1,368  Sunderland AFC vs. Arsenal: Both Teams to Score
       1,293  Will Aston Villa FC win on 2025-12-30?
       1,237  Will 1. FC Magdeburg win on 2026-01-30?
       1,219  Spread: Manchester City (-0.5)
       1,167  Spread: Real Sociedad (-0.5)
       1,097  ATP World Tour Finals, Jimmy Connors Group: Carlos Alcaraz v
       1,089  Will Real Madrid CF win on 2026-03-22?
       1,084  Will Real Madrid CF win on 2026-03-02?
       1,069  Will Morocco win on 2025-12-26?
       1,036  Will Egypt win on 2025-12-29?
       1,013  Will Manchester City FC win on 2025-12-27?
       1,003  Spread: Chelsea FC (-1.5)
       1,002  RCD Espanyol de Barcelona vs. Deportivo Alavés: Both Teams t
         949  Will NEC win on 2026-04-19?


### Category classification


```python
def classify_market(title, slug):
    """
    Classify a market into sport category and market type.
    Uses simple string matching — fast enough for exploration,
    intentionally non-vectorized for readability and debuggability.
    'Other' is a valid catch-all for long-tail global markets.
    """
    title_lower = title.lower()
    slug_lower  = slug.lower()
    combined    = title_lower + " " + slug_lower

    # --- Market type ---
    if "o/u" in title_lower or "over/under" in title_lower:
        market_type = "Over/Under"
    elif "games total" in title_lower:
        market_type = "Series Total"
    elif "spread:" in title_lower:
        market_type = "Spread"
    elif "both teams to score" in title_lower:
        market_type = "Both Teams Score"
    elif "will " in title_lower and " win" in title_lower:
        market_type = "Outright Winner"
    elif "end in a draw" in title_lower:
        market_type = "Draw"
    elif "vs." in title_lower or " vs " in title_lower:
        market_type = "Head to Head"
    else:
        market_type = "Other"

    # --- Sport ---
    esports = ["counter-strike","cs2","dota 2","dota2","league of legends",
               "lol:","valorant","overwatch","rainbow six","natus vincere",
               "alliance","aurora gaming","bilibili gaming","top esports",
               "forze","9ine","phantom","zero tenacity"]

    nba = ["celtics","lakers","warriors","bulls","heat","knicks","nets","bucks",
           "suns","nuggets","clippers","rockets","mavericks","timberwolves",
           "spurs","pelicans","hawks","pacers","pistons","magic","thunder",
           "trail blazers","kings","hornets","grizzlies","jazz","76ers",
           "raptors","wizards","cavaliers"]

    nhl = ["blues","sharks","bruins","canadiens","maple leafs","penguins",
           "blackhawks","red wings","flyers","capitals","senators","jets",
           "flames","oilers","canucks","ducks","avalanche","wild","predators",
           "hurricanes","lightning","islanders","devils","sabres","kraken",
           "golden knights","coyotes","stars"]

    nfl = ["chiefs","patriots","cowboys","packers","49ers","ravens","bears",
           "broncos","eagles","giants","jets","rams","saints","seahawks",
           "steelers","texans","titans","vikings","chargers","falcons","lions",
           "colts","browns","bengals","raiders","panthers","buccaneers","jaguars"]

    mlb = ["yankees","dodgers","red sox","cubs","astros","braves","cardinals",
           "mets","rockies","padres","phillies","mariners","athletics","tigers",
           "twins","white sox","orioles","rays","blue jays","reds","brewers",
           "pirates","marlins","nationals","angels","guardians","royals"]

    tennis = ["atp","wta","wimbledon","australian open","us open","french open",
              "roland","grand slam","alcaraz","djokovic","sinner","swiatek",
              "pegula","sabalenka","medvedev","zverev","rublev","tsitsipas",
              "open men","open women","dubrovnik","charleston"]

    liga_mx = ["tigres","cruz azul","toluca","chivas","guadalajara","atlas",
               "monterrey","pumas","leon fc","necaxa","pachuca","santos laguna",
               "queretaro","mazatlan","juarez","puebla","tijuana","atletico san luis"]

    soccer_brazil = ["ca mineiro","são paulo","sao paulo","fluminense","flamengo",
                     "corinthians","palmeiras","santos fc","grêmio","gremio",
                     "ec bahia","ec vitoria","vasco","cruzeiro","fortaleza",
                     "ceara","sport recife","coritiba","clube do remo","juventude",
                     "chapecoense","ca paranaense","atletico paranaense",
                     "ca gimnasia","independiente rivadavia","cs cristal",
                     "ca talleres","ca boca","boca juniors","river plate"]

    soccer_asia = ["wuhan","dalian","henan","beijing","shanghai","guangzhou",
                   "jiangsu","shandong","chongqing","tianjin","shenzhen",
                   "al nassr","al qadisiyah","al hilal","al ittihad","al ahli",
                   "al hazem","pharco","zed fc","zamalek","al ahly",
                   "gwangju","incheon","angola","kenya","ethiopia","mali",
                   "burkina","zimbabwe","namibia","zambia","tanzania","uganda"]

    soccer_eu = ["premier league","la liga","bundesliga","serie a","ligue 1",
                 "champions league","europa league","eredivisie","primeira liga",
                 "manchester","liverpool","chelsea","arsenal","aston villa",
                 "sunderland","tottenham","southampton","middlesbrough",
                 "west ham","newcastle","fulham","crystal palace","brentford",
                 "brighton","bournemouth","nottingham","leicester","everton",
                 "leeds united","oxford united","blackburn","real madrid",
                 "barcelona","atletico de madrid","atletico madrid","club atlético",
                 "rayo","espanyol","sociedad","las palmas","real betis","villarreal",
                 "getafe","mallorca","sevilla","stuttgart","borussia","schalke",
                 "hertha","werder","freiburg","hoffenheim","wolfsburg","bayer",
                 "st. pauli","hamburger","greuther","paderborn","magdeburg",
                 "elversberg","rb leipzig","bologna","juventus","milan","inter",
                 "lazio","roma","napoli","atalanta","udinese","sampdoria",
                 "sassuolo","genoa","lecce","pisa","como 1907","ac monza",
                 "ssc bari","reggiana","catanzaro","modena","psg",
                 "paris saint-germain","olympique","monaco","bastia","montpellier",
                 "lens","benfica","porto","sporting cp","braga","tondela",
                 "santa clara","galatasaray","fenerbah","besiktas","trabzonspor",
                 "konyaspor","kayserispor","gaziantep","kocaelispor","rizespor",
                 "antalyaspor","karagumrük","karagumruk","ajax","psv","nec",
                 "dinamo zagreb","celta","salzburg","feyenoord","anderlecht",
                 "celtic","rangers","dundee","kilmarnock","st mirren","aberdeen",
                 "fredrikstad","kfum","bodø","bodo","glimt","maccabi",
                 "club brugge","real racing"]

    soccer_intl = ["germany","england national","spain national","italy national",
                   "portugal national","netherlands","belgium national","croatia",
                   "denmark","sweden","norway","finland","switzerland","austria",
                   "poland","czech","slovakia","hungary","romania","bulgaria",
                   "serbia","ukraine","greece","scotland national","wales",
                   "northern ireland","ireland national","cyprus","estonia",
                   "albania","cameroon","morocco","egypt","algeria","nigeria",
                   "ghana","senegal","tunisia","ivory coast","côte d'ivoire",
                   "türkiye","turkiye","turkey national","argentina","colombia",
                   "chile","uruguay","venezuela","peru","ecuador","bolivia",
                   "paraguay","mexico national","costa rica","honduras","panama",
                   "jamaica","trinidad","haiti","cuba","usmnt","georgia national",
                   "portland timbers","seattle sounders","la galaxy","inter miami",
                   "atlanta united","nycfc","toronto fc","montreal","vancouver",
                   "colorado rapids","real salt lake"]

    ncaa = ["fighting camels","golden flashes","leopards","bison","ramblers",
            "revolutionaries","scarlet knights","hawkeyes","jaguars","bobcats",
            "cyclones","wolverines","islanders","wildcats","bulldogs","crimson tide",
            "longhorns","tar heels","blue devils","hoosiers","boilermakers",
            "buckeyes","sooners","cornhuskers","razorbacks","gamecocks","gators",
            "seminoles","hokies","terrapins","wolfpack","deacons","golden eagles",
            "campbell","bucknell","loyola","george washington","kent state",
            "northern illinois","south alabama","texas state","utah state",
            "utah valley","army","long beach","santa clara broncos","citadel",
            "samford","unlv","runnin rebels","anteaters","lumberjacks",
            "golden hurricane","ncaa"]

    # Priority order — most specific first
    if any(k in combined for k in esports):
        sport = "Esports"
    elif any(k in combined for k in nba) or "nba" in combined:
        sport = "NBA"
    elif any(k in combined for k in nhl) or "nhl" in combined:
        sport = "NHL"
    elif any(k in combined for k in nfl) or "nfl" in combined:
        sport = "NFL"
    elif any(k in combined for k in mlb) or "mlb" in combined:
        sport = "MLB"
    elif any(k in combined for k in tennis):
        sport = "Tennis"
    elif any(k in combined for k in liga_mx):
        sport = "Soccer (Liga MX)"
    elif any(k in combined for k in soccer_brazil):
        sport = "Soccer (Brazil)"
    elif any(k in combined for k in soccer_asia):
        sport = "Soccer (Asia/ME)"
    elif any(k in combined for k in soccer_eu):
        sport = "Soccer (EU)"
    elif any(k in combined for k in soccer_intl):
        sport = "Soccer (Intl)"
    elif any(k in combined for k in ncaa):
        sport = "NCAA"
    elif "mls" in combined:
        sport = "MLS"
    elif "crypto" in combined or "bitcoin" in combined or "eth" in combined:
        sport = "Crypto"
    elif "election" in combined or "president" in combined or "trump" in combined:
        sport = "Politics"
    else:
        sport = "Other"

    return market_type, sport
```


```python
# Apply to all activity and per-wallet dfs
activity["market_type"], activity["sport"] = zip(
    *activity.apply(
        lambda r: classify_market(r["title"], r["slug"]), axis=1
    )
)

for name in WALLETS:
    dfs[name]["market_type"], dfs[name]["sport"] = zip(
        *dfs[name].apply(
            lambda r: classify_market(r["title"], r["slug"]), axis=1
        )
    )

print("Classification complete.")
print("\nFull distribution (combined):")
print(activity["sport"].value_counts().to_string())
print(f"\nOther: {(activity['sport'] == 'Other').sum():,} rows")
print(f"Other %: {(activity['sport'] == 'Other').mean()*100:.1f}%")
```

    Classification complete.
    
    Full distribution (combined):
    sport
    Soccer (EU)         1798218
    NBA                 1391997
    Other               1232437
    Tennis               768943
    Esports              328487
    NHL                  275389
    NFL                  202923
    MLB                  195663
    Soccer (Brazil)      173624
    NCAA                 130685
    Soccer (Intl)        124215
    Soccer (Asia/ME)     120253
    Soccer (Liga MX)      74996
    MLS                   28149
    Crypto                   17
    Politics                  8
    
    Other: 1,232,437 rows
    Other %: 18.0%


### Sport distribution per wallet


```python
fig, axes = plt.subplots(1, 3, figsize=(18, 7))
fig.suptitle("Market Category Distribution per Wallet (by Trade Count)", 
             fontsize=14, fontweight="bold")

for ax, name in zip(axes, WALLETS):
    trades = dfs[name][dfs[name]["type"] == "TRADE"]
    dist   = trades["sport"].value_counts()
    total  = len(trades)
    color  = WALLET_COLORS[name]

    bars = ax.barh(
        dist.index[::-1],
        dist.values[::-1],
        color=color,
        alpha=0.85
    )

    for bar, count in zip(bars, dist.values[::-1]):
        pct = count / total * 100
        ax.text(
            bar.get_width() * 1.01,
            bar.get_y() + bar.get_height() / 2,
            f"{pct:.1f}%",
            va="center", fontsize=8
        )

    ax.set_title(name, fontweight="bold", color=color)
    ax.set_xlabel("Trade Count")
    ax.xaxis.set_major_formatter(
        mticker.FuncFormatter(lambda x, _: f"{x/1e6:.1f}M" if x >= 1e6 else f"{x/1e3:.0f}k")
    )

plt.tight_layout()
plt.savefig("plots/sport_distribution.png", dpi=150, bbox_inches="tight")
plt.show()
```


    
![png](02_analysis_files/02_analysis_28_0.png)
    


### Market type distribution per wallet


```python
fig, axes = plt.subplots(1, 3, figsize=(18, 6))
fig.suptitle("Market Type Distribution per Wallet (by Trade Count)",
             fontsize=14, fontweight="bold")

for ax, name in zip(axes, WALLETS):
    trades = dfs[name][dfs[name]["type"] == "TRADE"]
    dist   = trades["market_type"].value_counts()
    total  = len(trades)
    color  = WALLET_COLORS[name]

    bars = ax.barh(
        dist.index[::-1],
        dist.values[::-1],
        color=color,
        alpha=0.85
    )

    for bar, count in zip(bars, dist.values[::-1]):
        pct = count / total * 100
        ax.text(
            bar.get_width() * 1.01,
            bar.get_y() + bar.get_height() / 2,
            f"{pct:.1f}%",
            va="center", fontsize=8
        )

    ax.set_title(name, fontweight="bold", color=color)
    ax.set_xlabel("Trade Count")
    ax.xaxis.set_major_formatter(
        mticker.FuncFormatter(lambda x, _: f"{x/1e6:.1f}M" if x >= 1e6 else f"{x/1e3:.0f}k")
    )

plt.tight_layout()
plt.savefig("plots/market_type_distribution.png", dpi=150, bbox_inches="tight")
plt.show()
```


    
![png](02_analysis_files/02_analysis_30_0.png)
    


### Holding periods analysis


```python
# For each wallet+market, compute:
# - time of first buy
# - time of last activity (redeem, merge, or last buy)
# - holding period in hours

holding = []

for name in WALLETS:
    trades = dfs[name][dfs[name]["type"].isin(["TRADE", "REDEEM", "MERGE"])].copy()

    grouped = trades.groupby("conditionId").agg(
        first_buy  =("datetime", "min"),
        last_event =("datetime", "max"),
        has_redeem =("type", lambda x: "REDEEM" in x.values),
        sport      =("sport",   "last"),
        wallet     =("wallet",  "first"),
    ).reset_index()

    grouped["holding_hours"] = (
        grouped["last_event"] - grouped["first_buy"]
    ).dt.total_seconds() / 3600

    holding.append(grouped)

holding_df = pd.concat(holding, ignore_index=True)

print("Holding period summary (hours) per wallet:")
for name in WALLETS:
    h = holding_df[holding_df["wallet"] == name]["holding_hours"]
    print(f"\n{name}:")
    print(f"  median : {h.median():.1f}h")
    print(f"  mean   : {h.mean():.1f}h")
    print(f"  p25    : {h.quantile(0.25):.1f}h")
    print(f"  p75    : {h.quantile(0.75):.1f}h")
    print(f"  p95    : {h.quantile(0.95):.1f}h")
    print(f"  max    : {h.max():.1f}h")
```

    Holding period summary (hours) per wallet:
    
    rn1:
      median : 1.6h
      mean   : 51.6h
      p25    : 0.6h
      p75    : 3.3h
      p95    : 67.5h
      max    : 5016.9h
    
    sovereign:
      median : 12.2h
      mean   : 15.8h
      p25    : 7.0h
      p75    : 20.3h
      p95    : 28.5h
      max    : 2080.1h
    
    swisstony:
      median : 15.1h
      mean   : 32.0h
      p25    : 6.7h
      p75    : 35.5h
      p95    : 127.0h
      max    : 2514.7h



```python
fig, axes = plt.subplots(1, 3, figsize=(16, 5))
fig.suptitle("Holding Period Distribution (hours, capped at P95)", 
             fontsize=14, fontweight="bold")

for ax, name in zip(axes, WALLETS):
    h     = holding_df[holding_df["wallet"] == name]["holding_hours"]
    p95   = h.quantile(0.95)
    color = WALLET_COLORS[name]

    ax.hist(
        h[h <= p95],
        bins=60,
        color=color,
        alpha=0.85,
        edgecolor="none",
    )

    median = h.median()
    mean   = h.mean()
    ax.axvline(median, color="white",  linestyle="--", linewidth=1.5,
               label=f"Median {median:.1f}h")
    ax.axvline(mean,   color="yellow", linestyle="--", linewidth=1.5,
               label=f"Mean {mean:.1f}h")

    ax.set_title(name, fontweight="bold", color=color)
    ax.set_xlabel("Holding Period (hours)")
    ax.set_ylabel("Market Count")
    ax.legend(fontsize=8)

plt.tight_layout()
plt.savefig("plots/holding_period.png", dpi=150, bbox_inches="tight")
plt.show()
```


    
![png](02_analysis_files/02_analysis_33_0.png)
    


### Position sizing per market


```python
# Total USDC spent per market (cost basis per conditionId)
sizing = []

for name in WALLETS:
    trades = dfs[name][
        (dfs[name]["type"] == "TRADE") &
        (dfs[name]["side"] == "BUY")
    ].copy()

    per_market = (
        trades.groupby("conditionId")
        .agg(
            total_spent =("usdcSize", "sum"),
            trade_count =("usdcSize", "count"),
            sport       =("sport",    "last"),
            wallet      =("wallet",   "first"),
        )
        .reset_index()
    )

    sizing.append(per_market)

sizing_df = pd.concat(sizing, ignore_index=True)

print("Position size per market (USDC) per wallet:")
for name in WALLETS:
    s = sizing_df[sizing_df["wallet"] == name]["total_spent"]
    print(f"\n{name}:")
    print(f"  median : ${s.median():,.2f}")
    print(f"  mean   : ${s.mean():,.2f}")
    print(f"  p75    : ${s.quantile(0.75):,.2f}")
    print(f"  p95    : ${s.quantile(0.95):,.2f}")
    print(f"  p99    : ${s.quantile(0.99):,.2f}")
    print(f"  max    : ${s.max():,.2f}")
```

    Position size per market (USDC) per wallet:
    
    rn1:
      median : $544.64
      mean   : $3,228.33
      p75    : $2,446.50
      p95    : $15,121.46
      p99    : $41,378.54
      max    : $296,868.74
    
    sovereign:
      median : $1,017.96
      mean   : $4,180.65
      p75    : $4,504.27
      p95    : $18,611.68
      p99    : $36,892.09
      max    : $1,572,260.71
    
    swisstony:
      median : $712.18
      mean   : $4,023.11
      p75    : $2,938.96
      p95    : $17,792.75
      p99    : $51,020.68
      max    : $703,334.05



```python
fig, axes = plt.subplots(1, 3, figsize=(16, 5))
fig.suptitle("Position Size per Market (USDC, capped at P95)",
             fontsize=14, fontweight="bold")

for ax, name in zip(axes, WALLETS):
    s     = sizing_df[sizing_df["wallet"] == name]["total_spent"]
    p95   = s.quantile(0.95)
    color = WALLET_COLORS[name]

    ax.hist(
        s[s <= p95],
        bins=60,
        color=color,
        alpha=0.85,
        edgecolor="none",
    )

    median = s.median()
    mean   = s.mean()
    ax.axvline(median, color="white",  linestyle="--", linewidth=1.5,
               label=f"Median ${median:,.0f}")
    ax.axvline(mean,   color="yellow", linestyle="--", linewidth=1.5,
               label=f"Mean ${mean:,.0f}")

    ax.set_title(name, fontweight="bold", color=color)
    ax.set_xlabel("Total USDC per Market")
    ax.set_ylabel("Market Count")
    ax.legend(fontsize=8)
    ax.xaxis.set_major_formatter(
        mticker.FuncFormatter(lambda x, _: f"${x/1e3:.0f}k")
    )

plt.tight_layout()
plt.savefig("plots/position_sizing.png", dpi=150, bbox_inches="tight")
plt.show()
```


    
![png](02_analysis_files/02_analysis_36_0.png)
    



```python
# Average position size per sport per wallet
sport_sizing = (
    sizing_df
    .groupby(["wallet", "sport"])["total_spent"]
    .median()
    .reset_index()
    .rename(columns={"total_spent": "median_position_usdc"})
    .sort_values("median_position_usdc", ascending=False)
)

fig = px.bar(
    sport_sizing,
    x="sport",
    y="median_position_usdc",
    color="wallet",
    color_discrete_map=WALLET_COLORS,
    barmode="group",
    title="Median Position Size per Sport per Wallet (USDC)",
    labels={
        "sport": "Sport",
        "median_position_usdc": "Median Position (USDC)",
        "wallet": "Wallet"
    },
)
fig.update_layout(
    xaxis_tickangle=-35,
    yaxis_tickprefix="$",
    hovermode="x unified"
)
fig.show()
```



## Strategy reverse engineering

### Entry price distribution


```python
trades_buy = activity[
    (activity["type"] == "TRADE") &
    (activity["side"] == "BUY")
].copy()

fig, axes = plt.subplots(1, 3, figsize=(16, 5))
fig.suptitle("Entry Price Distribution (Implied Probability at Buy)",
             fontsize=14, fontweight="bold")

for ax, name in zip(axes, WALLETS):
    prices = trades_buy[trades_buy["wallet"] == name]["price"]
    color  = WALLET_COLORS[name]

    ax.hist(
        prices,
        bins=100,
        color=color,
        alpha=0.85,
        edgecolor="none",
    )

    median = prices.median()
    mean   = prices.mean()
    ax.axvline(median, color="white",  linestyle="--", linewidth=1.5,
               label=f"Median {median:.2f}")
    ax.axvline(mean,   color="yellow", linestyle="--", linewidth=1.5,
               label=f"Mean {mean:.2f}")

    # Mark key probability zones
    ax.axvline(0.5, color="red", linestyle=":", linewidth=1,
               alpha=0.7, label="50% (even)")

    ax.set_title(name, fontweight="bold", color=color)
    ax.set_xlabel("Entry Price (0 = 0%, 1 = 100%)")
    ax.set_ylabel("Trade Count")
    ax.set_xlim(0, 1)
    ax.legend(fontsize=8)

plt.tight_layout()
plt.savefig("plots/entry_price_distribution.png", dpi=150, bbox_inches="tight")
plt.show()
```


    
![png](02_analysis_files/02_analysis_40_0.png)
    


### Cross wallet market overlap


```python
# Get unique conditionIds per wallet
conditions = {
    name: set(dfs[name]["conditionId"].unique())
    for name in WALLETS
}

rn1_s   = conditions["rn1"]
sov_s   = conditions["sovereign"]
swiss_s = conditions["swisstony"]

# Pairwise overlaps
rn1_sov   = rn1_s   & sov_s
rn1_swiss = rn1_s   & swiss_s
sov_swiss = sov_s   & swiss_s
all_three = rn1_s   & sov_s & swiss_s

print("=== Market Overlap Analysis ===\n")
print(f"rn1 unique markets:       {len(rn1_s):,}")
print(f"sovereign unique markets: {len(sov_s):,}")
print(f"swisstony unique markets: {len(swiss_s):,}")
print(f"\nrn1 ∩ sovereign:          {len(rn1_sov):,}  "
      f"({len(rn1_sov)/len(rn1_s)*100:.1f}% of rn1, "
      f"{len(rn1_sov)/len(sov_s)*100:.1f}% of sovereign)")
print(f"rn1 ∩ swisstony:          {len(rn1_swiss):,}  "
      f"({len(rn1_swiss)/len(rn1_s)*100:.1f}% of rn1, "
      f"{len(rn1_swiss)/len(swiss_s)*100:.1f}% of swisstony)")
print(f"sovereign ∩ swisstony:    {len(sov_swiss):,}  "
      f"({len(sov_swiss)/len(sov_s)*100:.1f}% of sovereign, "
      f"{len(sov_swiss)/len(swiss_s)*100:.1f}% of swisstony)")
print(f"\nAll three overlap:        {len(all_three):,}")
```

    === Market Overlap Analysis ===
    
    rn1 unique markets:       55,717
    sovereign unique markets: 40,549
    swisstony unique markets: 85,235
    
    rn1 ∩ sovereign:          9,960  (17.9% of rn1, 24.6% of sovereign)
    rn1 ∩ swisstony:          43,743  (78.5% of rn1, 51.3% of swisstony)
    sovereign ∩ swisstony:    20,390  (50.3% of sovereign, 23.9% of swisstony)
    
    All three overlap:        7,645


### Overlap venn diagram


```python
# pip install matplotlib-venn if needed
# pip install matplotlib-venn

try:
    from matplotlib_venn import venn3
except ImportError:
    import subprocess
    subprocess.run(["pip", "install", "matplotlib-venn"], check=True)
    from matplotlib_venn import venn3

fig, ax = plt.subplots(1, 1, figsize=(8, 6))
fig.suptitle("Market Overlap Across Wallets\n(unique conditionIds)",
             fontsize=13, fontweight="bold")

v = venn3(
    subsets=[rn1_s, sov_s, swiss_s],
    set_labels=("rn1", "sovereign", "swisstony"),
    set_colors=(
        WALLET_COLORS["rn1"],
        WALLET_COLORS["sovereign"],
        WALLET_COLORS["swisstony"]
    ),
    alpha=0.6,
    ax=ax
)

plt.tight_layout()
plt.savefig("plots/market_overlap_venn.png", dpi=150, bbox_inches="tight")
plt.show()

print(f"\nExclusive markets (not shared):")
print(f"  rn1 only:       {len(rn1_s - sov_s - swiss_s):,}")
print(f"  sovereign only: {len(sov_s - rn1_s - swiss_s):,}")
print(f"  swisstony only: {len(swiss_s - rn1_s - sov_s):,}")
```


    
![png](02_analysis_files/02_analysis_44_0.png)
    


    
    Exclusive markets (not shared):
      rn1 only:       9,659
      sovereign only: 17,844
      swisstony only: 28,747


### Lead / follow timing analysis


```python
# For markets traded by both rn1 and swisstony,
# compare their first entry timestamp

rn1_first = (
    dfs["rn1"][dfs["rn1"]["type"] == "TRADE"]
    .groupby("conditionId")["timestamp"]
    .min()
    .rename("rn1_first")
)

swiss_first = (
    dfs["swisstony"][dfs["swisstony"]["type"] == "TRADE"]
    .groupby("conditionId")["timestamp"]
    .min()
    .rename("swiss_first")
)

sov_first = (
    dfs["sovereign"][dfs["sovereign"]["type"] == "TRADE"]
    .groupby("conditionId")["timestamp"]
    .min()
    .rename("sov_first")
)

# rn1 vs swisstony
rn1_vs_swiss = pd.concat([rn1_first, swiss_first], axis=1).dropna()
rn1_vs_swiss["diff_seconds"] = rn1_vs_swiss["rn1_first"] - rn1_vs_swiss["swiss_first"]
rn1_leads  = (rn1_vs_swiss["diff_seconds"] < 0).sum()
swiss_leads = (rn1_vs_swiss["diff_seconds"] > 0).sum()
same_time  = (rn1_vs_swiss["diff_seconds"] == 0).sum()

print("=== rn1 vs swisstony (shared markets) ===")
print(f"Total shared markets:  {len(rn1_vs_swiss):,}")
print(f"rn1 enters first:      {rn1_leads:,}  ({rn1_leads/len(rn1_vs_swiss)*100:.1f}%)")
print(f"swisstony enters first:{swiss_leads:,}  ({swiss_leads/len(rn1_vs_swiss)*100:.1f}%)")
print(f"Same timestamp:        {same_time:,}  ({same_time/len(rn1_vs_swiss)*100:.1f}%)")
print(f"\nMedian time diff: {rn1_vs_swiss['diff_seconds'].median()/60:.1f} minutes")
print(f"(negative = rn1 leads, positive = swisstony leads)")

# rn1 vs sovereign
rn1_vs_sov = pd.concat([rn1_first, sov_first], axis=1).dropna()
rn1_vs_sov["diff_seconds"] = rn1_vs_sov["rn1_first"] - rn1_vs_sov["sov_first"]
rn1_leads_sov = (rn1_vs_sov["diff_seconds"] < 0).sum()
sov_leads     = (rn1_vs_sov["diff_seconds"] > 0).sum()

print("\n=== rn1 vs sovereign (shared markets) ===")
print(f"Total shared markets:  {len(rn1_vs_sov):,}")
print(f"rn1 enters first:      {rn1_leads_sov:,}  ({rn1_leads_sov/len(rn1_vs_sov)*100:.1f}%)")
print(f"sovereign enters first:{sov_leads:,}  ({sov_leads/len(rn1_vs_sov)*100:.1f}%)")
print(f"\nMedian time diff: {rn1_vs_sov['diff_seconds'].median()/60:.1f} minutes")

# swisstony vs sovereign
swiss_vs_sov = pd.concat([swiss_first, sov_first], axis=1).dropna()
swiss_vs_sov["diff_seconds"] = swiss_vs_sov["swiss_first"] - swiss_vs_sov["sov_first"]
swiss_leads_sov = (swiss_vs_sov["diff_seconds"] < 0).sum()
sov_leads_swiss = (swiss_vs_sov["diff_seconds"] > 0).sum()

print("\n=== swisstony vs sovereign (shared markets) ===")
print(f"Total shared markets:  {len(swiss_vs_sov):,}")
print(f"swisstony enters first:{swiss_leads_sov:,}  ({swiss_leads_sov/len(swiss_vs_sov)*100:.1f}%)")
print(f"sovereign enters first:{sov_leads_swiss:,}  ({sov_leads_swiss/len(swiss_vs_sov)*100:.1f}%)")
print(f"\nMedian time diff: {swiss_vs_sov['diff_seconds'].median()/60:.1f} minutes")
```

    === rn1 vs swisstony (shared markets) ===
    Total shared markets:  43,738
    rn1 enters first:      5,670  (13.0%)
    swisstony enters first:37,616  (86.0%)
    Same timestamp:        452  (1.0%)
    
    Median time diff: 594.0 minutes
    (negative = rn1 leads, positive = swisstony leads)
    
    === rn1 vs sovereign (shared markets) ===
    Total shared markets:  9,959
    rn1 enters first:      1,714  (17.2%)
    sovereign enters first:8,138  (81.7%)
    
    Median time diff: 402.6 minutes
    
    === swisstony vs sovereign (shared markets) ===
    Total shared markets:  20,388
    swisstony enters first:5,819  (28.5%)
    sovereign enters first:13,868  (68.0%)
    
    Median time diff: 160.5 minutes



```python
fig, axes = plt.subplots(1, 3, figsize=(16, 5))
fig.suptitle("Entry Timing Difference Between Wallets (shared markets)\n"
             "Negative = left wallet leads, Positive = right wallet leads",
             fontsize=12, fontweight="bold")

pairs = [
    ("rn1 vs swisstony", rn1_vs_swiss["diff_seconds"] / 60,
     WALLET_COLORS["rn1"], WALLET_COLORS["swisstony"]),
    ("rn1 vs sovereign", rn1_vs_sov["diff_seconds"] / 60,
     WALLET_COLORS["rn1"], WALLET_COLORS["sovereign"]),
    ("swisstony vs sovereign", swiss_vs_sov["diff_seconds"] / 60,
     WALLET_COLORS["swisstony"], WALLET_COLORS["sovereign"]),
]

for ax, (label, diff, color_left, color_right) in zip(axes, pairs):
    # Cap at P2-P98 for readability
    p2  = diff.quantile(0.02)
    p98 = diff.quantile(0.98)
    capped = diff[(diff >= p2) & (diff <= p98)]

    ax.hist(capped, bins=80, color="#888888", alpha=0.7, edgecolor="none")

    # Color the negative side (left leads) and positive side (right leads)
    ax.axvspan(p2, 0,   alpha=0.15, color=color_left)
    ax.axvspan(0,  p98, alpha=0.15, color=color_right)
    ax.axvline(0, color="white", linewidth=1.5, linestyle="--")

    median = diff.median()
    ax.axvline(median, color="yellow", linewidth=1.5, linestyle="--",
               label=f"Median {median:.0f}min")

    ax.set_title(label, fontweight="bold")
    ax.set_xlabel("Time difference (minutes)")
    ax.set_ylabel("Market count")
    ax.legend(fontsize=8)

plt.tight_layout()
plt.savefig("plots/lead_follow_timing.png", dpi=150, bbox_inches="tight")
plt.show()
```


    
![png](02_analysis_files/02_analysis_47_0.png)
    


## Trade burst analysis


```python
def analyze_bursts(df, gap_seconds=60):
    """
    Detect trade bursts — clusters of rapid consecutive trades.
    
    A new burst starts when the gap between consecutive trades
    on the same market exceeds gap_seconds.
    
    Returns a DataFrame with one row per burst containing:
    - burst size (number of trades)
    - burst duration (seconds)
    - total USDC in burst
    - conditionId
    """
    trades = df[
        (df["type"] == "TRADE") & 
        (df["side"] == "BUY")
    ].copy().sort_values("timestamp")

    # Assign burst IDs — new burst when gap > threshold
    trades["prev_ts"]   = trades["timestamp"].shift(1)
    trades["gap"]       = trades["timestamp"] - trades["prev_ts"]
    trades["new_burst"] = (trades["gap"] > gap_seconds) | trades["gap"].isna()
    trades["burst_id"]  = trades["new_burst"].cumsum()

    bursts = (
        trades.groupby("burst_id")
        .agg(
            burst_size     =("timestamp",   "count"),
            burst_duration =("timestamp",   lambda x: x.max() - x.min()),
            total_usdc     =("usdcSize",    "sum"),
            conditionId    =("conditionId", "nunique"),
            wallet         =("wallet",      "first"),
        )
        .reset_index(drop=True)
    )

    return bursts
```


```python
burst_stats = {}

for name in WALLETS:
    bursts = analyze_bursts(dfs[name])
    burst_stats[name] = bursts

    single = (bursts["burst_size"] == 1).sum()
    multi  = (bursts["burst_size"] > 1).sum()
    total  = len(bursts)

    print(f"\n{name}:")
    print(f"  Total bursts:        {total:,}")
    print(f"  Single-trade bursts: {single:,} ({single/total*100:.1f}%)")
    print(f"  Multi-trade bursts:  {multi:,}  ({multi/total*100:.1f}%)")
    print(f"\n  Burst size distribution (multi-trade only):")
    mb = bursts[bursts["burst_size"] > 1]["burst_size"]
    print(f"    median : {mb.median():.0f} trades")
    print(f"    mean   : {mb.mean():.1f} trades")
    print(f"    p75    : {mb.quantile(0.75):.0f} trades")
    print(f"    p95    : {mb.quantile(0.95):.0f} trades")
    print(f"    max    : {mb.max():.0f} trades")
    print(f"\n  Burst duration (multi-trade, seconds):")
    md = bursts[bursts["burst_size"] > 1]["burst_duration"]
    print(f"    median : {md.median():.0f}s")
    print(f"    p95    : {md.quantile(0.95):.0f}s")
    print(f"    max    : {md.max():.0f}s")
```

    
    rn1:
      Total bursts:        50,328
      Single-trade bursts: 16,959 (33.7%)
      Multi-trade bursts:  33,369  (66.3%)
    
      Burst size distribution (multi-trade only):
        median : 5 trades
        mean   : 68.0 trades
        p75    : 16 trades
        p95    : 138 trades
        max    : 37787 trades
    
      Burst duration (multi-trade, seconds):
        median : 72s
        p95    : 950s
        max    : 46712s
    
    sovereign:
      Total bursts:        48,532
      Single-trade bursts: 14,083 (29.0%)
      Multi-trade bursts:  34,449  (71.0%)
    
      Burst size distribution (multi-trade only):
        median : 6 trades
        mean   : 34.3 trades
        p75    : 16 trades
        p95    : 98 trades
        max    : 33845 trades
    
      Burst duration (multi-trade, seconds):
        median : 82s
        p95    : 792s
        max    : 32728s
    
    swisstony:
      Total bursts:        41,067
      Single-trade bursts: 10,236 (24.9%)
      Multi-trade bursts:  30,831  (75.1%)
    
      Burst size distribution (multi-trade only):
        median : 7 trades
        mean   : 101.3 trades
        p75    : 21 trades
        p95    : 179 trades
        max    : 68282 trades
    
      Burst duration (multi-trade, seconds):
        median : 84s
        p95    : 1258s
        max    : 135718s



```python
fig, axes = plt.subplots(2, 3, figsize=(16, 10))
fig.suptitle("Trade Burst Analysis", fontsize=14, fontweight="bold")

for col, name in enumerate(WALLETS):
    bursts = burst_stats[name]
    multi  = bursts[bursts["burst_size"] > 1]
    color  = WALLET_COLORS[name]

    # Top row — burst size distribution (capped at P95)
    ax = axes[0, col]
    p95_size = multi["burst_size"].quantile(0.95)
    ax.hist(
        multi[multi["burst_size"] <= p95_size]["burst_size"],
        bins=60, color=color, alpha=0.85, edgecolor="none"
    )
    ax.axvline(multi["burst_size"].median(), color="white",
               linestyle="--", linewidth=1.5,
               label=f"Median {multi['burst_size'].median():.0f}")
    ax.set_title(f"{name} — Burst Size", fontweight="bold", color=color)
    ax.set_xlabel("Trades per burst")
    ax.set_ylabel("Burst count")
    ax.legend(fontsize=8)

    # Bottom row — burst duration distribution (capped at P95)
    ax = axes[1, col]
    p95_dur = multi["burst_duration"].quantile(0.95)
    ax.hist(
        multi[multi["burst_duration"] <= p95_dur]["burst_duration"] / 60,
        bins=60, color=color, alpha=0.85, edgecolor="none"
    )
    ax.axvline(multi["burst_duration"].median() / 60, color="white",
               linestyle="--", linewidth=1.5,
               label=f"Median {multi['burst_duration'].median()/60:.1f}min")
    ax.set_title(f"{name} — Burst Duration", fontweight="bold", color=color)
    ax.set_xlabel("Duration (minutes)")
    ax.set_ylabel("Burst count")
    ax.legend(fontsize=8)

plt.tight_layout()
plt.savefig("plots/burst_analysis.png", dpi=150, bbox_inches="tight")
plt.show()
```


    
![png](02_analysis_files/02_analysis_51_0.png)
    


### Position averaging behavior


```python
# For each wallet+market, count how many distinct bursts they entered
# A new burst = gap > 5 minutes between trades on same market

def count_entry_sessions(df, gap_minutes=5):
    """
    For each market, count distinct entry sessions.
    A new session starts when gap between trades on same market > gap_minutes.
    This is different from burst analysis — this is per-market, not global.
    """
    trades = df[
        (df["type"] == "TRADE") &
        (df["side"] == "BUY")
    ].copy().sort_values(["conditionId", "timestamp"])

    trades["prev_ts"] = trades.groupby("conditionId")["timestamp"].shift(1)
    trades["gap_min"] = (trades["timestamp"] - trades["prev_ts"]) / 60
    trades["new_session"] = (
        trades["gap_min"] > gap_minutes
    ) | trades["gap_min"].isna()

    sessions = (
        trades.groupby("conditionId")
        .agg(
            session_count  =("new_session", "sum"),
            total_trades   =("timestamp",   "count"),
            first_entry    =("timestamp",   "min"),
            last_entry     =("timestamp",   "max"),
            avg_price      =("price",       "mean"),
            price_std      =("price",       "std"),
            total_usdc     =("usdcSize",    "sum"),
            wallet         =("wallet",      "first"),
        )
        .reset_index()
    )

    # Entry span in hours
    sessions["entry_span_h"] = (
        sessions["last_entry"] - sessions["first_entry"]
    ) / 3600

    return sessions


session_stats = {}

for name in WALLETS:
    sessions = count_entry_sessions(dfs[name])
    session_stats[name] = sessions

    one   = (sessions["session_count"] == 1).sum()
    multi = (sessions["session_count"] > 1).sum()
    total = len(sessions)

    print(f"\n{name} — {total:,} markets:")
    print(f"  Single entry session:   {one:,}  ({one/total*100:.1f}%)")
    print(f"  Multiple entry sessions:{multi:,}  ({multi/total*100:.1f}%)")

    ms = sessions[sessions["session_count"] > 1]
    print(f"\n  Multi-session markets:")
    print(f"    median sessions : {ms['session_count'].median():.0f}")
    print(f"    max sessions    : {ms['session_count'].max():.0f}")
    print(f"    median span     : {ms['entry_span_h'].median():.1f}h")
    print(f"    median price std: {ms['price_std'].median():.4f}")
    print(f"    (price std = 0 means same price across sessions)")
```

    
    rn1 — 55,710 markets:
      Single entry session:   9,988  (17.9%)
      Multiple entry sessions:45,722  (82.1%)
    
      Multi-session markets:
        median sessions : 5
        max sessions    : 56
        median span     : 1.6h
        median price std: 0.2337
        (price std = 0 means same price across sessions)
    
    sovereign — 40,548 markets:
      Single entry session:   6,810  (16.8%)
      Multiple entry sessions:33,738  (83.2%)
    
      Multi-session markets:
        median sessions : 6
        max sessions    : 77
        median span     : 6.8h
        median price std: 0.0336
        (price std = 0 means same price across sessions)
    
    swisstony — 85,231 markets:
      Single entry session:   10,955  (12.9%)
      Multiple entry sessions:74,276  (87.1%)
    
      Multi-session markets:
        median sessions : 7
        max sessions    : 189
        median span     : 8.6h
        median price std: 0.1959
        (price std = 0 means same price across sessions)


### Concurrent posistion over time


```python
# For each day, how many unique markets is each wallet actively trading?
# "Active" = had at least one BUY trade on that day

concurrent = []

for name in WALLETS:
    trades = dfs[name][
        (dfs[name]["type"] == "TRADE") &
        (dfs[name]["side"] == "BUY")
    ].copy()

    trades["date"] = trades["datetime"].dt.date

    daily = (
        trades.groupby("date")["conditionId"]
        .nunique()
        .reset_index()
        .rename(columns={"conditionId": "active_markets"})
    )
    daily["wallet"] = name
    concurrent.append(daily)

concurrent_df = pd.concat(concurrent, ignore_index=True)

print("Daily active markets (concurrent positions):")
for name in WALLETS:
    d = concurrent_df[concurrent_df["wallet"] == name]["active_markets"]
    print(f"\n{name}:")
    print(f"  median : {d.median():.0f} markets/day")
    print(f"  mean   : {d.mean():.1f} markets/day")
    print(f"  p75    : {d.quantile(0.75):.0f} markets/day")
    print(f"  p95    : {d.quantile(0.95):.0f} markets/day")
    print(f"  max    : {d.max():.0f} markets/day")
```

    Daily active markets (concurrent positions):
    
    rn1:
      median : 145 markets/day
      mean   : 220.4 markets/day
      p75    : 321 markets/day
      p95    : 676 markets/day
      max    : 976 markets/day
    
    sovereign:
      median : 220 markets/day
      mean   : 248.0 markets/day
      p75    : 345 markets/day
      p95    : 543 markets/day
      max    : 922 markets/day
    
    swisstony:
      median : 498 markets/day
      mean   : 556.4 markets/day
      p75    : 797 markets/day
      p95    : 1475 markets/day
      max    : 2120 markets/day



```python
concurrent_df["date"] = pd.to_datetime(concurrent_df["date"])

fig = px.line(
    concurrent_df,
    x="date",
    y="active_markets",
    color="wallet",
    color_discrete_map=WALLET_COLORS,
    title="Daily Active Markets per Wallet (Concurrent Positions)",
    labels={
        "date":           "Date",
        "active_markets": "Unique Markets Traded",
        "wallet":         "Wallet"
    },
    markers=False,
)

fig.update_layout(
    hovermode="x unified",
    yaxis_title="Markets per Day",
)

fig.show()
```


