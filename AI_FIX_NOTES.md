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
from typing import Iterable, Hashable, Dict

def count_occurrences(items: Iterable[Hashable]) -> Dict[Hashable, int]:
    """
    Count how many times each value appears in `items`.

    Counter uses a hash map internally, giving O(n) time instead of O(n²).
    All elements must be hashable.

    Args:
        items: An iterable of hashable elements.

    Returns:
        A dictionary mapping each unique element to its occurrence count.

    Raises:
        TypeError: If `items` contains unhashable elements.
    """
    try:
        return dict(Counter(items))
    except TypeError as exc:
        raise TypeError("All elements in `items` must be hashable to be counted.") from exc
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
from sqlalchemy.orm import joinedload, Session
from typing import List

def get_users_with_orders(session: Session) -> List[User]:
    """
    Fetch all users and their associated orders in a single query.

    Uses eager loading via `joinedload` to avoid the N+1 query problem
    that occurs when accessing `user.orders` in a loop. For very large
    collections, consider `selectinload(User.orders)` to avoid row
    duplication from a JOIN.
    """
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
import copy
import requests
from functools import lru_cache
from urllib.parse import urljoin, quote
from typing import Any

BASE = "https://api.example.com"
MAX_CACHE_SIZE = 1024

# Reuse TCP connections across calls to reduce handshake overhead.
_SESSION = requests.Session()


def _sanitize_key(key: str) -> str:
    """
    Validate and URL-encode a path segment to prevent path traversal
    or injection in the final request URL.

    Args:
        key: The API identifier to fetch.

    Returns:
        A cleaned, URL-safe path segment.

    Raises:
        TypeError: If `key` is not a string.
        ValueError: If `key` is empty or contains only whitespace.
    """
    if not isinstance(key, str):
        raise TypeError(f"Expected str key, got {type(key).__name__}")
    key = key.strip().replace("\x00", "")
    if not key:
        raise ValueError("key must be a non-empty string")
    return quote(key, safe="")


@lru_cache(maxsize=MAX_CACHE_SIZE)
def fetch_data(key: str) -> Any:
    """
    Fetch JSON data for `key`, caching up to MAX_CACHE_SIZE entries.

    Args:
        key: A non-empty API identifier.

    Returns:
        Parsed JSON response. A deep copy is returned so callers cannot
        accidentally mutate the cached object.

    Raises:
        requests.RequestException: On network or HTTP error.
        TypeError / ValueError: On invalid input.
    """
    safe_key = _sanitize_key(key)
    url = urljoin(BASE + "/", safe_key)

    response = _SESSION.get(url, timeout=5)
    response.raise_for_status()

    # Return a copy so mutations by one caller don't poison the cache.
    return copy.deepcopy(response.json())
```

> For cross-request caching in a web app, swap `lru_cache` for Redis/memcached. Never cache user-specific or sensitive data in a process-wide cache unless it is intended to be shared.

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
import pandas as pd

def compute_total(df: pd.DataFrame) -> pd.DataFrame:
    """
    Add a `total` column computed as price * quantity.

    Uses vectorized arithmetic instead of iterating rows, which is
    orders of magnitude faster and avoids Python-level loop overhead.
    """
    required = {"price", "quantity"}
    missing = required - set(df.columns)
    if missing:
        raise ValueError(f"Input DataFrame is missing required columns: {missing}")

    # Work on a copy so callers are not surprised by side effects.
    result = df.copy()
    result["total"] = result["price"] * result["quantity"]
    return result
```

#### 6. Defer heavy imports until needed
File: `src/main.py`

Before:
```python
import heavy_ml_library  # imported at startup even for CLI help
```

After:
```python
# Defer the heavy import until the model is actually required.
# This keeps CLI help and startup paths fast.
_model = None

def load_model():
    """
    Lazily load the ML model and cache it for subsequent calls.
    """
    global _model
    if _model is None:
        import heavy_ml_library
        _model = heavy_ml_library.load("model.bin")
    return _model
```

---

### Follow-up actions

1. **Add a benchmark file** – create `tests/benchmark_main.py` using `pytest-benchmark` or `timeit` so every future PR can measure the hot path.
2. **Lock in the gain with CI** – add a GitHub Actions step that runs the benchmark and fails if the median latency regresses by more than 5–10%.
3. **Add runtime monitoring** – log slow function calls / DB query counts (e.g. `logging.warning` when `fetch_data` takes >1 s or a request triggers >10 DB queries).
4. **Profile again after each change** – rerun `cProfile` to confirm the bottleneck moved out of the top-20 list.
5. **Close the loop in the issue** – paste the before/after profiler numbers into issue #2 when the PR is opened.

> If you can paste the repo’s actual `src/` tree or the top 20 profiler lines, I can turn the placeholder file paths above into exact, project-specific edits.