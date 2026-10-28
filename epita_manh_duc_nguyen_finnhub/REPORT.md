# TP 2 — Student Report: Modern Python Tooling & Live API Systems

**Student Name:** Manh Duc NGUYEN  
**EPITA Login:** manh-duc.nguyen@epita.fr  
**Assigned API Group:** Group 1 — Financial Markets  
**Date:** 28/10/2026
**TestPyPI Live Package URL:** [https://test.pypi.org/project/epita_YOUR_LOGIN_DOMAIN/]  

---

## Step 0: Assigned Group Verification
- **Group Command Output:**  
  >>> SUCCESS: You are assigned to GROUP 2!
- **Assigned Domain:** Group 1 - Finnhub
- **Project Directory Name:** epita_manh_duc_nguyen_finnhub

---

## Part 1: Environment Isolation with `uv`

### 1. Comparison of `sys.executable` & `sys.prefix`
*Paste and explain the differences between `global_env_output.txt` and `uv_env_output.txt`:*

```text
[Global sys.executable]: /usr/bin/python3
[uv sys.executable]: /home/manhduc/practice/Eng-TOOLS-TP1-G1-S1/epita_manh_duc_nguyen_finnhub/.venv/bin/python

[Global sys.prefix]: /usr
[uv sys.prefix]: /home/manhduc/practice/Eng-TOOLS-TP1-G1-S1/epita_manh_duc_nguyen_finnhub/.venv
```

**Analysis:**  
In the global environment, sys.executable points to /usr/bin/python3, and sys.prefix is /usr, indicating that Python runs directly from the system installation.

In contrast, uv run uses the Python interpreter inside the project's .venv directory. The value of sys.prefix also points to this virtual environment, confirming that the execution is isolated from the global Python environment.

When executing uv run python scripts/inspect_env.py, uv automatically discovers the project environment, creates or synchronizes it when necessary, and runs the script using the appropriate Python interpreter and dependencies.

Unlike traditional virtual environment management, uv run does not require manually executing source .venv/bin/activate. It automatically configures the environment for the command without permanently modifying the parent shell session.
---

## Part 2: Live API Connection & 12-Factor Secret Hygiene

### 1. Git Status Output
*Paste the output of `git status` proving that `.env` is omitted and `.env.template` is tracked:*

```bash
$ git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
        new file:   .env.template
        modified:   .gitignore
        new file:   pyproject.toml
        new file:   src/epita_manh_duc_nguyen_finnhub/client.py
        new file:   src/epita_manh_duc_nguyen_finnhub/config.py
        new file:   uv.lock

Untracked files:
  (use "git add <file>..." to include in what will be committed)
        README.md
        REPORT_TEMPLATE.md
        src/epita_manh_duc_nguyen_finnhub/__init__.py
        src/epita_manh_duc_nguyen_finnhub/py.typed
```

### 2. Live API Response (Redacted)
*Paste the dictionary returned by calling your live client (`fetch_live_data()`):*

```json
{
  "status": "LIVE_SUCCESS",
  "symbol": "NVDA",
  "current_price": 216.78,
  "high_of_the_day": 219.99,
  "low_of_the_day": 216.33,
  "previous_close": 217.94
}
```

### 3. Security Analysis
- **Why is storing secrets in environment variables superior to putting them in configuration files like `config.json` or `settings.yaml`? Give 2 reasons:**
  1. Reduced risk of credential exposure: Environment variables separate sensitive credentials from application source code and tracked configuration files. This reduces the risk of accidentally committing API keys to a public Git repository. The local .env file is explicitly excluded through .gitignore.
  2. Secure configuration management across environments: Environment variables allow development, testing, and production environments to use different credentials without modifying application code. Credentials can be updated or rotated independently of the source code, supporting Factor III of the 12-Factor App methodology.
- **Why are default fallbacks like `os.getenv("API_KEY", "default")` dangerous for production secrets?**
Default fallback values can silently hide missing or incorrectly configured API credentials. Instead of failing immediately, the application may continue executing with an invalid, predictable, or insecure credential.

This can lead to authentication failures, unexpected behavior, and security vulnerabilities if shared fallback credentials are accepted by a service.

Production applications should follow the fail-fast principle by validating required credentials during initialization and raising an explicit error when they are missing or invalid.

In this project, config.validate() performs this validation before making the Finnhub API request.

---

## Part 3: Dependency Hygiene & Semantic Versioning

### 1. Dependency Tree (`uv tree`)
*Paste output of `uv tree`:*

```text
Resolved 16 packages in 1ms
epita-manh-duc-nguyen-finnhub v0.1.0
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

- **Direct dependencies:**
  + requests: Handles HTTP requests to the Finnhub API.

  + python-dotenv: Loads environment variables from the .env file.

  + rich: Provides formatted and enhanced terminal output.

  + pytest: Development-only dependency used for automated testing.
- **Transitive dependencies pulled in by `requests` / `rich`:** ...
  requests requires
    + certifi: Provides trusted CA certificates for HTTPS verification.

    + charset-normalizer: Handles character encoding detection.

    + idna: Supports internationalized domain names.

    + urllib3: Provides HTTP connection management.
  rich depends on:
    + markdown-it-py: Parses Markdown content for terminal rendering.

    + pygments: Provides syntax highlighting.

    + mdurl: An indirect dependency required by markdown-it-py for URL handling.
  Additionally, pytest introduces iniconfig, packaging, and pluggy, while sharing pygments with rich.
  These dependencies are automatically resolved by uv and recorded in uv.lock.

### 2. Lockfile & SemVer Analysis
1. **Meaning of version specifiers:**
   - `requests = "^2.31.0"`: Accepts versions >=2.31.0 and <3.0.0. It allows compatible minor and patch updates but excludes the next major release.
   - `urllib3 = "~=2.2.1"`: Accepts versions >=2.2.1 and <2.3.0. It permits compatible patch updates within the 2.2.x release series.
   - `rich = "==13.7.1"`: Requires exactly version 13.7.1, preventing automatic upgrades to other versions.
2. **If an upstream author releases `3.0.0`, will `^2.31.0` install it? Why or why not?**  
    No. The caret constraint ^2.31.0 restricts acceptable versions to >=2.31.0,<3.0.0.

    Therefore, version 3.0.0 is excluded because it falls outside the permitted range.

    If this constraint is expressed using its equivalent PEP 508 syntax (>=2.31.0,<3.0.0), uv lock --upgrade will respect the upper bound and will not select version 3.0.0.

    However, if the actual project declares only a minimum version such as requests>=2.34.2, a future major release may still be eligible unless another constraint excludes it.

    A major version upgrade requires explicitly modifying the dependency constraint to permit version 3.0.0.
3. **What is the fundamental difference between `pyproject.toml` (declaration) and `uv.lock` (pinning)?**  
    pyproject.toml declares the project's dependencies and their acceptable version ranges. It describes which packages are required without necessarily specifying one exact version.

    uv.lock, on the other hand, records the exact dependency versions resolved by uv, including transitive dependencies, package sources, and cryptographic hashes.

    Therefore, pyproject.toml defines the allowed dependencies, while uv.lock ensures that specific resolved versions can be reproduced consistently across development and deployment environments.
4. **How do the SHA-256 cryptographic hashes in `uv.lock` protect against supply chain attacks?**  
    The uv.lock file contains SHA-256 cryptographic hashes associated with downloaded package artifacts, including source distributions and Wheels.

    During installation, the package manager can verify the integrity of downloaded artifacts by comparing their calculated SHA-256 hashes with the expected hashes recorded in the lockfile.

    If an artifact has been unexpectedly modified, its hash will differ, allowing the installation process to detect and reject the mismatched file.

    This mechanism helps prevent unnoticed package tampering and improves build integrity and reproducibility.

    However, hash verification alone cannot detect malicious code already present in an approved package. Therefore, it should be combined with trusted package sources, dependency auditing, and secure version management.

---

## Part 4: Package Build & TestPyPI Deployment

### 1. TestPyPI Deployment Details
- **TestPyPI Package Name:** `epita-manh-duc-nguyen-finnhub`
- **Public URL:** `https://test.pypi.org/project/epita-manh-duc-nguyen-finnhub/`
- **CLI installation and execution output:**
```bash
$ uv run --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ epita-<login>-cli
╭─ EPITA Engineering Tools — Session 2 ─╮
│       Live System Telemetry:          │
│            LIVE_SUCCESS               │
│ ┏━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┓    │
│ ┃ Property        ┃ Value        ┃    │
│ ┡━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━┩    │
│ │ status          │ LIVE_SUCCESS │    │
│ │ symbol          │ NVDA         │    │
│ │ current_price   │ 215.38       │    │
│ │ high_of_the_day │ 219.99       │    │
│ │ low_of_the_day  │ 214.47       │    │
│ │ previous_close  │ 217.94       │    │
│ └─────────────────┴──────────────┘    │
╰───────────────────────────────────────╯
```

### 2. Packaging Theory
- **What is the difference between an `sdist` (.tar.gz) and a Wheel (.whl)?**  
  A Source Distribution (sdist) is an archive containing the source code and build metadata required to build a Python package. Installing an sdist generally requires executing a build process to produce an installable distribution.

  A Wheel (.whl) is a built distribution format containing files prepared for direct installation, including package metadata. Unlike an sdist, a Wheel normally does not require building the package during installation.
- **Why are Wheels significantly faster and safer to install on client machines?**  
  Wheels are faster because they avoid building the package from source during installation. This reduces installation time, compilation overhead, and build-time dependencies.

  Wheels can also reduce security risks because installation generally does not require executing arbitrary project-specific build scripts on the client's machine.

  However, Wheels are not inherently secure. They may still contain malicious code, so trusted package sources, dependency auditing, and artifact integrity verification remain important.
---

## Part 5: Professional Debugging Challenge (No `print()`!)

### 1. Root Cause Analysis
- **Observed types in the debugger:**  
  - Runtime type of `record['volume']`: `str`  
  - Runtime type of `record['weight']`: `str`  
- **Why did Python fail or miscalculate without crashing initially?**  
  The root cause was a runtime type mismatch in the parse_api_record() function.

  The function uses split(",") to parse the raw API payload. However, this operation returns strings, even when the original values represent numbers.

  Consequently, both volume and weight were stored as strings instead of numeric values.

  When compute_weighted_metric() attempted to multiply two strings using volume * weight, Python raised a TypeError.

  Furthermore, multiplying a string by an integer would perform string repetition rather than numerical multiplication, potentially producing incorrect results without an immediate exception.

  The issue was identified using the interactive Python debugger (pdb) without adding any debugging print() statements.

### 2. Call Stack Inspection
*Describe the Call Stack frames observed when paused at the conditional breakpoint inside `compute_weighted_metric`:*
```text
Frame 1: compute_weighted_metric() — Line 19
         Executes the weighted metric calculation:
         return volume * weight
Frame 2: aggregate_telemetry() — Line 32
         Calls compute_weighted_metric() while
         processing a successful API record.
Frame 3: <module> — Line 66
         Initializes the sample API payload and calls:
         aggregate_telemetry(api_payload)
```

### 3. The Fix (Code Diff / Snippet)
```python
def parse_api_record(raw_entry: str) -> dict:
    parts = raw_entry.strip().split(",")
    return {
        "id": parts[0],
        "entity": parts[1],
-       "volume": parts[2],
+       "volume": float(parts[2]),
        "status": parts[3],
-       "weight": parts[4]
+       "weight": float(parts[4])
    }
```

### 4. Verified Execution Output
```text
$ uv run python scripts/debug_challenge.py
Aggregate Result: {'successful_entries': 3, 'total_volume': 650.0, 'avg_volume': 216.66666666666666, 'high_impact_count': 1}
```
