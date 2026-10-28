# TP 2 — Student Report: Modern Python Tooling & Live API Systems

**Student Name:** Michel-Joe Zakhia  
**EPITA Login:** michel-joe.zakhia  
**Assigned API Group:** Group 1 — Markets (Finnhub)  
**Date:** 28/10/2026  
**TestPyPI Live Package URL:** [https://test.pypi.org/project/epita_michel_joe_zakhia_markets/]  

---

## Step 0: Assigned Group Verification
- **Group Command Output:**  
  *(Paste the output of `python3 -c "..."` proving your assigned group)*
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

**Analysis:**  
Globally, `python3` resolves to the macOS/Xcode system interpreter. `sys.prefix == sys.base_prefix`, so there is no isolation, and any package installed would land in the shared system `site-packages`.

Under `uv run`, `sys.executable` is `.venv/bin/python` and `sys.prefix` points to `.venv`, while `sys.base_prefix` points to the uv-managed CPython 3.13 the venv was built from. The fact that the two prefixes differ is exactly what Python uses to detect a virtual environment, so imports now resolve from `.venv/lib/python3.13/site-packages`.

`uv run` achieves this without `source .venv/bin/activate`: it walks up from the current directory to find `pyproject.toml`, checks that `.venv` is in sync with `uv.lock` (creating/updating it if needed), then launches the command as a child process with `.venv/bin` prepended to `PATH` and `VIRTUAL_ENV` set — visible as the first `$PATH` entry in `uv_env_output.txt`. Those changes exist only for that one process; the parent shell is never modified, so there is nothing to activate or deactivate and no risk of running in the wrong environment.

### 2. `.gitignore`
`.venv/`, `.env`, `__pycache__/`, `.pytest_cache/`, `dist/`, `*.egg-info/` and `*.txt` are ignored. `.gitignore` and `scripts/inspect_env.py` are committed.
