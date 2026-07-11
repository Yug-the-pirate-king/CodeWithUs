# AI Fix — Issue #2: Performance: Identify and optimize bottlenecks

**Issue body:**

This is an automated issue created by the AI agent to track planned code quality improvements. The AI will fix this in a subsequent run.

---

**AI-proposed fix:**

### Root cause
Issue #2 is a generic “optimize bottlenecks” placeholder with no profiling data attached. In most small-to-medium projects the real root cause is one of these unmeasured hot paths:

1. **Algorithmic inefficiency** – nested loops or repeated scans of lists/dicts.
2. **Unbatched I/O** – N+1 database queries or one API call per item.
3. **No caching** – re-fetching or re-computing the same expensive result.
4. **Blocking startup / heavy imports** – modules loaded eagerly even when unused.
5. **Non-vectorized data processing** – row-by-row loops over pandas/numpy structures.

Because the repo contents aren’t shown here, the file paths below are **placeholders using a standard `src/` layout**. Map them to your actual files after profiling.

---

### Exact code changes needed

#### 1. Find the real bottleneck first
File: any entrypoint (e.g. `src/main.py`, `src/app.py`, or `manage.py`)  
Run a deterministic profiler before changing code:

```bash
# Python example
python -m cProfile -o profile.stats -m src.main
python -c "import pstats; p = pstats.Stats('profile.stats'); p.sort_stats('cumulative'); p.print_stats(20)"
```

Use the top 3 functions by `cumtime` as your target list.

#### 2. Replace O(n²) lookup loops with hash maps
File: `src/processing.py`

Before:
```python
def count_occurrences(items):
    result = {}
    for i in items:
        count = 0
        for j in items:
            if i == j:
                count += 1
        result[i] = count
    return result
```

After:
```python
from collections import Counter

def count_occurrences(items):
    return dict(Counter(items))
```

#### 3. Eliminate N+1 database queries
File: `src/db.py` or `src/models.py`

Before:
```python
def get_users_with_orders(session):
    users = session.query(User).all()
    for user in users:
        print(len(user.orders))   # triggers a query per user
    return users
```

After:
```python
from sqlalchemy.orm import joinedload

def get_users_with_orders(session):
    return (
        session.query(User)
        .options(joinedload(User.orders))
        .all()
    )
```

#### 4. Cache expensive API / computation calls
File: `src/api_client.py`

Before:
```python
import requests

BASE = "https://api.example.com"

def fetch_data(key):
    return requests.get(f"{BASE}/{key}").json()
```

After:
```python
import requests
from functools import lru_cache

BASE = "https://api.example.com"

@lru_cache(maxsize=1024)
def fetch_data(key):
    return requests.get(f"{BASE}/{key}", timeout=5).json()
```

> For cross-request caching in a web app, swap `lru_cache` for Redis/memcached.

#### 5. Vectorize pandas/numpy operations
File: `src/data_processing.py`

Before:
```python
def compute_total(df):
    totals = []
    for _, row in df.iterrows():
        totals.append(row["price"] * row["quantity"])
    df["total"] = totals
    return df
```

After:
```python
def compute_total(df):
    df["total"] = df["price"] * df["quantity"]
    return df
```

#### 6. Defer heavy imports until needed
File: `src/main.py`

Before:
```python
import heavy_ml_library  # imported at startup even for CLI help
```

After:
```python
def load_model():
    import heavy_ml_library
    return heavy_ml_library.load("model.bin")
```

---

### Follow-up actions

1. **Add a benchmark file** – create `tests/benchmark_main.py` using `pytest-benchmark` or `timeit` so every future PR can measure the hot path.
2. **Lock in the gain with CI** – add a GitHub Actions step that runs the benchmark and fails if the median latency regresses by more than 5–10%.
3. **Add runtime monitoring** – log slow function calls / DB query counts (e.g. `logging.warning` when `fetch_data` takes >1 s or a request triggers >10 DB queries).
4. **Profile again after each change** – rerun `cProfile` to confirm the bottleneck moved out of the top-20 list.
5. **Close the loop in the issue** – paste the before/after profiler numbers into issue #2 when the PR is opened.

> If you can paste the repo’s actual `src/` tree or the top 20 profiler lines, I can turn the placeholder file paths above into exact, project-specific edits.
