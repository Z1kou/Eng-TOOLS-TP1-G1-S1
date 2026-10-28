# TP 2 — Student Report: Modern Python Tooling & Live API Systems

**Student Name:** [FIRSTNAME LASTNAME]  
**EPITA Login:** [login]  
**Assigned API Group:** [Group 0 to 7] — [Domain Name, e.g. Fintech / Weather / Markets]  
**Date:** [DD/MM/YYYY]  
**TestPyPI Live Package URL:** [https://test.pypi.org/project/epita_YOUR_LOGIN_DOMAIN/]  

---

## Step 0: Assigned Group Verification
- **Group Command Output:**  
  *(Paste the output of `python3 -c "..."` proving your assigned group)*
- **Assigned Domain:** `[e.g. Group 1 - Finnhub]`
- **Project Directory Name:** `epita_<login>_<domain>`

---

## Part 1: Environment Isolation with `uv`

### 1. Comparison of `sys.executable` & `sys.prefix`
*Paste and explain the differences between `global_env_output.txt` and `uv_env_output.txt`:*

```text
[Global sys.executable]: 
[uv sys.executable]: 

[Global sys.prefix]: 
[uv sys.prefix]: 
```

**Analysis:**  
*(Explain how `uv run` dynamically isolates the execution context without manual shell activation)*

---

## Part 2: Live API Connection & 12-Factor Secret Hygiene

### 1. Git Status Output
*Paste the output of `git status` proving that `.env` is omitted and `.env.template` is tracked:*

```bash
$ git status
[Paste output here]
```

### 2. Live API Response (Redacted)
*Paste the dictionary returned by calling your live client (`fetch_live_data()`):*

```json
{
  "status": "LIVE_SUCCESS",
  ...
}
```

### 3. Security Analysis
- **Why is storing secrets in environment variables superior to putting them in configuration files like `config.json` or `settings.yaml`? Give 2 reasons:**
  1. ...
  2. ...
- **Why are default fallbacks like `os.getenv("API_KEY", "default")` dangerous for production secrets?**
  ...

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
