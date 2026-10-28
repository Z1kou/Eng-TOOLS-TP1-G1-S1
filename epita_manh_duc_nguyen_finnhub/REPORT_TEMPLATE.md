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
[Paste output of uv tree here]
```

- **Direct dependencies:** ...
- **Transitive dependencies pulled in by `requests` / `rich`:** ...

### 2. Lockfile & SemVer Analysis
1. **Meaning of version specifiers:**
   - `requests = "^2.31.0"`: ...
   - `urllib3 = "~=2.2.1"`: ...
   - `rich = "==13.7.1"`: ...
2. **If an upstream author releases `3.0.0`, will `^2.31.0` install it? Why or why not?**  
   ...
3. **What is the fundamental difference between `pyproject.toml` (declaration) and `uv.lock` (pinning)?**  
   ...
4. **How do the SHA-256 cryptographic hashes in `uv.lock` protect against supply chain attacks?**  
   ...

---

## Part 4: Package Build & TestPyPI Deployment

### 1. TestPyPI Deployment Details
- **TestPyPI Package Name:** `epita_<login>_<domain>`
- **Public URL:** `https://test.pypi.org/project/epita_<login>_<domain>/`
- **CLI installation and execution output:**
```bash
$ uv run --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ epita-<login>-cli
[Paste output here]
```

### 2. Packaging Theory
- **What is the difference between an `sdist` (.tar.gz) and a Wheel (.whl)?**  
  ...
- **Why are Wheels significantly faster and safer to install on client machines?**  
  ...

---

## Part 5: Professional Debugging Challenge (No `print()`!)

### 1. Root Cause Analysis
- **Observed types in the debugger:**  
  - Runtime type of `record['volume']`: `[e.g. str / int]`  
  - Runtime type of `record['weight']`: `[e.g. str / int]`  
- **Why did Python fail or miscalculate without crashing initially?**  
  ...

### 2. Call Stack Inspection
*Describe the Call Stack frames observed when paused at the conditional breakpoint inside `compute_weighted_metric`:*
```text
Frame 1: ...
Frame 2: ...
Frame 3: ...
```

### 3. The Fix (Code Diff / Snippet)
```python
# Show your code correction in parse_api_record or aggregate_telemetry
```

### 4. Verified Execution Output
```text
$ uv run python scripts/debug_challenge.py
Aggregate Result: {'successful_entries': 3, 'total_volume': 650.0, 'avg_volume': 216.66666666666666, 'high_impact_count': 1}
```
