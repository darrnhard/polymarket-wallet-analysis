# API Fetching Notebook

## Import 


```python
import os
import time
import requests
import json
import pandas as pd
from datetime import datetime, timezone

pd.set_option("display.max_columns", None)
pd.set_option("display.max_colwidth", 60)
```

## Config


```python
WALLETS = {
    "rn1":      "0x2005d16a84ceefa912d4e380cd32e7ff827875ea",
    "sovereign": "0xee613b3fc183ee44f9da9c05f53e2da107e3debf",
    "swisstony": "0x204f72f35326db932158cba6adff0b9a1da95e14",
}

DATA_API  = "https://data-api.polymarket.com"   # positions, trades, activity
GAMMA_API = "https://gamma-api.polymarket.com"  # profiles, market metadata

DATA_DIR    = "data"
LEDGER_FILE = f"{DATA_DIR}/ledger.json"
os.makedirs(DATA_DIR, exist_ok=True)
```

## Ledger Setup


```python
def load_ledger():
    if os.path.exists(LEDGER_FILE):
        with open(LEDGER_FILE) as f:
            return json.load(f)
    return {name: [] for name in WALLETS}

def save_ledger(ledger):
    with open(LEDGER_FILE, "w") as f:
        json.dump(ledger, f, indent=2)

def window_done(ledger, wallet_name, win_start, win_end):
    """Check if this exact window is already in the ledger."""
    return [win_start, win_end] in ledger.get(wallet_name, [])

def mark_window_done(ledger, wallet_name, win_start, win_end):
    """Record a completed window in the ledger and flush to disk."""
    ledger.setdefault(wallet_name, []).append([win_start, win_end])
    save_ledger(ledger)

ledger = load_ledger()
print("Ledger loaded:", {k: len(v) for k, v in ledger.items()}, "windows completed")
```

    Ledger loaded: {'rn1': 0, 'sovereign': 0, 'swisstony': 0} windows completed


## HTTP Caller


```python
import time

def get(base_url, endpoint, params=None, retries=3, backoff=2.0):
    """
    GET request with simple retry on transient connection errors.

    retries: number of attempts before giving up
    backoff: seconds to wait between attempts (doubles each retry)
    """
    url = f"{base_url}{endpoint}"
    wait = backoff

    for attempt in range(1, retries + 1):
        try:
            response = requests.get(url, params=params, timeout=30)
            if not response.ok:
                raise RuntimeError(f"HTTP {response.status_code}: {response.text}")
            return response.json()

        except (requests.exceptions.ConnectionError,
                requests.exceptions.Timeout) as e:
            if attempt == retries:
                raise  # give up after final attempt
            print(f"  ⚠ connection error (attempt {attempt}/{retries}), retrying in {wait}s...")
            time.sleep(wait)
            wait *= 2  # exponential backoff: 2s → 4s → 8s
```

## Fetch Exploration

### Position


```python
positions = {}

for name, address in WALLETS.items():
    data = get(DATA_API, "/positions", params={"user": address, "limit": 500})
    positions[name] = data
    print(f"\n=== {name} — {len(data)} open positions ===")
    if data:
        print("Fields:", list(data[0].keys()))
        print("First record:")
        print(json.dumps(data[0], indent=2))
```

    
    === rn1 — 500 open positions ===
    Fields: ['proxyWallet', 'asset', 'conditionId', 'size', 'avgPrice', 'initialValue', 'currentValue', 'cashPnl', 'percentPnl', 'totalBought', 'realizedPnl', 'percentRealizedPnl', 'curPrice', 'redeemable', 'mergeable', 'title', 'slug', 'icon', 'eventId', 'eventSlug', 'outcome', 'outcomeIndex', 'oppositeOutcome', 'oppositeAsset', 'endDate', 'negativeRisk']
    First record:
    {
      "proxyWallet": "0x2005d16a84ceefa912d4e380cd32e7ff827875ea",
      "asset": "77985259937045308793816579916638590572189987943927681873921893711796592307973",
      "conditionId": "0xa9a9acb457b3567dd8d1cacaa400ae6aec8e63f858a4560afdb612161e43e57c",
      "size": 181369.1314,
      "avgPrice": 0.6281,
      "initialValue": 113919.221,
      "currentValue": 58944.9677,
      "cashPnl": -54974.2533,
      "percentPnl": -48.2572,
      "totalBought": 181684.2282,
      "realizedPnl": 0,
      "percentRealizedPnl": -48.3469,
      "curPrice": 0.325,
      "redeemable": false,
      "mergeable": true,
      "title": "Timberwolves vs. Nuggets",
      "slug": "nba-min-den-2026-04-20",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/super+cool+basketball+in+red+and+blue+wow.png",
      "eventId": "382090",
      "eventSlug": "nba-min-den-2026-04-20",
      "outcome": "Nuggets",
      "outcomeIndex": 1,
      "oppositeOutcome": "Timberwolves",
      "oppositeAsset": "101049751548410708430293002292169561720588907594020452919685141368802218026926",
      "endDate": "2026-04-21",
      "negativeRisk": false
    }
    
    === sovereign — 77 open positions ===
    Fields: ['proxyWallet', 'asset', 'conditionId', 'size', 'avgPrice', 'initialValue', 'currentValue', 'cashPnl', 'percentPnl', 'totalBought', 'realizedPnl', 'percentRealizedPnl', 'curPrice', 'redeemable', 'mergeable', 'title', 'slug', 'icon', 'eventId', 'eventSlug', 'outcome', 'outcomeIndex', 'oppositeOutcome', 'oppositeAsset', 'endDate', 'negativeRisk']
    First record:
    {
      "proxyWallet": "0xee613b3fc183ee44f9da9c05f53e2da107e3debf",
      "asset": "101049751548410708430293002292169561720588907594020452919685141368802218026926",
      "conditionId": "0xa9a9acb457b3567dd8d1cacaa400ae6aec8e63f858a4560afdb612161e43e57c",
      "size": 21010.072,
      "avgPrice": 0.364,
      "initialValue": 7648.3385,
      "currentValue": 14181.7986,
      "cashPnl": 6533.46,
      "percentPnl": 85.4232,
      "totalBought": 39198.0727,
      "realizedPnl": 2401.2866,
      "percentRealizedPnl": -0.6135,
      "curPrice": 0.675,
      "redeemable": false,
      "mergeable": true,
      "title": "Timberwolves vs. Nuggets",
      "slug": "nba-min-den-2026-04-20",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/super+cool+basketball+in+red+and+blue+wow.png",
      "eventId": "382090",
      "eventSlug": "nba-min-den-2026-04-20",
      "outcome": "Timberwolves",
      "outcomeIndex": 0,
      "oppositeOutcome": "Nuggets",
      "oppositeAsset": "77985259937045308793816579916638590572189987943927681873921893711796592307973",
      "endDate": "2026-04-21",
      "negativeRisk": false
    }
    
    === swisstony — 500 open positions ===
    Fields: ['proxyWallet', 'asset', 'conditionId', 'size', 'avgPrice', 'initialValue', 'currentValue', 'cashPnl', 'percentPnl', 'totalBought', 'realizedPnl', 'percentRealizedPnl', 'curPrice', 'redeemable', 'mergeable', 'title', 'slug', 'icon', 'eventId', 'eventSlug', 'outcome', 'outcomeIndex', 'oppositeOutcome', 'oppositeAsset', 'endDate', 'negativeRisk']
    First record:
    {
      "proxyWallet": "0x204f72f35326db932158cba6adff0b9a1da95e14",
      "asset": "74242218918645333207499760215586417516176977896360830393316065607539566187019",
      "conditionId": "0xee67dc2d6cdb80df4149af594158ff4d6f8b76780b35f156845c85488aed5e36",
      "size": 15676.6855,
      "avgPrice": 0.5322,
      "initialValue": 8344.6683,
      "currentValue": 8230.2599,
      "cashPnl": -114.4084,
      "percentPnl": -1.371,
      "totalBought": 15725.4906,
      "realizedPnl": 0,
      "percentRealizedPnl": -1.6771,
      "curPrice": 0.525,
      "redeemable": false,
      "mergeable": true,
      "title": "Will FC Internazionale Milano win on 2026-04-21?",
      "slug": "itc-int-com-2026-04-21-int",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/itc.png",
      "eventId": "306840",
      "eventSlug": "itc-int-com-2026-04-21",
      "outcome": "Yes",
      "outcomeIndex": 0,
      "oppositeOutcome": "No",
      "oppositeAsset": "77021733609879286707746138781235777402898124697034997655549907257716356269823",
      "endDate": "2026-04-21",
      "negativeRisk": true
    }


### Values


```python
values = {}

for name, address in WALLETS.items():
    data = get(DATA_API, "/value", params={"user": address})
    values[name] = data
    print(f"{name}: {json.dumps(data, indent=2)}")
```

    rn1: [
      {
        "user": "0x2005d16a84ceefa912d4e380cd32e7ff827875ea",
        "value": 517349.941
      }
    ]
    sovereign: [
      {
        "user": "0xee613b3fc183ee44f9da9c05f53e2da107e3debf",
        "value": 82537.4196
      }
    ]
    swisstony: [
      {
        "user": "0x204f72f35326db932158cba6adff0b9a1da95e14",
        "value": 200664.8995
      }
    ]


### Activity


```python
activity_sample = {}

for name, address in WALLETS.items():
    data = get(DATA_API, "/activity", params={
        "user":          address,
        "limit":         10,           # small on purpose — just inspecting shape
        "sortDirection": "DESC",       # most recent first
    })
    activity_sample[name] = data
    print(f"\n=== {name} — showing {len(data)} records ===")
    if data:
        print("Fields:", list(data[0].keys()))
        print("Most recent event:")
        print(json.dumps(data[0], indent=2))
        print(f"Oldest in this page:")
        print(json.dumps(data[-1], indent=2))
```

    
    === rn1 — showing 10 records ===
    Fields: ['proxyWallet', 'timestamp', 'conditionId', 'type', 'size', 'usdcSize', 'transactionHash', 'price', 'asset', 'side', 'outcomeIndex', 'title', 'slug', 'icon', 'eventSlug', 'outcome', 'name', 'pseudonym', 'bio', 'profileImage', 'profileImageOptimized']
    Most recent event:
    {
      "proxyWallet": "0x2005d16a84ceefa912d4e380cd32e7ff827875ea",
      "timestamp": 1776748930,
      "conditionId": "0xf743001c011ce91856ef3eca6daacbce587a98d88ebbb8669c65437b4e1db06e",
      "type": "TRADE",
      "size": 4.02,
      "usdcSize": 1.8492,
      "transactionHash": "0x395afda3f5a3efe1b1842ab2a0ab3836ccd0331c980414e1748421de2c4598ac",
      "price": 0.46,
      "asset": "113848110479592031259253354608837956257747317426814833913131888568473157087516",
      "side": "BUY",
      "outcomeIndex": 1,
      "title": "Gwangju: Alex Bolt vs Ilia Simakin",
      "slug": "atp-bolt-simakin-2026-04-19",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/atp-tour-b4390c4fb8.jpg",
      "eventSlug": "atp-bolt-simakin-2026-04-19",
      "outcome": "Ilia Simakin",
      "name": "RN1",
      "pseudonym": "Scary-Edible",
      "bio": "",
      "profileImage": "https://polymarket-upload.s3.us-east-2.amazonaws.com/profile-image-1567750-84001aec-310a-470f-9584-67dfcaa61267.png",
      "profileImageOptimized": ""
    }
    Oldest in this page:
    {
      "proxyWallet": "0x2005d16a84ceefa912d4e380cd32e7ff827875ea",
      "timestamp": 1776748904,
      "conditionId": "0xa9a9acb457b3567dd8d1cacaa400ae6aec8e63f858a4560afdb612161e43e57c",
      "type": "TRADE",
      "size": 78.947367,
      "usdcSize": 63.947368,
      "transactionHash": "0xdc17032436f5e03f36748f28d08e3c51a04b5ef046e9e0f67c704e0f5f9ad3da",
      "price": 0.8100000092466668,
      "asset": "101049751548410708430293002292169561720588907594020452919685141368802218026926",
      "side": "BUY",
      "outcomeIndex": 0,
      "title": "Timberwolves vs. Nuggets",
      "slug": "nba-min-den-2026-04-20",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/super+cool+basketball+in+red+and+blue+wow.png",
      "eventSlug": "nba-min-den-2026-04-20",
      "outcome": "Timberwolves",
      "name": "RN1",
      "pseudonym": "Scary-Edible",
      "bio": "",
      "profileImage": "https://polymarket-upload.s3.us-east-2.amazonaws.com/profile-image-1567750-84001aec-310a-470f-9584-67dfcaa61267.png",
      "profileImageOptimized": ""
    }
    
    === sovereign — showing 10 records ===
    Fields: ['proxyWallet', 'timestamp', 'conditionId', 'type', 'size', 'usdcSize', 'transactionHash', 'price', 'asset', 'side', 'outcomeIndex', 'title', 'slug', 'icon', 'eventSlug', 'outcome', 'name', 'pseudonym', 'bio', 'profileImage', 'profileImageOptimized']
    Most recent event:
    {
      "proxyWallet": "0xee613b3fc183ee44f9da9c05f53e2da107e3debf",
      "timestamp": 1776748908,
      "conditionId": "0x09ad3ebd93fc9fcd06a3cdf3ef87e455680df700819f9152091c871e3eeb3a0d",
      "type": "TRADE",
      "size": 10,
      "usdcSize": 4.7,
      "transactionHash": "0xe596b123af4af90ae6c7a7aa93e5035bfc1bcd57ed5b34967eaf82dfddbec70e",
      "price": 0.47,
      "asset": "38458219072631761052185748920826760655200598268151429309818729848162840263562",
      "side": "BUY",
      "outcomeIndex": 0,
      "title": "Spread: Rockets (-5.5)",
      "slug": "nba-hou-lal-2026-04-21-spread-away-5pt5",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/super+cool+basketball+in+red+and+blue+wow.png",
      "eventSlug": "nba-hou-lal-2026-04-21",
      "outcome": "Rockets",
      "name": "sovereign2013",
      "pseudonym": "Ultimate-Locality",
      "bio": "",
      "profileImage": "",
      "profileImageOptimized": ""
    }
    Oldest in this page:
    {
      "proxyWallet": "0xee613b3fc183ee44f9da9c05f53e2da107e3debf",
      "timestamp": 1776748368,
      "conditionId": "0x5eb7402498c07e888c6b89f0413a41b950f62974b4745ad211d6423e377345dd",
      "type": "REDEEM",
      "size": 1665.48021,
      "usdcSize": 1665.48021,
      "transactionHash": "0x93270358e6a93d6f059fb42ef9c4bf5d2b1e92145983b3fa0f6aeb4617c5cc0a",
      "price": 0,
      "asset": "",
      "side": "",
      "outcomeIndex": 999,
      "title": "Athletics vs. Seattle Mariners: O/U 7.5",
      "slug": "mlb-oak-sea-2026-04-20-total-7pt5",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/Repetitive-markets/MLB.jpg",
      "eventSlug": "mlb-oak-sea-2026-04-20",
      "outcome": "",
      "name": "sovereign2013",
      "pseudonym": "Ultimate-Locality",
      "bio": "",
      "profileImage": "",
      "profileImageOptimized": ""
    }
    
    === swisstony — showing 10 records ===
    Fields: ['proxyWallet', 'timestamp', 'conditionId', 'type', 'size', 'usdcSize', 'transactionHash', 'price', 'asset', 'side', 'outcomeIndex', 'title', 'slug', 'icon', 'eventSlug', 'outcome', 'name', 'pseudonym', 'bio', 'profileImage', 'profileImageOptimized']
    Most recent event:
    {
      "proxyWallet": "0x204f72f35326db932158cba6adff0b9a1da95e14",
      "timestamp": 1776748916,
      "conditionId": "0x45b6509d2504a671940c7c9967e56122bb30a4823b7d06bdad03d4f5a1021ea8",
      "type": "REDEEM",
      "size": 1580.949639,
      "usdcSize": 1580.949639,
      "transactionHash": "0x7da685cd17e7d4e3c615de8148f95390e12090f6ce10c22c3f70e77a72f18e90",
      "price": 0,
      "asset": "",
      "side": "",
      "outcomeIndex": 999,
      "title": "Senators vs. Hurricanes",
      "slug": "nhl-ott-car-2026-04-20",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/nhl.png",
      "eventSlug": "nhl-ott-car-2026-04-20",
      "outcome": "",
      "name": "swisstony",
      "pseudonym": "Frail-Possible",
      "bio": "loincloth chic",
      "profileImage": "",
      "profileImageOptimized": ""
    }
    Oldest in this page:
    {
      "proxyWallet": "0x204f72f35326db932158cba6adff0b9a1da95e14",
      "timestamp": 1776748614,
      "conditionId": "0xa9a9acb457b3567dd8d1cacaa400ae6aec8e63f858a4560afdb612161e43e57c",
      "type": "TRADE",
      "size": 333.5082,
      "usdcSize": 155.94,
      "transactionHash": "0x3214d653242cd54409ef7acd906ddb5524b2afb4169e83c8257cab4baca591b7",
      "price": 0.46,
      "asset": "77985259937045308793816579916638590572189987943927681873921893711796592307973",
      "side": "BUY",
      "outcomeIndex": 1,
      "title": "Timberwolves vs. Nuggets",
      "slug": "nba-min-den-2026-04-20",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/super+cool+basketball+in+red+and+blue+wow.png",
      "eventSlug": "nba-min-den-2026-04-20",
      "outcome": "Nuggets",
      "name": "swisstony",
      "pseudonym": "Frail-Possible",
      "bio": "loincloth chic",
      "profileImage": "",
      "profileImageOptimized": ""
    }


### Erliest Activity


```python
for name, address in WALLETS.items():
    data = get(DATA_API, "/activity", params={
        "user":          address,
        "limit":         1,
        "sortDirection": "ASC",   # flip to oldest first
    })
    if data:
        ts = data[0]["timestamp"]
        dt = datetime.fromtimestamp(ts, tz=timezone.utc)
        print(f"{name}: first activity {dt.strftime('%Y-%m-%d')}  (unix: {ts})")
```

    rn1: first activity 2025-07-09  (unix: 1752062570)
    sovereign: first activity 2025-08-07  (unix: 1754578485)
    swisstony: first activity 2025-08-09  (unix: 1754716147)


## Full activity backfill with time-window chunking


```python
from datetime import datetime, timezone, timedelta
MIN_WINDOW_SECONDS = 3600  # 1 hour floor — prevents infinite recursion
```


```python
def save_chunk(records, wallet_name):
    """
    Append a batch of records to the wallet's parquet file.
    Uses parquet partitioning by appending a new row group.
    If file doesn't exist yet, creates it.
    """
    if not records:
        return

    path = f"{DATA_DIR}/activity_{wallet_name}.parquet"
    df   = pd.DataFrame(records)

    if os.path.exists(path):
        existing = pd.read_parquet(path)
        combined = pd.concat([existing, df], ignore_index=True)
        # Deduplicate on transactionHash at save time as a safety net
        combined = combined.drop_duplicates(subset=["transactionHash"])
        combined.to_parquet(path, index=False)
    else:
        df.drop_duplicates(subset=["transactionHash"]).to_parquet(path, index=False)
```


```python
def fetch_window(address, win_start, win_end):
    """
    Fetch all records in a single time window via offset pagination.

    Returns (records, hit_cap).
    hit_cap=True means the window is too dense and should be split.
    We stop at offset 3000 (not 3500) to stay safely under the hard cap.
    """
    all_records = {}
    offset      = 0
    limit       = 500
    SAFE_CAP    = 3000

    while True:
        if offset > SAFE_CAP:
            return list(all_records.values()), True

        batch = get(DATA_API, "/activity", params={
            "user":          address,
            "start":         win_start,
            "end":           win_end,
            "limit":         limit,
            "offset":        offset,
            "sortDirection": "ASC",
        })

        if not batch:
            break

        for record in batch:
            key = record.get("transactionHash") or f"{record['timestamp']}_{record['conditionId']}"
            all_records[key] = record

        if len(batch) < limit:
            break
        
        time.sleep(0.5)
        offset += limit

    return list(all_records.values()), False
```


```python
def fetch_recursive(address, win_start, win_end, depth=0):
    """
    Recursively fetch all activity in [win_start, win_end].

    If the window overflows the offset cap, bisect it and
    recurse into each half. Continues until windows fit
    or the minimum window size is reached.

    depth is tracked only for readable indentation in logs.
    """
    indent = "  " * (depth + 1)
    label  = datetime.fromtimestamp(win_start, tz=timezone.utc).strftime("%Y-%m-%d %H:%M")
    span   = win_end - win_start

    records, hit_cap = fetch_window(address, win_start, win_end)

    if not hit_cap:
        print(f"{indent}{label} (+{span//3600}h) | +{len(records)} records")
        return records

    # Window overflowed — check if we can still bisect
    if span <= MIN_WINDOW_SECONDS:
        print(f"{indent}⚠️  {label} | minimum window reached with overflow — data may be incomplete")
        return records

    # Bisect and recurse
    mid = (win_start + win_end) // 2
    print(f"{indent}{label} | overflow → bisecting into two {span//2//3600}h windows")

    left  = fetch_recursive(address, win_start, mid, depth + 1)
    right = fetch_recursive(address, mid, win_end, depth + 1)

    # Deduplicate across halves (bisect boundary could duplicate a record)
    merged = {}
    for record in left + right:
        key = record.get("transactionHash") or f"{record['timestamp']}_{record['conditionId']}"
        merged[key] = record

    return list(merged.values())
```


```python
def fetch_activity_full(address, wallet_name, start_ts, end_ts, ledger, initial_window_days=7):
    """
    Fetch complete activity history, with per-window checkpointing.
    Skips any window already recorded in the ledger.
    Appends each completed window to the wallet's parquet file immediately.
    """
    current = start_ts
    delta   = initial_window_days * 86400
    total   = 0

    while current < end_ts:
        win_end = min(current + delta, end_ts)

        if window_done(ledger, wallet_name, current, win_end):
            print(f"  skip {datetime.fromtimestamp(current, tz=timezone.utc).strftime('%Y-%m-%d')} (already fetched)")
            current = win_end
            continue

        # Fetch this window (with recursive bisection if needed)
        records = fetch_recursive(address, current, win_end, depth=0)

        # Checkpoint: write to disk THEN mark ledger
        save_chunk(records, wallet_name)
        mark_window_done(ledger, wallet_name, current, win_end)

        total  += len(records)
        current = win_end

    return total
```


```python
# Full history from each wallet's first known activity
WALLET_FIRST_TS = {
    "rn1":       1752062570,   # 2025-07-09
    "sovereign":  1754578485,   # 2025-08-07
    "swisstony":  1754716147,   # 2025-08-09
}

NOW_TS = int(datetime.now(tz=timezone.utc).timestamp())
```


```python
ledger = load_ledger()

for name, address in WALLETS.items():
    start_dt = datetime.fromtimestamp(WALLET_FIRST_TS[name], tz=timezone.utc).strftime("%Y-%m-%d")
    print(f"\n{'='*50}")
    print(f"Fetching {name} from {start_dt}")
    total = fetch_activity_full(address, name, WALLET_FIRST_TS[name], NOW_TS, ledger)
    print(f"→ {name}: +{total} new records this run")
```
