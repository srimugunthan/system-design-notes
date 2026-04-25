## Dead Code Hunting — The Non-LLM Way

The core idea is to use **static analysis tools + runtime tracing + manual reading**, depending on how confident you need to be.

---

### 1. Static Analysis Tools (fastest starting point)

These parse your code without running it and flag things that look unused.

**Python**
- `vulture myproject/` — dedicated dead code finder, very straightforward
- `pylint` — flags unused imports (`W0611`) and unused variables (`W0612`)
- `pyflakes` — lightweight, fast, good for unused imports

**JavaScript / TypeScript**
- `ts-prune` — finds exports that are never imported anywhere
- `eslint` with `no-unused-vars` and `no-unreachable` rules
- `knip` — newer, more aggressive; finds unused files, exports, and dependencies

**General / Multi-language**
- `semgrep` — write custom rules to find patterns you care about
- IDE built-ins (VSCode, PyCharm) — grey-out unused symbols right in the editor; underrated

---

### 2. Coverage-Based Detection (dynamic, more reliable)

Static tools can miss things that *look* used but are only reachable under conditions that never actually occur in practice.

Run your test suite with coverage enabled:

```bash
# Python
pytest --cov=myproject --cov-report=html

# JS
jest --coverage
```

Then look at the HTML report. Lines that are **never hit** across your full test suite are candidates for dead code — not guaranteed, but strong signal.

The limitation: if your tests have poor coverage, this tells you more about your tests than your code.

---

### 3. Call Graph Analysis (deeper, for functions)

Tools that build an actual graph of what calls what, then find nodes with no callers.

- **Python**: `pycallgraph2`, or `py-call-graph` for visualization; `radon` for complexity + reachability hints
- **C/C++**: `cflow`, `egypt` (with gcc's `-fdump-rtl-expand`)
- **Java**: IntelliJ's "Find Usages" is the gold standard here

For a quick manual version: pick a function, grep for its name across the entire repo. If the only hit is its own definition, it's dead.

```bash
grep -rn "function_name" ./src
```

---

### 4. Manual Reading with Intent

Tools catch the obvious cases. The subtler ones require reading:

- **Unreachable branches**: look for conditions like `if config.FEATURE_FLAG and False:`, or constants that make one branch permanently dead
- **Commented-out code blocks**: often dead code in disguise, kept "just in case"
- **Functions called only by other dead functions**: tools miss these chains unless they do full reachability from entry points

The discipline here is to always ask: *what is the entry point, and can I trace a path from there to this code?*

---

### 5. Git as a Dead Code Signal

Not a detector, but a useful lens:

```bash
git log --follow -p -- path/to/file.py
```

If a function hasn't been touched in 2 years while everything around it has been refactored heavily, that's a strong hint it's either dead or load-bearing in a way nobody remembers. Both are worth investigating.

`git blame` on a suspicious block tells you *who* wrote it and *when* — often enough context to know if it was speculative scaffolding or a deliberate guard.

---

### The honest limitation

No non-LLM tool can tell you **why** dead code exists or whether it's safe to delete. That judgment always requires a human reading git history, comments, and surrounding context. The tools just help you find *where to look*.
