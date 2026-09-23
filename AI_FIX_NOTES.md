# AI Fix — Issue #2: Performance: Identify and optimize bottlenecks

> **Issue body:**  
> This is an automated issue created by the AI agent to track planned code quality improvements. The AI will fix this in a subsequent run.

---

## Table of Contents

1. [Root cause](#root-cause)
2. [Exact code changes](#exact-code-changes)
   1. [Find the real bottleneck first](#1-find-the-real-bottleneck-first)
   2. [Replace O(n²) lookup loops with hash maps](#2-replace-on²-lookup-loops-with-hash-maps)
   3. [Eliminate N+1 database queries](#3-eliminate-n1-database-queries)
   4. [Cache expensive API / computation calls](#4-cache-expensive-api--computation-calls)
   5. [Vectorize pandas/numpy operations](#5-vectorize-pandasnumpy-operations)
   6. [Defer heavy imports until needed](#6-defer-heavy-imports-until-needed)
3. [Follow-up actions](#follow-up-actions)

---

## Root cause

Issue #2 is a generic “optimize bottlenecks” placeholder with no profiling data attached. In most small-to-medium projects the real root cause is one of these unmeasured hot paths:

1. **Algorithmic inefficiency** – nested loops or repeated scans of lists/dicts.
2. **Unbatched I/O** – N+1 database queries or one API call per item.
3. **No caching** – re-fetching or re-computing the same expensive result.
4. **Blocking startup / heavy imports** – modules loaded eagerly even when unused.
5. **Non-vectorized data processing** – row-by-row loops over pandas/numpy structures.

Because the repo contents aren’t shown here, the file paths below are **placeholders using a standard `src/` layout**. Map them to your actual files after profiling.

---

## Exact code changes

### 1. Find the real bottleneck first

File: any entrypoint (e.g. `src/main.py`, `src/app.py`, or `manage.py`)  
Run a deterministic profiler before changing code:

```bash
# Python example
python -m cProfile -o profile.stats -m src.main
python - <<'PY'
import pstats
p = pstats.Stats("profile.stats")
p.sort_stats("cumulative")
p.print_stats(20)
PY
```

Use the top 3 functions by `cumtime` as your target list.

---

### 2. Replace O(n²) lookup loops with hash maps

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
from collections.abc import Hashable, Iterable

def count_occurrences(items: Iterable[Hashable]) -> dict[Hashable, int]:
    """Return a frequency map for the items in ``items``.

    This implementation runs in O(n) time by delegating to
    ``collections.Counter`` instead of using a nested O(n²) loop.

    Args:
        items: An iterable of hashable elements.

    Returns:
        A dictionary mapping each distinct element to the number of
        times it appears.
    """
    return dict(Counter(items))
```

---

### 3. Eliminate N+1 database queries

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
from sqlalchemy import select
from sqlalchemy.orm import Session, selectinload

def get_users_with_orders(session: Session) -> list[User]:
    """Return every user with their orders eagerly loaded.

    ``selectinload`` loads the ``User.orders`` collection in a single
    follow-up query, eliminating the N+1 query problem that occurs
    when iterating over users and accessing ``user.orders``.

    Args:
        session: An active SQLAlchemy ORM session.

    Returns:
        A list of ``User`` instances with ``orders`` already populated.
    """
    stmt = select(User).options(selectinload(User.orders))
    return list(session.scalars(stmt))
```

---

### 4. Cache expensive API / computation calls

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
import logging
from functools import lru_cache

import requests

BASE = "https://api.example.com"
logger = logging.getLogger(__name__)

@lru_cache(maxsize=1024)
def fetch_data(key: str) -> dict:
    """Fetch JSON data from the remote API and cache it locally.

    The result is memoized in-process with a bounded LRU cache. For
    multi-process or cross-request caching, replace ``lru_cache`` with
    Redis, Memcached, or another shared cache backend.

    Args:
        key: The API resource identifier.

    Returns:
        The parsed JSON response.

    Raises:
        requests.HTTPError: If the API returns a non-2xx status code.
    """
    response = requests.get(f"{BASE}/{key}", timeout=5)
    response.raise_for_status()
    return response.json()
```

---

### 5. Vectorize pandas/numpy operations

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
    """Add a ``total`` column computed vectorized from price and quantity.

    Using whole-array arithmetic avoids the slow Python-level loop
    introduced by ``DataFrame.iterrows()`` and lets pandas dispatch to
    an efficient C implementation.

    Args:
        df: A DataFrame containing at least ``price`` and ``quantity``
            columns.

    Returns:
        The same DataFrame with a new ``total`` column.
    """
    df["total"] = df["price"] * df["quantity"]
    return df
```

---

### 6. Defer heavy imports until needed

File: `src/main.py`

Before:
```python
import heavy_ml_library  # imported at startup even for CLI help
```

After:
```python
def load_model() -> object:
    """Load the heavy ML model lazily on first use.

    Deferring the import keeps CLI help, unit tests, and other
    lightweight entry points fast.

    Returns:
        The loaded model object exposed by ``heavy_ml_library``.
    """
    import heavy_ml_library
    return heavy_ml_library.load("model.bin")
```

---

## Follow-up actions

1. **Add a benchmark file** – create `tests/benchmark_main.py` using `pytest-benchmark` so every future PR can measure the hot path.

   ```python
   # tests/benchmark_main.py
   """Benchmarks for the hot path identified in issue #2."""

   import pytest

   from src.main import hot_path  # replace with the actual entry point

   @pytest.mark.benchmark
   def test_hot_path_latency(benchmark):
       """Measure the median latency of the hot path.

       The benchmark runner reports min/median/max execution times.
       Store the median as a baseline and compare it in CI.
       """
       result = benchmark(hot_path)
       assert result is not None
   ```

2. **Lock in the gain with CI** – add a GitHub Actions step that runs the benchmark and fails if the median latency regresses by more than 5–10%.

   ```yaml
   - name: Run performance benchmarks
     run: pytest tests/benchmark_main.py --benchmark-only --benchmark-json=benchmark.json

   - name: Fail on regression
     run: python tests/check_regression.py benchmark.json --max-regression=0.10
   ```

3. **Add runtime monitoring** – log slow function calls and database query counts. Example helper:

   ```python
   import logging
   import time
   from functools import wraps
   from typing import Callable

   logger = logging.getLogger(__name__)

   def warn_if_slow(threshold_seconds: float = 1.0) -> Callable:
       """Decorator that warns when a call exceeds the given threshold.

       Args:
           threshold_seconds: Duration in seconds above which a warning
               is emitted.

       Returns:
           A decorator wrapping the target callable.
       """
       def decorator(func: Callable) -> Callable:
           @wraps(func)
           def wrapper(*args, **kwargs):
               start = time.perf_counter()
               result = func(*args, **kwargs)
               elapsed = time.perf_counter() - start
               if elapsed > threshold_seconds:
                   logger.warning(
                       "%s took %.3f s (threshold %.3f s)",
                       func.__name__, elapsed, threshold_seconds,
                   )
               return result
           return wrapper
       return decorator
   ```

4. **Profile again after each change** – rerun `cProfile` to confirm the bottleneck moved out of the top-20 list.

5. **Close the loop in the issue** – paste the before/after profiler numbers into issue #2 when the PR is opened.

> If you can paste the repo’s actual `src/` tree or the top 20 profiler lines, I can turn the placeholder file paths above into exact, project-specific edits.