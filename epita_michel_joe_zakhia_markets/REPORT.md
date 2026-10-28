# TP 2 — Student Report: Modern Python Tooling & Live API Systems

**Student Name:** Michel-Joe Zakhia  
**EPITA Login:** michel-joe.zakhia  
**Assigned API Group:** Group 1 — Markets (Finnhub)  
**Date:** 28/10/2026  
**TestPyPI Live Package URL:** <https://test.pypi.org/project/epita-michel-joe-zakhia-markets/>  

---

## Step 0: Assigned Group Verification
- **Group Command Output:**  
  ```text
  Enter your EPITA login (e.g. ismail.lachheb): michel-joe.zakhia

  >>> SUCCESS: You are assigned to GROUP 3!
  ```
  > **Note:** The formula assigns my login to Group 3, but this lab is done as a team and our team was assigned **Group 1 (Financial Markets / Finnhub)**, so the whole project targets Group 1.
- **Assigned Domain:** `Group 1 - Finnhub`
- **Project Directory Name:** `epita_michel_joe_zakhia_markets`

---

## Part 1: Environment Isolation with `uv`

### 1. Comparison of `sys.executable` & `sys.prefix`

```text
[Global sys.executable]: /Applications/Xcode.app/Contents/Developer/usr/bin/python3
[uv sys.executable]:     <project>/.venv/bin/python

[Global sys.prefix]:     /Applications/Xcode.app/Contents/Developer/Library/Frameworks/Python3.framework/Versions/3.9
[uv sys.prefix]:         <project>/.venv
```

| | Global (`python3`) | `uv run python` |
| :--- | :--- | :--- |
| Python version | 3.9.6 (Xcode system Python) | 3.13.18 (pinned by `.python-version`) |
| `sys.base_prefix` | same as `sys.prefix` | `~/.local/share/uv/python/cpython-3.13.18-macos-aarch64-none` |
| In a venv? | `False` | `True` |
| First `$PATH` entry | `~/.local/bin` | `<project>/.venv/bin` |
| `site-packages` | global Xcode `site-packages` | `<project>/.venv/lib/python3.13/site-packages` |

#### `$PATH` comparison (first 4 entries, from the two output files)
```text
global_env_output.txt                 uv_env_output.txt
=== OS Executable Path ($PATH) ===     === OS Executable Path ($PATH) ===
  - /Users/micho/.local/bin             - <project>/.venv/bin          <-- injected by uv run
  - /opt/homebrew/bin                   - /Users/micho/.local/bin
  - /opt/homebrew/sbin                  - /opt/homebrew/bin
  - /usr/local/bin                      - /opt/homebrew/sbin
```

The shell resolves a command name by scanning `$PATH` left to right and running the first match. Globally, no directory before `/usr/bin` contains `python3`, so `python3` resolves to the macOS stub `/usr/bin/python3`, which forwards to the Xcode interpreter (hence the `/Applications/Xcode.app/...` executable). Under `uv run`, `<project>/.venv/bin` is **prepended** to `$PATH`, so `python` resolves to `.venv/bin/python` first, as does any console script installed in the venv (e.g. `epita-michel-joe-zakhia-cli`, `pytest`). The rest of `$PATH` is unchanged, only shifted by one entry.

#### `sys.path` comparison (import search order)
- Global: `.../Python3.framework/Versions/3.9/lib/python3.9/...` and the global `site-packages` → packages are shared by every project on the machine.
- `uv run`: the uv-managed CPython 3.13 standard library, then `<project>/.venv/lib/python3.13/site-packages` → only this project's locked dependencies are importable.

**Analysis:**  
Globally, `python3` resolves to the macOS/Xcode system interpreter. `sys.prefix == sys.base_prefix`, so there is no isolation, and any package installed would land in the shared system `site-packages`.

Under `uv run`, `sys.executable` is `.venv/bin/python` and `sys.prefix` points to `.venv`, while `sys.base_prefix` points to the uv-managed CPython 3.13 the venv was built from. The fact that the two prefixes differ is exactly what Python uses to detect a virtual environment, so imports now resolve from `.venv/lib/python3.13/site-packages`.

`uv run` achieves this without `source .venv/bin/activate`: it walks up from the current directory to find `pyproject.toml`, checks that `.venv` is in sync with `uv.lock` (creating/updating it if needed), then launches the command as a child process with `.venv/bin` prepended to `PATH` and `VIRTUAL_ENV` set — visible as the first `$PATH` entry in `uv_env_output.txt`. Those changes exist only for that one process; the parent shell is never modified, so there is nothing to activate or deactivate and no risk of running in the wrong environment.

### 2. `.gitignore`
`.venv/`, `.env`, `__pycache__/`, `.pytest_cache/`, `dist/`, `*.egg-info/` and `*.txt` are ignored. `.gitignore` and `scripts/inspect_env.py` are committed.

---

## Part 2: Live API Connection & 12-Factor Secret Hygiene

### 1. Git Status Output
```bash
$ git status .
On branch main
Your branch is ahead of 'origin/main' by 3 commits.
  (use "git push" to publish your local commits)

nothing to commit, working tree clean

$ git status --short --ignored .
!! .env
!! .venv/
!! global_env_output.txt
!! src/epita_michel_joe_zakhia_markets/__pycache__/
!! uv_env_output.txt

$ git ls-files .env.template .env
.env.template

$ git check-ignore -v .env
.gitignore:2:.env	.env
```

`.env` exists locally (it holds the real key) but never appears as untracked or staged: it is only listed as ignored (`!!`), matched by line 2 of `.gitignore`. `.env.template` (placeholder values only) is tracked and committed. `.venv/`, `__pycache__/` and the `*.txt` outputs are ignored too.

### 2. Live API Response (Redacted)
```bash
$ uv run python -c "from epita_michel_joe_zakhia_markets.client import fetch_live_data; print(fetch_live_data())"
```

```json
{
  "status": "LIVE_SUCCESS",
  "symbol": "NVDA",
  "current_price": 216.98,
  "high_of_the_day": 219.99,
  "low_of_the_day": 216.62,
  "previous_close": 217.94
}
```
*(The API key is sent only in the `X-Finnhub-Token` header and never appears in the response.)*

The key is read from the group-specific variable `FINNHUB_API_KEY` and validated defensively in `config.py`, before any request is sent:
```python
api_key: str = os.getenv("FINNHUB_API_KEY", "")   # empty default: no hidden fallback key

def validate(self):
    if not self.api_key or self.api_key.startswith("your_"):   # missing, or still the template placeholder
        raise ValueError("CRITICAL SECURITY / CONFIG ERROR: API key is not configured! ...")
```
`client.fetch_live_data()` calls `config.validate()` first, and the request uses `timeout=10` and `resp.raise_for_status()`, so HTTP errors (e.g. `401` for a bad key) surface as exceptions instead of silently returning bad data.

Without a valid key (e.g. `.env` freshly copied from `.env.template`), `config.validate()` fails fast before any network call:
```text
ValueError: CRITICAL SECURITY / CONFIG ERROR: API key is not configured!
Ensure your .env file exists and contains a valid API key.
```

### 3. Security Analysis
- **Why are environment variables superior to `config.json` / `settings.yaml` for secrets?**
  1. **No secrets in version control.** Config files usually live next to the code and get committed, so a key ends up in Git history forever (and in every clone, fork and CI cache). Environment variables are injected at runtime, and the only file that holds them locally (`.env`) is git-ignored; the repo ships only `.env.template` with dummy values.
  2. **Per-environment isolation and easy rotation.** The same code artifact runs in dev, staging and prod, each with its own credentials supplied by the environment (shell, CI secrets, secret manager). A leaked dev key does not expose prod, and rotating a key means changing the environment, not editing, rebuilding and redeploying code.
- **Why are default fallbacks like `os.getenv("API_KEY", "default")` dangerous?**
  A fallback hides misconfiguration: if the variable is missing, the app silently runs with a fake or shared hardcoded value instead of crashing at startup. That either fails later in confusing ways (401s deep in the app) or, worse, a real default key gets hardcoded in source and leaked. Using an empty default plus `validate()` makes the app fail fast with a clear error.

---

## Part 3: Dependency Hygiene & Semantic Versioning

### 1. Dependency Tree (`uv tree`)

```text
$ uv tree
Resolved 16 packages in 3ms
epita-michel-joe-zakhia-markets v0.1.0
├── python-dotenv v1.2.4
├── requests v2.34.2
│   ├── certifi v2026.10.23
│   ├── charset-normalizer v3.5.2
│   ├── idna v3.20
│   └── urllib3 v2.8.0
├── rich v15.0.0
│   ├── markdown-it-py v4.2.0
│   │   └── mdurl v0.1.2
│   └── pygments v2.21.0
└── pytest v9.2.0 (group: dev)
    ├── iniconfig v2.3.1
    ├── packaging v26.3
    ├── pluggy v1.7.0
    └── pygments v2.21.0
```

- **Direct dependencies** (declared in `pyproject.toml`):
  - Runtime (`[project].dependencies`): `python-dotenv`, `requests`, `rich`
  - Development only (`[dependency-groups].dev`, not shipped in the wheel's requirements): `pytest`
- **Transitive dependencies:**
  - Pulled in by `requests`: `certifi` (CA bundle), `charset-normalizer` (encoding detection), `idna` (internationalized domain names), `urllib3` (HTTP connection pooling)
  - Pulled in by `rich`: `markdown-it-py` → `mdurl`, and `pygments` (syntax highlighting)
  - (Pulled in by `pytest`, dev only: `iniconfig`, `packaging`, `pluggy`, `pygments`)

### 2. Lockfile & SemVer Analysis
1. **Meaning of version specifiers:**
   - `requests = "^2.31.0"`: caret = "compatible with 2.31.0" → `>=2.31.0, <3.0.0`. Any minor/patch release of major version 2 is allowed, never 3.x.
   - `urllib3 = "~=2.2.1"`: PEP 440 compatible-release → `>=2.2.1, <2.3.0`. Only patch updates of 2.2 are allowed.
   - `rich = "==13.7.1"`: exact pin → only version `13.7.1`, nothing else.

   *Note:* `^` is Poetry/npm syntax and is not valid PEP 440; in a `pyproject.toml` read by `uv`, the equivalent is `requests>=2.31.0,<3`.
2. **If an upstream author releases `3.0.0`, will `^2.31.0` install it?**  
   No. Under SemVer a major bump (2 → 3) signals breaking API changes, and the caret range has an upper bound of `<3.0.0`. `uv lock --upgrade` re-resolves to the newest version *that still satisfies the constraints*, so it would move to the latest 2.x and ignore 3.0.0. Moving to 3.x requires deliberately editing the constraint.  
   (By contrast, the `>=` specifiers that `uv add` writes by default, e.g. `requests>=2.34.2`, have **no** upper bound, so `uv lock --upgrade` *would* pick 3.0.0 — the lockfile and tests are what protect us there.)
3. **`pyproject.toml` (declaration) vs `uv.lock` (pinning):**  
   `pyproject.toml` is the human-edited *intent*: which direct packages the project needs and which version *ranges* are acceptable. It contains no transitive dependencies and several concrete versions could satisfy it. `uv.lock` is the machine-generated *resolution*: the exact version of every package in the whole graph (direct + transitive, 16 here), its source URL and its file hashes. The same lockfile produces the byte-identical environment on every machine and in CI (`uv sync --locked`), while `pyproject.toml` alone could resolve differently tomorrow.
4. **How do SHA-256 hashes in `uv.lock` protect against supply-chain attacks?**  
   Each sdist and wheel in `uv.lock` records the SHA-256 digest of the exact file that was resolved, e.g.:
   ```toml
   wheels = [
       { url = ".../requests-2.34.2-py3-none-any.whl", hash = "sha256:2a0d60c172f83ac6ab31e4554906c0f3b3588d37b5cb939b1c061f4907e278e0", ... },
   ]
   ```
   On install, `uv` hashes the downloaded file and refuses to install it if the digest does not match. So if the package index, a mirror or a proxy is compromised, or an attacker re-uploads a tampered file under the same name and version, the hash check fails and the build stops instead of executing malicious code. Because every transitive package is hashed too (136 hashes in our lockfile), the whole dependency graph is verified, not just the direct dependencies.

---

## Part 4: Package Build & TestPyPI Deployment

### 1. CLI Entrypoint & Local Run
```toml
[project.scripts]
epita-michel-joe-zakhia-cli = "epita_michel_joe_zakhia_markets.cli:main"
```

```text
$ uv run epita-michel-joe-zakhia-cli
╭─ EPITA Engineering Tools — Session 2 ─╮
│       Live System Telemetry:          │
│            LIVE_SUCCESS               │
│ ┏━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┓    │
│ ┃ Property        ┃ Value        ┃    │
│ ┡━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━┩    │
│ │ status          │ LIVE_SUCCESS │    │
│ │ symbol          │ NVDA         │    │
│ │ current_price   │ 217.04       │    │
│ │ high_of_the_day │ 219.99       │    │
│ │ low_of_the_day  │ 216.62       │    │
│ │ previous_close  │ 217.94       │    │
│ └─────────────────┴──────────────┘    │
╰───────────────────────────────────────╯
```

### 2. Build Artifacts (`uv build`)
```text
Successfully built dist/epita_michel_joe_zakhia_markets-0.1.0.tar.gz
Successfully built dist/epita_michel_joe_zakhia_markets-0.1.0-py3-none-any.whl
```

### 3. TestPyPI Deployment Details
- **TestPyPI Package Name:** `epita-michel-joe-zakhia-markets`
- **Public URL:** `https://test.pypi.org/project/epita-michel-joe-zakhia-markets/`
- **CLI installation and execution output:**
```bash
$ uv publish --publish-url https://test.pypi.org/legacy/ --token "pypi-***"
Publishing 2 files to https://test.pypi.org/legacy/
Uploading epita_michel_joe_zakhia_markets-0.1.0-py3-none-any.whl (3.9KiB)
Uploading epita_michel_joe_zakhia_markets-0.1.0.tar.gz (2.1KiB)

$ cd /tmp
$ export FINNHUB_API_KEY="***"
$ uvx --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ --index-strategy unsafe-best-match --from epita-michel-joe-zakhia-markets epita-michel-joe-zakhia-cli
Installed 11 packages in 10ms
╭─ EPITA Engineering Tools — Session 2 ─╮
│       Live System Telemetry:          │
│            LIVE_SUCCESS               │
│ ┏━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┓    │
│ ┃ Property        ┃ Value        ┃    │
│ ┡━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━┩    │
│ │ status          │ LIVE_SUCCESS │    │
│ │ symbol          │ NVDA         │    │
│ │ current_price   │ 216.63       │    │
│ │ high_of_the_day │ 219.99       │    │
│ │ low_of_the_day  │ 216.56       │    │
│ │ previous_close  │ 217.94       │    │
│ └─────────────────┴──────────────┘    │
╰───────────────────────────────────────╯
```

Notes on the verification command:
- `uvx` (= `uv tool run`) is used instead of `uv run`: outside a project, `uv run` has no environment containing the CLI, whereas `uvx --from <package>` installs the package into a throw-away environment and runs its `[project.scripts]` entrypoint.
- `--index-strategy unsafe-best-match` is needed because TestPyPI hosts old/squatted copies of `requests`, `rich`, etc.; with uv's default "first index wins" strategy it would try to take those from TestPyPI and fail to satisfy our version constraints.
- `FINNHUB_API_KEY` is exported in the shell because the installed tool runs outside the project, where there is no `.env` — the secret is still supplied by the environment, never baked into the package.

### 4. Packaging Theory
- **What is the difference between an `sdist` (.tar.gz) and a Wheel (.whl)?**  
  An **sdist** is an archive of the *source*: `pyproject.toml`, `src/` and metadata. To install it, the client must download the build backend (`uv_build`) declared in `[build-system]` and run a build step locally to produce a wheel. A **wheel** is the *already-built* distribution: a zip whose files are laid out exactly as they must land in `site-packages`, plus `dist-info` metadata (`METADATA`, `entry_points.txt`, `RECORD`). The tag `py3-none-any` means it is pure Python and works on any OS/CPU with Python 3.
- **Why are Wheels significantly faster and safer to install on client machines?**
  1. **Faster:** installing a wheel is just unzip-and-copy; there is no build environment to create, no build-backend download and no compilation (critical for packages with C extensions such as `numpy`, where building from source can take minutes and needs a compiler).
  2. **Safer:** an sdist build *executes arbitrary code* from the package (the build backend, or `setup.py` for legacy packages) on the client machine at install time, which is a classic supply-chain attack vector. A wheel executes no code during installation. It is also reproducible: the exact file that was tested is what gets installed, and every file inside is listed with its SHA-256 hash in `RECORD`, so tampering is detectable.

---

## Part 5: Professional Debugging Challenge (No `print()`!)

Debugger used: Python's interactive debugger, `uv run python -m pdb scripts/debug_challenge.py` (no `print()` added). The breakpoint line is **line 32** in our file (`impact = compute_weighted_metric(...)`).

### 1. Root Cause Analysis
- **Observed types in the debugger:**
  - Runtime type of `record['volume']`: `str` (`'120'`)
  - Runtime type of `record['weight']`: `str` (`'2'`)

```text
(Pdb) b 32
(Pdb) c
> scripts/debug_challenge.py(32)aggregate_telemetry()
-> impact = compute_weighted_metric(record["volume"], record["weight"])
(Pdb) p idx
0
(Pdb) p type(record["volume"]), type(record["weight"])
(<class 'str'>, <class 'str'>)
```

- **Why did the original code fail?**  
  `parse_api_record` splits a CSV line with `str.split(",")`, which always yields **strings**; the numeric fields were never converted. In `compute_weighted_metric`, `'120' * '2'` is `str * str`, which Python rejects:
  ```text
  TypeError: can't multiply sequence by non-int of type 'str'
  ```
  This happens on the **first** SUCCESS record (`idx == 0`), so with only the conditional breakpoint `b 32, idx == 2`, the program crashed *before* the breakpoint was ever hit and pdb dropped into post-mortem mode at `return volume * weight`, with `volume = '120'`, `weight = '2'`.  
  The bug is also insidious because Python is dynamically typed and the code "almost" works: had only `weight` been an `int`, `'120' * 2` would silently produce the string `'120120'` (sequence repetition) instead of `240`. The failure would then surface later and further from the cause (`'120120' > 500` → `TypeError: '>' not supported between 'str' and 'int'`, and `total_volume += '120'` → `TypeError: unsupported operand type(s) for +=: 'int' and 'str'`). Type hints such as `raw_entry: str -> dict` are not enforced at runtime, so nothing stopped the strings from flowing through.

### 2. Call Stack Inspection
Paused after stepping into (`s`, equivalent of `F11`) `compute_weighted_metric`, the `where` command shows:
```text
(Pdb) where
  scripts/debug_challenge.py(66)<module>()
-> result = aggregate_telemetry(api_payload)
  scripts/debug_challenge.py(32)aggregate_telemetry()
-> impact = compute_weighted_metric(record["volume"], record["weight"])
> scripts/debug_challenge.py(20)compute_weighted_metric()
-> return volume * weight
(Pdb) args
volume = '120'
weight = '2'
```
- **Frame 1 — `<module>` (line 66):** the script's top level, which called `aggregate_telemetry(api_payload)`.
- **Frame 2 — `aggregate_telemetry` (line 32):** the loop, with locals `idx`, `row`, `record`, `total_volume`, `successful_entries`, `high_impact_count`; it called `compute_weighted_metric`.
- **Frame 3 — `compute_weighted_metric` (line 20), current frame:** arguments `volume='120'`, `weight='2'`, about to evaluate `volume * weight`.

(Above `<module>`, pdb also lists its own internal frames `bdb.run` / `<string>`, which are the debugger launching the script.)

### 3. The Fix (Code Diff / Snippet)
The fix is applied at the source of the bad data — the parser — so every consumer receives numbers, instead of patching `compute_weighted_metric` and the accumulators separately:
```diff
@@ def parse_api_record(raw_entry: str) -> dict:
     return {
         "id": parts[0],
         "entity": parts[1],
-        "volume": parts[2],        # <-- Hint: What runtime type is this?
+        "volume": float(parts[2]),  # CSV fields are str: cast to a number at parse time
         "status": parts[3],
-        "weight": parts[4]          # <-- Hint: What runtime type is this?
+        "weight": int(parts[4])     # multiplier is an integer count
     }
```
`volume` is a `float` so that `total_volume` comes out as `650.0` as expected; `weight` is an integer multiplier.

Re-checked in the debugger with the conditional breakpoint, which now fires on the third record (REC_003):
```text
(Pdb) b 32, idx == 2
(Pdb) c
-> impact = compute_weighted_metric(record["volume"], record["weight"])
(Pdb) p idx
2
(Pdb) p type(record["volume"]), type(record["weight"])
(<class 'float'>, <class 'int'>)
(Pdb) s
(Pdb) s
(Pdb) args
volume = 450.0
weight = 3
(Pdb) p volume * weight
1350.0
```

### 4. Verified Execution Output
```text
$ uv run python scripts/debug_challenge.py
Aggregate Result: {'successful_entries': 3, 'total_volume': 650.0, 'avg_volume': 216.66666666666666, 'high_impact_count': 1}
```
This matches the expected result exactly (650 / 3 = 216.67, and only REC_003 has impact > 500).
