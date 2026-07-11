# AI Fix — Issue #1: Cleanup: Remove dead code and unused imports

**Issue body:**

This is an automated issue created by the AI agent to track planned code quality improvements. The AI will fix this in a subsequent run.

---

**AI-proposed fix:**

I can’t provide **exact** file paths or line-level deletions without inspecting the repository contents (which I don’t have access to from this prompt). However, below is the standard root-cause analysis, a concrete cleanup checklist you can apply directly, and follow-up actions to prevent regressions.

---

### 1. Root Cause

Dead code and unused imports typically accumulate from:

- **Refactoring leftovers**: old methods, variables, or files kept after logic was moved elsewhere.
- **Commented-out blocks** left for “quick testing” and never removed.
- **Unused imports** added by auto-completion or copy-paste and never cleaned up.
- **Abandoned branches/features** merged without final cleanup.

This increases cognitive load, build time, and lint noise without adding value.

---

### 2. Exact Code Changes Needed

Apply the following **per language** across the codebase. Replace paths with the actual files in your repo.

#### A. Remove unused imports

- **Python**: delete any `import` or `from ... import` lines not referenced.
  ```python
  # BEFORE
  import os
  import json
  from datetime import datetime

  def main():
      print(datetime.now())

  # AFTER
  from datetime import datetime

  def main():
      print(datetime.now())
  ```

- **Java / Kotlin**: delete unused `import` statements and wildcard imports that are no longer needed.
  ```java
  // BEFORE
  import java.util.List;
  import java.util.ArrayList;
  import java.io.File;          // unused

  // AFTER
  import java.util.ArrayList;
  import java.util.List;
  ```

- **JavaScript / TypeScript**: remove unused `import`, `require`, or `import type` statements.
  ```ts
  // BEFORE
  import React, { useState } from 'react';
  import axios from 'axios';   // unused

  // AFTER
  import React, { useState } from 'react';
  ```

#### B. Delete dead code

Remove files/functions/variables that are never called:

- Functions with no references.
- Empty catch/try blocks.
- Large blocks of commented-out code.
- Unused constants or configuration values.
- Duplicate helper methods—consolidate into one.

Example refactor:
```python
# BEFORE
def old_helper(x):
    return x * 2  # no longer used

def new_helper(x):
    return x + 1

# AFTER
def new_helper(x):
    return x + 1
```

#### C. Use linters to identify candidates

Run the appropriate command for your stack:

- **Python**: `ruff check .` or `flake8 .` (targets `F401` unused imports, `F841` unused variables)
- **JavaScript/TypeScript**: `eslint .` (rules `@typescript-eslint/no-unused-vars`, `unused-imports/no-unused-imports`)
- **Java**: `mvn compile` / IDE inspection for “Unused declaration”
- **Go**: `go vet ./...` and `golangci-lint run`
- **Rust**: `cargo clippy`

---

### 3. Follow-Up Actions

1. **Automate linting in CI**  
   Add a GitHub Actions step that fails the build on unused imports or dead-code warnings.

2. **Enable editor/IDE warnings**  
   Configure team IDEs (VS Code, IntelliJ, etc.) to highlight unused imports and unreachable code.

3. **Add a pre-commit hook**  
   Use `pre-commit` (Python), `husky` (JS/TS), or similar to block commits with lint errors.

4. **Document cleanup in PR checklist**  
   Require reviewers to confirm: “No commented-out code or unused imports remain.”

5. **Schedule recurring cleanup**  
   Create a monthly “code health” issue/PR to prevent future accumulation.

---

If you paste the repository files or grant me access to the actual code, I can produce the precise diff with real file paths and line numbers.
