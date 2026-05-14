# StringeeX Django Sync — Debugging Notes
**Date:** 2026-05-13  
**Project:** `stringeex_django`  
**Goal:** Flush today's data and resync correctly — expected ~8,000–9,600 records, only getting ~1,900–2,000

---

## Summary of Root Causes

| # | Issue | Source | Status |
|---|-------|--------|--------|
| 1 | Backfill used `get_or_create` — skipped existing records | Our code (`sync_calls.py`) | ✅ Fixed |
| 2 | Pagination stopped early — trusted broken `totalCount` from API | Our code (`stringeex_service.py`) | ✅ Fixed |
| 3 | `fetch_all_portals_for_range` was accidentally deleted | Our edit mistake | ✅ Restored |
| 4 | Duplicate `fetch_calls_for_range` defined twice in file | Our edit mistake | ✅ Fixed |
| 5 | Cross-portal dedup key was too aggressive (`account+start_time`) | Our code logic error | ✅ Reverted to `id` |
| 6 | API `page` parameter does nothing — always returns same 1,000 rows | **StringeeX API bug** | ⚠️ Unresolvable via API |
| 7 | API `start_time`/`end_time` filter parameters are completely ignored | **StringeeX API bug** | ⚠️ Unresolvable via API |

---

## Timeline of Debugging Steps

### Step 1 — Initial flush and backfill
Deleted today's records and ran backfill with `--start`/`--end`:
```bash
# Delete today's records
python manage.py shell -c "
from datetime import datetime
from zoneinfo import ZoneInfo
from calls.models import CallRecord
LOCAL_TZ = ZoneInfo('Asia/Kolkata')
now = datetime.now(tz=LOCAL_TZ)
start_ms = int(datetime(now.year, now.month, now.day, tzinfo=LOCAL_TZ).timestamp() * 1000)
deleted, _ = CallRecord.objects.filter(start_time_ms__gte=start_ms).delete()
print(f'Deleted {deleted} today records')
"

python manage.py sync_calls --start 2026-05-13 --end 2026-05-13
```
**Result:** `54000 fetched → 1900 new, 52100 skipped`  
**Finding:** Backfill uses `replace_today=False` (`get_or_create`) — skips anything already in DB.

---

### Step 2 — Fixed backfill to upsert today
Changed `_backfill()` in `sync_calls.py` to detect today and pass `replace_today=True`:
```python
is_today = current.date() == datetime.now(tz=LOCAL_TZ).date()
new, skip = upsert_records(rows, replace_today=is_today)
```
**Result:** Still only ~2,000 fetched — problem moved upstream to the API fetch layer.

---

### Step 3 — Discovered API totalCount bug
Tested what the API returns for `totalCount`:
```python
params = {'limit': 1000, 'page': 1, 'start_time': start_ms, 'end_time': end_ms}
resp = requests.get(url, headers=headers, params=params, timeout=20)
total = resp.json()['data']['totalCount']
# totalCount from API: 270394  ← all-time total, not today's
```
**Finding:** `totalCount` is the all-time record count across all dates, not filtered by date.  
**Fix:** Changed pagination stop condition from `len(all_rows) >= total` to `len(rows) < page_size`.

---

### Step 4 — Discovered API page looping
Manually paginated bluewave3 — it returned 37 pages of 1,000 rows when only ~5,600 were expected:
```
bluewave3 page 34: 1000 rows  ← impossible, only ~5633 exist
bluewave3 page 35: 1000 rows
bluewave3 page 37: 1000 rows  ← API is looping
```
**Finding:** After exhausting real records, the API loops and repeats earlier pages infinitely.  
**Fix:** Added `seen_ids` set per portal — stop when a page returns only already-seen IDs.

---

### Step 5 — Accidental deletion of `fetch_all_portals_for_range`
During edits to add `seen_ids` logic, `fetch_all_portals_for_range` was accidentally removed.  
**Symptom:** `ImportError: cannot import name 'fetch_all_portals_for_range'`  
**Fix:** Restored the function. Also removed duplicate `fetch_calls_for_range` definition.

---

### Step 6 — Cross-portal dedup key too aggressive
Changed dedup key from `id` to `(account_name, start_time_ms)` thinking portals share IDs.  
**Result:** Dropped legitimate records — bluewave3 page 1 matched bluewave2 records and stopped at 1,922 fetched.  
**Finding:** `start_time` varies slightly even for the same call across portals (ms-level drift). Key was wrong.  
**Fix:** Reverted to `id`-based cross-portal dedup. Call IDs are globally unique per call.

---

### Step 7 — Confirmed API ignores time filter params entirely
Passed a date range from year 2001 — API still returned today's records:
```python
params = {'limit': 1000, 'page': 1, 'start_time': 1000000000000, 'end_time': 1000000001000}
rows = resp.json()['data']['rows']
# Rows for year 2001 date range: 1000
# First row start_time: 1778672956053  ← today's timestamp
```
**Finding:** `start_time` and `end_time` query params are completely ignored by the API.

---

### Step 8 — Confirmed page parameter does nothing
Pages 1, 2, 3 all return identical records:
```
Page 1: 1000 rows | min=1778669417343 | max=1778673250998
Page 2: 1000 rows | min=1778669417343 | max=1778673250998  ← identical
Page 3: 1000 rows | min=1778669417343 | max=1778673250998  ← identical
```
**Finding:** The API `page` parameter also does nothing. It always returns the same latest 1,000 records.  
**Conclusion:** The StringeeX API is fundamentally broken for historical/date-filtered fetching.

---

## What Was Our Code's Fault vs API's Fault

### Our Code Errors (fixable, now fixed)
1. `_backfill()` used `replace_today=False` → skipped re-syncing today's records
2. Pagination trusted `totalCount` (270k) as stop condition → over-paginated or stopped wrong
3. `fetch_all_portals_for_range` was deleted during an edit
4. `fetch_calls_for_range` was defined twice — Python used the second broken definition
5. Cross-portal dedup used `(account_name, start_time_ms)` — too aggressive, dropped real records

### API Errors (not our fault, unresolvable via API calls)
1. `start_time` / `end_time` filter parameters are completely ignored
2. `page` parameter does nothing — always returns same 1,000 records
3. `totalCount` returns all-time count, not date-filtered count
4. After real records exhausted, API loops pages instead of returning empty

---

## Current State of `stringeex_service.py`

The file is correct and clean. Key behaviors:
- `fetch_calls_for_range`: paginates with `seen_ids` per portal to detect API looping
- `fetch_all_portals_for_range`: merges portals with `id`-based cross-portal dedup
- No duplicate function definitions
- No time filter params sent (they do nothing anyway)

**Limitation:** Due to API bugs, only the latest ~1,000 records are ever retrievable via API.  
The daemon (polling every 4 min) works correctly for live tracking within this limit.

---

## Recommended Solution for Full Historical Data

Since the API cannot return more than 1,000 records regardless of parameters, use **CSV export**:

1. Export today's calls from StringeeX dashboard (both portals)
2. Import via Django shell:
```python
import pandas as pd
from calls.management.commands.sync_calls import upsert_records

df = pd.read_csv('/path/to/today_export.csv', low_memory=False)
rows = df.to_dict(orient='records')
saved, skipped = upsert_records(rows, replace_today=True)
print(f'Saved: {saved}, Skipped: {skipped}')
```

---

## Useful Debug Commands

### Check today's record count in DB
```bash
python manage.py shell -c "
from datetime import datetime
from zoneinfo import ZoneInfo
from calls.models import CallRecord
LOCAL_TZ = ZoneInfo('Asia/Kolkata')
now = datetime.now(tz=LOCAL_TZ)
start_ms = int(datetime(now.year, now.month, now.day, tzinfo=LOCAL_TZ).timestamp() * 1000)
print(CallRecord.objects.filter(start_time_ms__gte=start_ms).count())
"
```

### Check records per day (last 3 days)
```bash
python manage.py shell -c "
from datetime import datetime, timedelta
from zoneinfo import ZoneInfo
from calls.models import CallRecord
LOCAL_TZ = ZoneInfo('Asia/Kolkata')
now = datetime.now(tz=LOCAL_TZ)
for i in range(3):
    day = now - timedelta(days=i)
    s = int(datetime(day.year, day.month, day.day, tzinfo=LOCAL_TZ).timestamp() * 1000)
    e = int(datetime(day.year, day.month, day.day, 23, 59, 59, tzinfo=LOCAL_TZ).timestamp() * 1000)
    print(f'{day.strftime(\"%Y-%m-%d\")}: {CallRecord.objects.filter(start_time_ms__gte=s, start_time_ms__lte=e).count()}')
"
```

### Test if API time filters work
```bash
python manage.py shell -c "
from calls.stringeex_service import get_auth_token_for_portal
import requests
token, base_url = get_auth_token_for_portal('bluewave2')
url = f'{base_url}/v1/call/history'
headers = {'X-STRINGEE-AUTH': token}
# Pass year 2001 — if returns today's records, API ignores filters
params = {'limit': 5, 'page': 1, 'start_time': 1000000000000, 'end_time': 1000000001000}
rows = requests.get(url, headers=headers, params=params, timeout=20).json()['data']['rows']
print(f'Rows: {len(rows)}, first start_time: {rows[0][\"start_time\"] if rows else None}')
"
```

### Check if API pagination works (pages should differ)
```bash
python manage.py shell -c "
from calls.stringeex_service import get_auth_token_for_portal
import requests
token, base_url = get_auth_token_for_portal('bluewave2')
url = f'{base_url}/v1/call/history'
headers = {'X-STRINGEE-AUTH': token}
from datetime import datetime
from zoneinfo import ZoneInfo
LOCAL_TZ = ZoneInfo('Asia/Kolkata')
now = datetime.now(tz=LOCAL_TZ)
today_ms = int(datetime(now.year, now.month, now.day, tzinfo=LOCAL_TZ).timestamp() * 1000)
for page in [1, 2, 3]:
    params = {'limit': 1000, 'page': page}
    rows = requests.get(url, headers=headers, params=params, timeout=20).json()['data']['rows']
    times = [r.get('start_time') for r in rows]
    print(f'Page {page}: {len(rows)} rows | today={sum(1 for t in times if t and t >= today_ms)} | min={min(t for t in times if t)} | max={max(t for t in times if t)}')
"
```

### Manually paginate a portal to find true unique count
```bash
python manage.py shell -c "
from datetime import datetime
from zoneinfo import ZoneInfo
from calls.stringeex_service import get_auth_token_for_portal
import requests
LOCAL_TZ = ZoneInfo('Asia/Kolkata')
now = datetime.now(tz=LOCAL_TZ)
today_ms = int(datetime(now.year, now.month, now.day, tzinfo=LOCAL_TZ).timestamp() * 1000)
for portal in ['bluewave2', 'bluewave3']:
    token, base_url = get_auth_token_for_portal(portal)
    url = f'{base_url}/v1/call/history'
    headers = {'X-STRINGEE-AUTH': token}
    seen = set()
    page = 1
    while True:
        rows = requests.get(url, headers=headers, params={'limit': 1000, 'page': page}, timeout=20).json()['data']['rows']
        ids = [str(r.get('id','')) for r in rows]
        new = [i for i in ids if i not in seen]
        seen.update(ids)
        print(f'  {portal} page {page}: {len(rows)} rows, {len(new)} new')
        if not new or len(rows) < 1000:
            break
        page += 1
    print(f'{portal} TOTAL unique: {len(seen)}')
"
```
