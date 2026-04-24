# API Fetching Notebook

## Import 


```python
import os
import time
import requests
import json
import pandas as pd
import numpy as np
from datetime import datetime, timezone, timedelta

pd.options.future.infer_string = False
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

    Ledger loaded: {'rn1': 42, 'sovereign': 38, 'swisstony': 37} windows completed


## HTTP Caller


```python
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
      "asset": "1085310679959577834629905872167609894989809542716168749768240119209698690157",
      "conditionId": "0xd57db42dc26f8cdd6f26e9148ba5b40775af15ea104b6d0068ebfd4c7342d7aa",
      "size": 168767.8979,
      "avgPrice": 0.3264,
      "initialValue": 55090.7361,
      "currentValue": 22783.6662,
      "cashPnl": -32307.0699,
      "percentPnl": -58.6433,
      "totalBought": 169734.0224,
      "realizedPnl": 0,
      "percentRealizedPnl": -58.8787,
      "curPrice": 0.135,
      "redeemable": false,
      "mergeable": true,
      "title": "Rockets vs. Lakers",
      "slug": "nba-hou-lal-2026-04-21",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/super+cool+basketball+in+red+and+blue+wow.png",
      "eventId": "382093",
      "eventSlug": "nba-hou-lal-2026-04-21",
      "outcome": "Rockets",
      "outcomeIndex": 0,
      "oppositeOutcome": "Lakers",
      "oppositeAsset": "45949080565643676028230495363401189884973675894037073023779613649229440981535",
      "endDate": "2026-04-22",
      "negativeRisk": false
    }
    
    === sovereign — 39 open positions ===
    Fields: ['proxyWallet', 'asset', 'conditionId', 'size', 'avgPrice', 'initialValue', 'currentValue', 'cashPnl', 'percentPnl', 'totalBought', 'realizedPnl', 'percentRealizedPnl', 'curPrice', 'redeemable', 'mergeable', 'title', 'slug', 'icon', 'eventId', 'eventSlug', 'outcome', 'outcomeIndex', 'oppositeOutcome', 'oppositeAsset', 'endDate', 'negativeRisk']
    First record:
    {
      "proxyWallet": "0xee613b3fc183ee44f9da9c05f53e2da107e3debf",
      "asset": "83993024274387765539759485552031958770178918792546828081099848196418729481237",
      "conditionId": "0x2d87bca4fdb59ce4a91fd4f5bd5ca7ddd19a15097ff2f7b508bca3df99911b28",
      "size": 10784.55,
      "avgPrice": 0.4197,
      "initialValue": 4527.2354,
      "currentValue": 4475.5882,
      "cashPnl": -51.6472,
      "percentPnl": -1.1408,
      "totalBought": 12389.55,
      "realizedPnl": 128.7386,
      "percentRealizedPnl": -13.9474,
      "curPrice": 0.415,
      "redeemable": false,
      "mergeable": true,
      "title": "Suns vs. Thunder: O/U 212.5",
      "slug": "nba-phx-okc-2026-04-22-total-212pt5",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/super+cool+basketball+in+red+and+blue+wow.png",
      "eventId": "391579",
      "eventSlug": "nba-phx-okc-2026-04-22",
      "outcome": "Under",
      "outcomeIndex": 1,
      "oppositeOutcome": "Over",
      "oppositeAsset": "107522066901789299119423528273975254094927988780730589689435885467506648622631",
      "endDate": "2026-04-23",
      "negativeRisk": false
    }
    
    === swisstony — 500 open positions ===
    Fields: ['proxyWallet', 'asset', 'conditionId', 'size', 'avgPrice', 'initialValue', 'currentValue', 'cashPnl', 'percentPnl', 'totalBought', 'realizedPnl', 'percentRealizedPnl', 'curPrice', 'redeemable', 'mergeable', 'title', 'slug', 'icon', 'eventId', 'eventSlug', 'outcome', 'outcomeIndex', 'oppositeOutcome', 'oppositeAsset', 'endDate', 'negativeRisk']
    First record:
    {
      "proxyWallet": "0x204f72f35326db932158cba6adff0b9a1da95e14",
      "asset": "32970592838518231857960834683525212992911465765611987905622054312859827533999",
      "conditionId": "0x8aa2f0b3b9edd1b07163286d3159c3a501d94cf0808841d5274336c68b1f7d44",
      "size": 26214.4958,
      "avgPrice": 0.6349,
      "initialValue": 16645.0514,
      "currentValue": 17170.4947,
      "cashPnl": 525.4432,
      "percentPnl": 3.1567,
      "totalBought": 26485.9946,
      "realizedPnl": 0,
      "percentRealizedPnl": 2.0993,
      "curPrice": 0.655,
      "redeemable": false,
      "mergeable": true,
      "title": "Will Club Atl\u00e9tico de Madrid win on 2026-04-22?",
      "slug": "lal-elc-mad-2026-04-22-mad",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/league-lal.png",
      "eventId": "359811",
      "eventSlug": "lal-elc-mad-2026-04-22",
      "outcome": "No",
      "outcomeIndex": 1,
      "oppositeOutcome": "Yes",
      "oppositeAsset": "17684283368478579011485556298092659675294671893414587403391837552748491189817",
      "endDate": "2026-04-22",
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
        "value": 547960.1841
      }
    ]
    sovereign: [
      {
        "user": "0xee613b3fc183ee44f9da9c05f53e2da107e3debf",
        "value": 26881.2012
      }
    ]
    swisstony: [
      {
        "user": "0x204f72f35326db932158cba6adff0b9a1da95e14",
        "value": 583961.2258
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
      "timestamp": 1776834846,
      "conditionId": "0xd57db42dc26f8cdd6f26e9148ba5b40775af15ea104b6d0068ebfd4c7342d7aa",
      "type": "TRADE",
      "size": 27.6,
      "usdcSize": 26.772,
      "transactionHash": "0x55f5ebebc817457bdae312e378d536f91f80f59eb6674796046da6353753018d",
      "price": 0.97,
      "asset": "45949080565643676028230495363401189884973675894037073023779613649229440981535",
      "side": "BUY",
      "outcomeIndex": 1,
      "title": "Rockets vs. Lakers",
      "slug": "nba-hou-lal-2026-04-21",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/super+cool+basketball+in+red+and+blue+wow.png",
      "eventSlug": "nba-hou-lal-2026-04-21",
      "outcome": "Lakers",
      "name": "RN1",
      "pseudonym": "Scary-Edible",
      "bio": "",
      "profileImage": "https://polymarket-upload.s3.us-east-2.amazonaws.com/profile-image-1567750-84001aec-310a-470f-9584-67dfcaa61267.png",
      "profileImageOptimized": ""
    }
    Oldest in this page:
    {
      "proxyWallet": "0x2005d16a84ceefa912d4e380cd32e7ff827875ea",
      "timestamp": 1776834840,
      "conditionId": "0xd57db42dc26f8cdd6f26e9148ba5b40775af15ea104b6d0068ebfd4c7342d7aa",
      "type": "TRADE",
      "size": 33.333332,
      "usdcSize": 32.333333,
      "transactionHash": "0x86126652665b8ca88af078f75405ec1bf48001ac42780d7c469016f2623f50cd",
      "price": 0.9700000288000011,
      "asset": "45949080565643676028230495363401189884973675894037073023779613649229440981535",
      "side": "BUY",
      "outcomeIndex": 1,
      "title": "Rockets vs. Lakers",
      "slug": "nba-hou-lal-2026-04-21",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/super+cool+basketball+in+red+and+blue+wow.png",
      "eventSlug": "nba-hou-lal-2026-04-21",
      "outcome": "Lakers",
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
      "timestamp": 1776834768,
      "conditionId": "0x3ae6b8df561c6dccddf6283ed060232a8e8bcb42959d7e54b0482f945683ffc3",
      "type": "REDEEM",
      "size": 469.387752,
      "usdcSize": 469.387752,
      "transactionHash": "0xafda8cec8c9d9aa2b9faa344f153e7ba50e74d5ab86d32490691de727f237704",
      "price": 0,
      "asset": "",
      "side": "",
      "outcomeIndex": 999,
      "title": "Spread: Los Angeles Dodgers (-1.5)",
      "slug": "mlb-lad-sf-2026-04-21-spread-away-1pt5",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/Repetitive-markets/MLB.jpg",
      "eventSlug": "mlb-lad-sf-2026-04-21",
      "outcome": "",
      "name": "sovereign2013",
      "pseudonym": "Ultimate-Locality",
      "bio": "",
      "profileImage": "",
      "profileImageOptimized": ""
    }
    Oldest in this page:
    {
      "proxyWallet": "0xee613b3fc183ee44f9da9c05f53e2da107e3debf",
      "timestamp": 1776831168,
      "conditionId": "0xc6786942d5d1e3530c2b0de4f9f5f6875ba9a793838f1ac988e4d0cb9670917d",
      "type": "REDEEM",
      "size": 1002.56,
      "usdcSize": 1002.56,
      "transactionHash": "0xcd73d3058104bf355c4ad59a7c6bfb32ce1192e9b008868f7e30c183aa1bad63",
      "price": 0,
      "asset": "",
      "side": "",
      "outcomeIndex": 999,
      "title": "Bruins vs. Sabres",
      "slug": "nhl-bos-buf-2026-04-21",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/nhl.png",
      "eventSlug": "nhl-bos-buf-2026-04-21",
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
      "timestamp": 1776834838,
      "conditionId": "0x94bc8d28b9b30405f18df92537c4f905492714613d674d79dc4eaf7d0c4a40d6",
      "type": "TRADE",
      "size": 71.905,
      "usdcSize": 36.5,
      "transactionHash": "0x777c1945a1316a1fb75b13c6763ffb365aca7456e1b28b0647d104956dc2a1c1",
      "price": 0.5,
      "asset": "5434892888476756561552847544174287868850309604401844213343369938992528074490",
      "side": "BUY",
      "outcomeIndex": 0,
      "title": "Will FK Orenburg win on 2026-04-22?",
      "slug": "rus-ore-nov-2026-04-22-ore",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/russian-premier-league-93612b0ab7.png",
      "eventSlug": "rus-ore-nov-2026-04-22",
      "outcome": "Yes",
      "name": "swisstony",
      "pseudonym": "Frail-Possible",
      "bio": "loincloth chic",
      "profileImage": "",
      "profileImageOptimized": ""
    }
    Oldest in this page:
    {
      "proxyWallet": "0x204f72f35326db932158cba6adff0b9a1da95e14",
      "timestamp": 1776834512,
      "conditionId": "0x62dee78c2621684e9475a39d0f9181dfe24b5d13cc01c3201eb6a9287c8cc8dd",
      "type": "TRADE",
      "size": 49.97,
      "usdcSize": 49,
      "transactionHash": "0x2d7b280cb1fb976e093b93574724f55fadfa383f61e68ef8625608076145f8f1",
      "price": 0.98,
      "asset": "62235404601764175552254786179014005338468861457577445987184097407138409313584",
      "side": "BUY",
      "outcomeIndex": 1,
      "title": "Spread: Rockets (-4.5)",
      "slug": "nba-hou-lal-2026-04-21-spread-away-4pt5",
      "icon": "https://polymarket-upload.s3.us-east-2.amazonaws.com/super+cool+basketball+in+red+and+blue+wow.png",
      "eventSlug": "nba-hou-lal-2026-04-21",
      "outcome": "Lakers",
      "name": "swisstony",
      "pseudonym": "Frail-Possible",
      "bio": "loincloth chic",
      "profileImage": "",
      "profileImageOptimized": ""
    }


### Earliest Activity


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
MIN_WINDOW_SECONDS = 3600  # 1 hour floor — prevents infinite recursion
```


```python
def save_chunk(records, wallet_name):
    """
    Append a batch of records to the wallet's parquet file.
    Deduplicates using transactionHash when available, falling back
    to a composite key for events that have empty transactionHash
    (e.g. some SPLIT/MERGE events).
    """
    if not records:
        return

    def make_dedup_key(row):
        if row["transactionHash"]:
            return row["transactionHash"]
        return f"{row['timestamp']}_{row['conditionId']}_{row['type']}"

    path = f"{DATA_DIR}/activity_{wallet_name}.parquet"
    df   = pd.DataFrame(records)

    # Python 3.14 + pandas 2.x creates Arrow-backed string columns by default
    # fastparquet cannot write these — force all string columns to numpy object
    str_cols = df.select_dtypes(include="string").columns
    df[str_cols] = df[str_cols].astype(object)

    df["_dedup_key"] = df.apply(make_dedup_key, axis=1)

    if os.path.exists(path):
        existing = pd.read_parquet(path, engine="fastparquet")
        existing["_dedup_key"] = existing.apply(make_dedup_key, axis=1)
        combined = pd.concat([existing, df], ignore_index=True)
        combined = combined.drop_duplicates(subset=["_dedup_key"])
        combined = combined.drop(columns=["_dedup_key"])
        combined.to_parquet(path, index=False, engine="fastparquet")
    else:
        df = df.drop_duplicates(subset=["_dedup_key"])
        df = df.drop(columns=["_dedup_key"])
        df.to_parquet(path, index=False, engine="fastparquet")
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
    now_ts   = int(datetime.now(tz=timezone.utc).timestamp())  # fresh each wallet
    start_dt = datetime.fromtimestamp(WALLET_FIRST_TS[name], tz=timezone.utc).strftime("%Y-%m-%d")
    print(f"\n{'='*50}")
    print(f"Fetching {name} from {start_dt}")
    total = fetch_activity_full(address, name, WALLET_FIRST_TS[name], now_ts, ledger)
    print(f"→ {name}: +{total} new records this run")
```

    
    ==================================================
    Fetching rn1 from 2025-07-09
      skip 2025-07-09 (already fetched)
      skip 2025-07-16 (already fetched)
      skip 2025-07-23 (already fetched)
      skip 2025-07-30 (already fetched)
      skip 2025-08-06 (already fetched)
      skip 2025-08-13 (already fetched)
      skip 2025-08-20 (already fetched)
      skip 2025-08-27 (already fetched)
      skip 2025-09-03 (already fetched)
      skip 2025-09-10 (already fetched)
      skip 2025-09-17 (already fetched)
      skip 2025-09-24 (already fetched)
      skip 2025-10-01 (already fetched)
      skip 2025-10-08 (already fetched)
      skip 2025-10-15 (already fetched)
      skip 2025-10-22 (already fetched)
      skip 2025-10-29 (already fetched)
      skip 2025-11-05 (already fetched)
      skip 2025-11-12 (already fetched)
      skip 2025-11-19 (already fetched)
      skip 2025-11-26 (already fetched)
      skip 2025-12-03 (already fetched)
      skip 2025-12-10 (already fetched)
      skip 2025-12-17 (already fetched)
      skip 2025-12-24 (already fetched)
      skip 2025-12-31 (already fetched)
      skip 2026-01-07 (already fetched)
      skip 2026-01-14 (already fetched)
      skip 2026-01-21 (already fetched)
      skip 2026-01-28 (already fetched)
      skip 2026-02-04 (already fetched)
      skip 2026-02-11 (already fetched)
      skip 2026-02-18 (already fetched)
      skip 2026-02-25 (already fetched)
      skip 2026-03-04 (already fetched)
      skip 2026-03-11 (already fetched)
      skip 2026-03-18 (already fetched)
      skip 2026-03-25 (already fetched)
      skip 2026-04-01 (already fetched)
      skip 2026-04-08 (already fetched)
      skip 2026-04-15 (already fetched)
      2026-04-22 12:02 | overflow → bisecting into two 5h windows
        2026-04-22 12:02 | overflow → bisecting into two 2h windows
          2026-04-22 12:02 (+2h) | +2470 records
          2026-04-22 14:48 (+2h) | +2935 records
        2026-04-22 17:34 | overflow → bisecting into two 2h windows
          2026-04-22 17:34 | overflow → bisecting into two 1h windows
            2026-04-22 17:34 (+1h) | +2233 records
            2026-04-22 18:57 (+1h) | +2402 records
          2026-04-22 20:19 (+2h) | +2095 records
    → rn1: +12135 new records this run
    
    ==================================================
    Fetching sovereign from 2025-08-07
      skip 2025-08-07 (already fetched)
      skip 2025-08-14 (already fetched)
      skip 2025-08-21 (already fetched)
      skip 2025-08-28 (already fetched)
      skip 2025-09-04 (already fetched)
      skip 2025-09-11 (already fetched)
      skip 2025-09-18 (already fetched)
      skip 2025-09-25 (already fetched)
      skip 2025-10-02 (already fetched)
      skip 2025-10-09 (already fetched)
      skip 2025-10-16 (already fetched)
      skip 2025-10-23 (already fetched)
      skip 2025-10-30 (already fetched)
      skip 2025-11-06 (already fetched)
      skip 2025-11-13 (already fetched)
      skip 2025-11-20 (already fetched)
      skip 2025-11-27 (already fetched)
      skip 2025-12-04 (already fetched)
      skip 2025-12-11 (already fetched)
      skip 2025-12-18 (already fetched)
      skip 2025-12-25 (already fetched)
      skip 2026-01-01 (already fetched)
      skip 2026-01-08 (already fetched)
      skip 2026-01-15 (already fetched)
      skip 2026-01-22 (already fetched)
      skip 2026-01-29 (already fetched)
      skip 2026-02-05 (already fetched)
      skip 2026-02-12 (already fetched)
      skip 2026-02-19 (already fetched)
      skip 2026-02-26 (already fetched)
      skip 2026-03-05 (already fetched)
      skip 2026-03-12 (already fetched)
      skip 2026-03-19 (already fetched)
      skip 2026-03-26 (already fetched)
      skip 2026-04-02 (already fetched)
      skip 2026-04-09 (already fetched)
      2026-04-16 14:54 | overflow → bisecting into two 76h windows
        2026-04-16 14:54 | overflow → bisecting into two 38h windows
          2026-04-16 14:54 | overflow → bisecting into two 19h windows
            2026-04-16 14:54 | overflow → bisecting into two 9h windows
              2026-04-16 14:54 | overflow → bisecting into two 4h windows
                2026-04-16 14:54 (+4h) | +1700 records
                2026-04-16 19:40 (+4h) | +2711 records
              2026-04-17 00:25 (+9h) | +1806 records
            2026-04-17 09:56 (+19h) | +2615 records
          2026-04-18 04:57 | overflow → bisecting into two 19h windows
            2026-04-18 04:57 (+19h) | +2352 records
            2026-04-18 23:59 (+19h) | +2055 records
        2026-04-19 19:00 | overflow → bisecting into two 38h windows
          2026-04-19 19:00 | overflow → bisecting into two 19h windows
            2026-04-19 19:00 | overflow → bisecting into two 9h windows
              2026-04-19 19:00 (+9h) | +2809 records
              2026-04-20 04:31 (+9h) | +2005 records
            2026-04-20 14:02 | overflow → bisecting into two 9h windows
              2026-04-20 14:02 (+9h) | +2389 records
              2026-04-20 23:33 (+9h) | +2193 records
          2026-04-21 09:03 | overflow → bisecting into two 19h windows
            2026-04-21 09:03 | overflow → bisecting into two 9h windows
              2026-04-21 09:03 (+9h) | +2815 records
              2026-04-21 18:34 (+9h) | +714 records
            2026-04-22 04:05 (+19h) | +233 records
    → sovereign: +26395 new records this run
    
    ==================================================
    Fetching swisstony from 2025-08-09
      skip 2025-08-09 (already fetched)
      skip 2025-08-16 (already fetched)
      skip 2025-08-23 (already fetched)
      skip 2025-08-30 (already fetched)
      skip 2025-09-06 (already fetched)
      skip 2025-09-13 (already fetched)
      skip 2025-09-20 (already fetched)
      skip 2025-09-27 (already fetched)
      skip 2025-10-04 (already fetched)
      skip 2025-10-11 (already fetched)
      skip 2025-10-18 (already fetched)
      skip 2025-10-25 (already fetched)
      skip 2025-11-01 (already fetched)
      skip 2025-11-08 (already fetched)
      skip 2025-11-15 (already fetched)
      skip 2025-11-22 (already fetched)
      skip 2025-11-29 (already fetched)
      skip 2025-12-06 (already fetched)
      skip 2025-12-13 (already fetched)
      skip 2025-12-20 (already fetched)
      skip 2025-12-27 (already fetched)
      skip 2026-01-03 (already fetched)
      skip 2026-01-10 (already fetched)
      skip 2026-01-17 (already fetched)
      skip 2026-01-24 (already fetched)
      skip 2026-01-31 (already fetched)
      skip 2026-02-07 (already fetched)
      skip 2026-02-14 (already fetched)
      skip 2026-02-21 (already fetched)
      skip 2026-02-28 (already fetched)
      skip 2026-03-07 (already fetched)
      skip 2026-03-14 (already fetched)
      skip 2026-03-21 (already fetched)
      skip 2026-03-28 (already fetched)
      skip 2026-04-04 (already fetched)
      skip 2026-04-11 (already fetched)
      2026-04-18 05:09 | overflow → bisecting into two 57h windows
        2026-04-18 05:09 | overflow → bisecting into two 28h windows
          2026-04-18 05:09 | overflow → bisecting into two 14h windows
            2026-04-18 05:09 | overflow → bisecting into two 7h windows
              2026-04-18 05:09 | overflow → bisecting into two 3h windows
                2026-04-18 05:09 | overflow → bisecting into two 1h windows
                  2026-04-18 05:09 (+1h) | +2077 records
                  2026-04-18 06:55 (+1h) | +1476 records
                2026-04-18 08:42 (+3h) | +2501 records
              2026-04-18 12:16 | overflow → bisecting into two 3h windows
                2026-04-18 12:16 | overflow → bisecting into two 1h windows
                  2026-04-18 12:16 (+1h) | +3219 records
                  2026-04-18 14:03 | overflow → bisecting into two 0h windows
                    2026-04-18 14:03 (+0h) | +2032 records
                    2026-04-18 14:56 (+0h) | +1797 records
                2026-04-18 15:50 | overflow → bisecting into two 1h windows
                  2026-04-18 15:50 (+1h) | +2566 records
                  2026-04-18 17:37 (+1h) | +2691 records
            2026-04-18 19:24 | overflow → bisecting into two 7h windows
              2026-04-18 19:24 | overflow → bisecting into two 3h windows
                2026-04-18 19:24 | overflow → bisecting into two 1h windows
                  2026-04-18 19:24 (+1h) | +2255 records
                  2026-04-18 21:10 (+1h) | +2641 records
                2026-04-18 22:57 | overflow → bisecting into two 1h windows
                  2026-04-18 22:57 (+1h) | +2426 records
                  2026-04-19 00:44 (+1h) | +2112 records
              2026-04-19 02:31 | overflow → bisecting into two 3h windows
                2026-04-19 02:31 | overflow → bisecting into two 1h windows
                  2026-04-19 02:31 (+1h) | +2421 records
                  2026-04-19 04:18 (+1h) | +2196 records
                2026-04-19 06:05 (+3h) | +3157 records
          2026-04-19 09:39 | overflow → bisecting into two 14h windows
            2026-04-19 09:39 | overflow → bisecting into two 7h windows
              2026-04-19 09:39 | overflow → bisecting into two 3h windows
                2026-04-19 09:39 | overflow → bisecting into two 1h windows
                  2026-04-19 09:39 (+1h) | +918 records
                  2026-04-19 11:26 (+1h) | +2902 records
                2026-04-19 13:12 | overflow → bisecting into two 1h windows
                  2026-04-19 13:12 (+1h) | +2728 records
                  2026-04-19 14:59 | overflow → bisecting into two 0h windows
                    2026-04-19 14:59 (+0h) | +1746 records
                    2026-04-19 15:53 (+0h) | +1738 records
              2026-04-19 16:46 | overflow → bisecting into two 3h windows
                2026-04-19 16:46 | overflow → bisecting into two 1h windows
                  2026-04-19 16:46 (+1h) | +2546 records
                  2026-04-19 18:33 (+1h) | +1497 records
                2026-04-19 20:20 (+3h) | +3017 records
            2026-04-19 23:54 | overflow → bisecting into two 7h windows
              2026-04-19 23:54 | overflow → bisecting into two 3h windows
                2026-04-19 23:54 (+3h) | +2261 records
                2026-04-20 03:27 (+3h) | +2741 records
              2026-04-20 07:01 (+7h) | +1166 records
        2026-04-20 14:09 | overflow → bisecting into two 28h windows
          2026-04-20 14:09 | overflow → bisecting into two 14h windows
            2026-04-20 14:09 | overflow → bisecting into two 7h windows
              2026-04-20 14:09 (+7h) | +3255 records
              2026-04-20 21:16 (+7h) | +3319 records
            2026-04-21 04:24 | overflow → bisecting into two 7h windows
              2026-04-21 04:24 (+7h) | +2146 records
              2026-04-21 11:31 | overflow → bisecting into two 3h windows
                2026-04-21 11:31 (+3h) | +1676 records
                2026-04-21 15:05 (+3h) | +1950 records
          2026-04-21 18:39 | overflow → bisecting into two 14h windows
            2026-04-21 18:39 | overflow → bisecting into two 7h windows
              2026-04-21 18:39 | overflow → bisecting into two 3h windows
                2026-04-21 18:39 | overflow → bisecting into two 1h windows
                  2026-04-21 18:39 (+1h) | +2700 records
                  2026-04-21 20:26 (+1h) | +1350 records
                2026-04-21 22:12 (+3h) | +1929 records
              2026-04-22 01:46 (+7h) | +2967 records
            2026-04-22 08:54 | overflow → bisecting into two 7h windows
              2026-04-22 08:54 (+7h) | +2643 records
              2026-04-22 16:01 | overflow → bisecting into two 3h windows
                2026-04-22 16:01 | overflow → bisecting into two 1h windows
                  2026-04-22 16:01 (+1h) | +2341 records
                  2026-04-22 17:48 (+1h) | +3152 records
                2026-04-22 19:35 | overflow → bisecting into two 1h windows
                  2026-04-22 19:35 (+1h) | +2827 records
                  2026-04-22 21:22 (+1h) | +1605 records
    → swisstony: +92681 new records this run


## Data Validation 


```python
import os

print("=" * 50)
print("1. FILE SIZES")
print("=" * 50)
for name in WALLETS:
    path = f"{DATA_DIR}/activity_{name}.parquet"
    if os.path.exists(path):
        df      = pd.read_parquet(path, engine="fastparquet")
        size_mb = os.path.getsize(path) / 1024 / 1024
        print(f"{name}: {len(df):,} rows | {size_mb:.1f} MB")
    else:
        print(f"{name}: ⚠️  FILE NOT FOUND")

print()
print("=" * 50)
print("2. DATE RANGE COVERAGE")
print("=" * 50)
for name, address in WALLETS.items():
    path = f"{DATA_DIR}/activity_{name}.parquet"
    df   = pd.read_parquet(path, engine="fastparquet")
    df["datetime"] = pd.to_datetime(df["timestamp"], unit="s", utc=True)
    earliest = df["datetime"].min()
    latest   = df["datetime"].max()
    expected = datetime.fromtimestamp(WALLET_FIRST_TS[name], tz=timezone.utc)
    gap_days = (earliest - expected).days
    print(f"{name}:")
    print(f"  expected start : {expected.strftime('%Y-%m-%d')}")
    print(f"  actual start   : {earliest.strftime('%Y-%m-%d')}")
    print(f"  actual end     : {latest.strftime('%Y-%m-%d')}")
    if gap_days > 1:
        print(f"  ⚠️  gap of {gap_days} days at start — possible missing data")
    else:
        print(f"  ✓ start coverage looks good")

print()
print("=" * 50)
print("3. DUPLICATE CHECK")
print("=" * 50)
for name in WALLETS:
    path = f"{DATA_DIR}/activity_{name}.parquet"
    df   = pd.read_parquet(path, engine="fastparquet")

    # transactionHash duplicates
    tx_dupes = df[df["transactionHash"] != ""]["transactionHash"].duplicated().sum()

    # timestamp gaps > 7 days (potential missing windows)
    df_sorted = df.sort_values("timestamp")
    gaps      = df_sorted["timestamp"].diff()
    big_gaps  = gaps[gaps > 7 * 86400]

    print(f"{name}:")
    print(f"  duplicate transactionHashes : {tx_dupes}")
    if len(big_gaps) > 0:
        for idx, gap_sec in big_gaps.items():
            gap_ts = df_sorted.loc[idx, "timestamp"]
            gap_dt = datetime.fromtimestamp(gap_ts, tz=timezone.utc).strftime("%Y-%m-%d")
            print(f"  ⚠️  gap of {gap_sec/86400:.1f} days ending at {gap_dt}")
    else:
        print(f"  ✓ no gaps > 7 days in activity")

print()
print("=" * 50)
print("4. EVENT TYPE DISTRIBUTION")
print("=" * 50)
for name in WALLETS:
    path = f"{DATA_DIR}/activity_{name}.parquet"
    df   = pd.read_parquet(path, engine="fastparquet")
    dist = df["type"].value_counts()
    print(f"\n{name} — {len(df):,} total rows")
    print(dist.to_string())
```

    ==================================================
    1. FILE SIZES
    ==================================================
    rn1: 2,298,316 rows | 286.2 MB
    sovereign: 1,271,417 rows | 160.4 MB
    swisstony: 3,276,271 rows | 442.6 MB
    
    ==================================================
    2. DATE RANGE COVERAGE
    ==================================================
    rn1:
      expected start : 2025-07-09
      actual start   : 2025-07-09
      actual end     : 2026-04-22
      ✓ start coverage looks good
    sovereign:
      expected start : 2025-08-07
      actual start   : 2025-08-07
      actual end     : 2026-04-22
      ✓ start coverage looks good
    swisstony:
      expected start : 2025-08-09
      actual start   : 2025-08-09
      actual end     : 2026-04-22
      ✓ start coverage looks good
    
    ==================================================
    3. DUPLICATE CHECK
    ==================================================
    rn1:
      duplicate transactionHashes : 0
      ✓ no gaps > 7 days in activity
    sovereign:
      duplicate transactionHashes : 0
      ⚠️  gap of 19.0 days ending at 2025-09-09
    swisstony:
      duplicate transactionHashes : 0
      ✓ no gaps > 7 days in activity
    
    ==================================================
    4. EVENT TYPE DISTRIBUTION
    ==================================================
    
    rn1 — 2,298,316 total rows
    type
    TRADE           2287110
    REDEEM             6623
    MERGE              4457
    REWARD               83
    MAKER_REBATE         42
    CONVERSION            1
    
    sovereign — 1,271,417 total rows
    type
    TRADE           1194961
    REDEEM            40207
    MERGE             36126
    REWARD               74
    MAKER_REBATE         49
    
    swisstony — 3,276,271 total rows
    type
    TRADE           3132655
    REDEEM           141001
    MERGE              2481
    REWARD               84
    MAKER_REBATE         50


## Save Posistion Snapshot


```python
for name, address in WALLETS.items():
    data = get(DATA_API, "/positions", params={"user": address, "limit": 500})
    path = f"{DATA_DIR}/positions_{name}.csv"
    df   = pd.DataFrame(data)
    df["wallet"] = name
    df.to_csv(path, index=False)
    print(f"{name}: {len(df)} positions saved → {path}")
```

    rn1: 500 positions saved → data/positions_rn1.csv
    sovereign: 72 positions saved → data/positions_sovereign.csv
    swisstony: 500 positions saved → data/positions_swisstony.csv

