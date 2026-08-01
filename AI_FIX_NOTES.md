# AI Fix — Issue #7: Performance: Identify and optimize bottlenecks

**Issue body:**

This is an automated issue created by the AI agent to track planned code quality improvements. The AI will fix this in a subsequent run.

---

**AI-proposed fix:**

I can’t pinpoint the exact bottleneck without access to the repo code or a CPU/memory/profile snapshot. Issue #7 is a broad tracking issue, so the “root cause” right now is **lack of a measured baseline plus one or more common hotspots** (e.g., unbounded loops/list rendering, N+1 DB queries, missing caching/memoization, or large synchronous work on the main thread).

Below is an actionable, **diagnose-first** plan, plus the exact code changes you should apply once you map them to your real file structure. I’ve included placeholder file paths; replace them with the actual paths in `Yug-the-pirate-king/CodeWithUs`.

---

## 1. First, get evidence

### Profiling steps
| Layer | Tool | What to look for |
|---|---|---|
| Frontend | Chrome DevTools → Performance tab / Lighthouse / React Profiler | Long scripting tasks, forced reflow, excessive re-renders |
| Backend (Node) | `node --prof`, [Clinic.js](https://clinicjs.org/), or `perf` | Functions consuming the most CPU, event-loop lag |
| Database | Query log / `EXPLAIN ANALYZE` / ORM slow-query log | Missing indexes, N+1 queries, full-table scans |
| API | `autocannon`, `k6`, or Postman runner | High latency endpoints, high memory usage under load |

Add a dedicated profiling script so this is reproducible:

```jsonc
// package.json  (or wherever your npm scripts live)
{
  "scripts": {
    "profile:backend": "node --prof src/server.js",
    "profile:flame": "0x src/server.js",
    "load:test": "autocannon -c 50 -d 10 http://localhost:3000/api/challenges"
  }
}
```

---

## 2. Likely high-impact code changes

### A. Paginate and project list endpoints
Unbounded `SELECT *` queries are the #1 backend bottleneck for “list” screens.

```diff
// <backend>/src/routes/challenges.js  (or equivalent list route)
- app.get('/api/challenges', async (req, res) => {
-   const rows = await db.query('SELECT * FROM challenges ORDER BY created_at DESC');
-   res.json(rows);
- });

+ app.get('/api/challenges', async (req, res) => {
+   const page = Math.max(1, parseInt(req.query.page, 10) || 1);
+   const limit = Math.min(100, parseInt(req.query.limit, 10) || 20);
+   const offset = (page - 1) * limit;
+
+   // Only select columns the UI needs
+   const rows = await db.query(
+     `SELECT id, title, difficulty, tags, created_at
+      FROM challenges
+      ORDER BY created_at DESC
+      LIMIT $1 OFFSET $2`,
+     [limit, offset]
+   );
+
+   res.json({ page, limit, data: rows });
+ });
```

### B. Add database indexes for common filters
If you filter by `difficulty`, `tags`, `user_id`, or `created_at`, add indexes.

```diff
// <backend>/migrations/002_add_challenge_indexes.sql
+ CREATE INDEX idx_challenges_difficulty ON challenges(difficulty);
+ CREATE INDEX idx_challenges_created_at ON challenges(created_at DESC);
+ CREATE INDEX idx_submissions_user_id ON submissions(user_id);
```

### C. Memoize expensive React components
If a list re-renders on every keystroke or state change, wrap it.

```diff
// <frontend>/src/components/ChallengeList.jsx
- function ChallengeList({ challenges, filter }) {
+ const ChallengeList = React.memo(function ChallengeList({ challenges, filter }) {
    const visible = useMemo(
      () => challenges.filter(c => c.title.toLowerCase().includes(filter.toLowerCase())),
      [challenges, filter]
    );

    return (
      <ul>
-       {visible.map(c => <ChallengeCard key={c.id} challenge={c} />)}
+       {visible.map(c => <ChallengeCard key={c.id} challenge={c} />)}
      </ul>
    );
- }
+ });
```

### D. Virtualize very long lists
If a list can contain hundreds+ of items, use a windowing library.

```bash
npm install react-window
```

```diff
// <frontend>/src/components/ChallengeList.jsx
+ import { FixedSizeList as List } from 'react-window';

  const Row = ({ index, style, data }) => (
    <div style={style}>
      <ChallengeCard challenge={data[index]} />
    </div>
  );

  function ChallengeList({ challenges }) {
-   return <ul>{challenges.map(...)}</ul>;
+   return (
+     <List
+       height={600}
+       itemCount={challenges.length}
+       itemSize={80}
+       itemData={challenges}
+     >
+       {Row}
+     </List>
+   );
  }
```

### E. Debounce search input
Typing fires too many re-renders/filter calls without debouncing.

```diff
// <frontend>/src/hooks/useDebounce.js
+ import { useState, useEffect } from 'react';
+
+ export function useDebounce(value, delay = 300) {
+   const [debounced, setDebounced] = useState(value);
+   useEffect(() => {
+     const t = setTimeout(() => setDebounced(value), delay);
+     return () => clearTimeout(t);
+   }, [value, delay]);
+   return debounced;
+ }

// <frontend>/src/components/SearchBar.jsx
+ import { useDebounce } from '../hooks/useDebounce';
+
  function SearchBar({ onSearch }) {
    const [term, setTerm] = useState('');
+   const debouncedTerm = useDebounce(term, 300);
+
+   useEffect(() => {
+     onSearch(debouncedTerm);
+   }, [debouncedTerm, onSearch]);
+
    return (
      <input
        value={term}
-       onChange={e => onSearch(e.target.value)}
+       onChange={e => setTerm(e.target.value)}
      />
    );
  }
```

### F. Cache read-heavy endpoints
If challenge metadata rarely changes, cache it for a few minutes.

```diff
// <backend>/src/middleware/cache.js
+ const cache = new Map();
+
+ export function memoize(handler, ttlMs = 60_000) {
+   return async (req, res) => {
+     const key = req.originalUrl;
+     const hit = cache.get(key);
+     if (hit && Date.now() - hit.ts < ttlMs) {
+       return res.json(hit.body);
+     }
+     res.json = (function (original) {
+       return function (body) {
+         cache.set(key, { body, ts: Date.now() });
+         return original.call(res, body);
+       };
+     })(res.json);
+     return handler(req, res);
+   };
+ }

// <backend>/src/routes/challenges.js
+ import { memoize } from '../middleware/cache.js';
+
- app.get('/api/challenges', listChallenges);
+ app.get('/api/challenges', memoize(listChallenges, 120_000));
```

For multi-instance deployments, replace the in-memory `Map` with Redis.

### G. Replace nested loops with hash maps
If you compare arrays/objects with O(n²) loops, rewrite them.

```diff
// <backend>/src/utils/matcher.js  (or any matching logic)
- const matches = users.filter(u => challenges.some(c => c.id === u.lastChallengeId));
+ const challengeIds = new Set(challenges.map(c => c.id));
+ const matches = users.filter(u => challengeIds.has(u.lastChallengeId));
```

### H. Code-split and lazy-load heavy routes
If the bundle is large, split by route.

```diff
// <frontend>/src/App.jsx
- import ChallengePage from './pages/ChallengePage';
+ const ChallengePage = lazy(() => import('./pages/ChallengePage'));
+
  function App() {
    return (
+     <Suspense fallback={<Spinner />}>
        <Routes>
          <Route path="/challenge/:id" element={<ChallengePage />} />
        </Routes>
+     </Suspense>
    );
  }
```

---

## 3. Follow-up actions

1. **Run a baseline benchmark** before merging anything. Record p50/p95 latency, bundle size, and Lighthouse score.
2. **Create child issues** for each confirmed hotspot (e.g., “Paginate `/api/challenges`”, “Add DB indexes for challenges table”, “Virtualize ChallengeList”).
3. **Add a CI performance gate** so regressions are caught automatically:
   ```bash
   # Example threshold
   npm run load:test -- --latency-p95 200
   ```
4. **Instrument with real-user monitoring** (e.g., Web Vitals in the frontend, APM in the backend).
5. **Re-test after each change** and close #7 only when the measured bottleneck is gone and performance targets are met.

If you can paste the relevant files (especially the main list route, the DB schema, and the component that renders the biggest list), I can convert the placeholders above into exact diffs for `Yug-the-pirate-king/CodeWithUs`.
