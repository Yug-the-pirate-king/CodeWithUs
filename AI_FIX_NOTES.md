# AI Fix — Issue #2: Performance: Identify and optimize bottlenecks

**Issue body:**

This is an automated issue created by the AI agent to track planned code quality improvements. The AI will fix this in a subsequent run.

---

## AI-proposed fix

### Overview

Issue #2 is a generic placeholder asking to "optimize bottlenecks." Before any code is changed, profile the application so the work targets the real hot path. This document lists the most common performance anti-patterns, shows how to refactor them safely, and adds docstrings, input validation, and error handling to the resulting functions.

### Root cause

In most small-to-medium projects, generic performance issues come from one or more unmeasured hot paths:

1. **Algorithmic inefficiency** – nested loops or repeated scans of lists/dicts.
2. **Unbatched I/O** – N+1 database queries or one API call per item.
3. **No caching** – re-fetching or re-computing the same expensive result.
4. **Blocking startup / heavy imports** – modules loaded eagerly even when unused.
5. **Non-vectorized data processing** – row-by-row loops over pandas/numpy structures.

Because the repo contents are not shown here, the file paths below are **placeholders using a standard `src/` layout**. Map them to your actual files after profiling.

### Reusable validation helpers

Repeated validation patterns are centralized in `src/performance_utils.py` so every optimized module can reuse them. Import the helpers below in the refactored modules, or keep the checks inline if you prefer a single-file change.

```python
"""Shared input-validation helpers used by the performance fixes."""

from typing import Any, Iterable, TypeVar

T = TypeVar("T")


def require_not_none(value: Any, name: str) -> Any:
    """Return `value` if it is not None, otherwise raise a TypeError.

    Args:
        value: Any object to validate.
        name: Human-readable name used in error messages.

    Returns:
        The same `value` reference.

    Raises:
        TypeError: If `value` is None.
    """
    if value is None:
        raise TypeError(f"{name} must not be None")
    return value


def require_iterable(items: Iterable[T], name: str = "items") -> Iterable[T]:
    """Return `items` after validating it is iterable.

    Args:
        items: Any iterable object.
        name: Human-readable name used in error messages.

    Returns:
        The same `items` reference.

    Raises:
        TypeError: If `items` is None or not iterable.
    """
    require_not_none(items, name)
    if not hasattr(items, "__iter__"):
        raise TypeError(f"{name} must be iterable")
    return items


def require_positive_int(value: int, name: str) -> int:
    """Return `value` after validating it is a positive integer.

    Args:
        value: Integer to validate.
        name: Human-readable name used in error messages.

    Returns:
        The same integer value.

    Raises:
        ValueError: If `value` is not a positive integer.
    """
    if not isinstance(value, int) or value <= 0:
        raise ValueError(f"{name} must be a positive integer, got {value!r}")
    return value
```

For pandas-specific modules you can add:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    import pandas as pd


def require_dataframe_columns(df: "pd.DataFrame", *columns: str) -> None:
    """Raise ValueError if a DataFrame is missing any required column.

    Args:
        df: A pandas DataFrame.
        *columns: Column names that must be present.

    Raises:
        TypeError: If `df` is not a pandas DataFrame.
        ValueError: If any required column is missing.
    """
    if df is None or not hasattr(df, "columns"):
        raise TypeError("df must be a pandas DataFrame")
    missing = [c for c in columns if c not in df.columns]
    if missing:
        raise ValueError(f"DataFrame missing required columns: {missing}")
```

---

### 1. Find the real bottleneck first

File: any entrypoint (e.g. `src/main.py`, `src/app.py`, or `manage.py`)

Run a deterministic profiler before changing code. You can either run the shell commands directly or use the reusable helper below to keep profiling consistent across entrypoints.

```bash
# Python example
python -m cProfile -o profile.stats -m src.main
python -c "import pstats; p = pstats.Stats('profile.stats'); p.sort_stats('cumulative'); p.print_stats(20)"
```

```python
"""Reusable profiling helper."""

import cProfile
import pstats
import runpy
from pathlib import Path
from typing import Union


def profile_entrypoint(
    entrypoint: str,
    output: Union[Path, str] = Path("profile.stats"),
    top_n: int = 20,
) -> None:
    """Profile an entrypoint module and print the top cumulative-time functions.

    Args:
        entrypoint: Dotted module path to run, e.g. "src.main".
        output: File path where the raw profile data is saved.
        top_n: Number of functions to print from the sorted profile.

    Raises:
        ValueError: If `entrypoint` is empty or `top_n` is not positive.
    """
    if not entrypoint or not isinstance(entrypoint, str):
        raise ValueError("entrypoint must be a non-empty string")

    require_positive_int(top_n, "top_n")

    output_path = Path(output)
    profiler = cProfile.Profile()
    try:
        profiler.enable()
        runpy.run_module(entrypoint, run_name="__main__")
    finally:
        profiler.disable()
        profiler.dump_stats(str(output_path))

    stats = pstats.Stats(str(output_path))
    stats.sort_stats("cumulative")
    stats.print_stats(top_n)
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
from typing import Dict, Iterable, TypeVar

T = TypeVar("T")


def count_occurrences(items: Iterable[T]) -> Dict[T, int]:
    """Return a mapping of each distinct item to its occurrence count.

    This is an O(n) replacement for the original nested-loop O(n²) implementation.

    Args:
        items: A finite iterable of hashable objects.

    Returns:
        A dictionary mapping each unique item to its count.

    Raises:
        TypeError: If `items` is None, not iterable, or contains unhashable elements.
    """
    items = require_iterable(items, "items")

    try:
        return dict(Counter(items))
    except TypeError as exc:
        raise TypeError("all items must be hashable") from exc
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
from typing import Any, List

from sqlalchemy.orm import joinedload


def get_users_with_orders(session) -> List[Any]:
    """Fetch all users with their orders loaded in a single query.

    Uses SQLAlchemy's `joinedload` to eager-load the related `orders`,
    eliminating the N+1 query problem when iterating over users.

    Args:
        session: A valid SQLAlchemy ORM session.

    Returns:
        A list of `User` instances with `orders` preloaded.

    Raises:
        TypeError: If `session` is missing the required `query` method.
        RuntimeError: If the database query fails.
    """
    require_not_none(session, "session")
    if not hasattr(session, "query"):
        raise TypeError("session must be a valid SQLAlchemy session")

    try:
        return session.query(User).options(joinedload(User.orders)).all()
    except Exception as exc:
        raise RuntimeError("failed to load users with orders") from exc
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
from typing import Any

import requests

BASE = "https://api.example.com"
_LOGGER = logging.getLogger(__name__)


@lru_cache(maxsize=1024)
def fetch_data(key: str) -> Any:
    """Fetch JSON data for `key` with bounded in-memory caching and a timeout.

    Args:
        key: Identifier used to build the request URL.

    Returns:
        Parsed JSON response body.

    Raises:
        ValueError: If `key` is not a non-empty string.
        requests.exceptions.RequestException: If the HTTP request fails.
    """
    if not isinstance(key, str) or not key:
        raise ValueError("key must be a non-empty string")

    url = f"{BASE}/{key}"
    try:
        response = requests.get(url, timeout=5)
        response.raise_for_status()
        return response.json()
    except requests.exceptions.Timeout:
        _LOGGER.warning("Request timed out for %s", url)
        raise
    except requests.exceptions.HTTPError as exc:
        _LOGGER.warning("HTTP error for %s: %s", url, exc)
        raise
```

> For cross-request caching in a web app, swap `lru_cache` for Redis/memcached.

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
    """Add a `total` column to `df` using vectorized multiplication.

    This replaces the slower `iterrows()` loop with a single pandas
    vectorized operation.

    Args:
        df: A pandas DataFrame containing numeric `price` and `quantity` columns.

    Returns:
        The same DataFrame with a new `total` column (`price * quantity`).

    Raises:
        TypeError: If `df` is not a pandas DataFrame or the price/quantity columns are not numeric.
        ValueError: If required columns are missing.
    """
    if df is None or not hasattr(df, "columns"):
        raise TypeError("df must be a pandas DataFrame")

    required = ("price", "quantity")
    missing = [c for c in required if c not in df.columns]
    if missing:
        raise ValueError(f"DataFrame missing required columns: {missing}")

    for col in required:
        if not pd.api.types.is_numeric_dtype(df[col]):
            raise TypeError(f"column '{col}' must be numeric")

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
from pathlib import Path
from typing import Union

DEFAULT_MODEL_PATH = Path("model.bin")


def load_model(model_path: Union[str, Path] = DEFAULT_MODEL_PATH):
    """Import the heavy ML library lazily and load a model from disk.

    This keeps startup fast for commands that do not need ML inference.

    Args:
        model_path: Path to the serialized model file.

    Returns:
        The loaded model object returned by the heavy library.

    Raises:
        FileNotFoundError: If the model file does not exist.
        RuntimeError: If the model cannot be loaded.
    """
    model_path = Path(model_path)
    if not model_path.exists():
        raise FileNotFoundError(f"model file not found: {model_path}")

    try:
        import heavy_ml_library
        return heavy_ml_library.load(str(model_path))
    except Exception as exc:
        raise RuntimeError(f"failed to load model from {model_path}") from exc
```

---

## Follow-up actions

1. **Add a benchmark file** – create `tests/benchmark_main.py` using `pytest-benchmark` or `timeit` so every future PR can measure the hot path.
2. **Lock in the gain with CI** – add a GitHub Actions step that runs the benchmark and fails if the median latency regresses by more than 5–10%.
3. **Add runtime monitoring** – log slow function calls / DB query counts (e.g. `logging.warning` when `fetch_data` takes >1 s or a request triggers >10 DB queries).
4. **Profile again after each change** – rerun `cProfile` to confirm the bottleneck moved out of the top-20 list.
5. **Close the loop in the issue** – paste the before/after profiler numbers into issue #2 when the PR is opened.

> If you can paste the repo's actual `src/` tree or the top 20 profiler lines, the placeholder file paths above can be replaced with exact, project-specific edits.