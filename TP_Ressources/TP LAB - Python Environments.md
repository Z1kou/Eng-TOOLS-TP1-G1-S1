# TP 2: Modern Python Tooling with `uv`, Live API Systems & TestPyPI Deployment

**Course:** Engineering Tools — Session 2 (CM2)  
**Institution:** EPITA — School of Engineering and Computer Science  
**Instructor:** Ismail Lachheb  
**Evaluation:** Graded Lab (Barème sur 20 points)  
**Submission:** By Email (instructions at the end)  
**Toolchain Focus:** `uv` (Astral), Live REST APIs, PyPI / TestPyPI, Interactive Debugger  
**Estimated Time:** 1h30 – 2h00  

---

## 🎯 Learning Objectives

By the end of this practical lab, you will be able to:
1. **Master `uv` (by Astral)**: Manage project lifecycles, virtual environments, dependencies, and lockfiles with the high-speed Rust-based toolchain.
2. **Apply 12-Factor App configuration hygiene (Factor III)**: Connect to a **real, live external API** using authenticated API keys stored in `.env`, strictly decoupled from code and protected by `.gitignore`.
3. **Manage dependencies deterministically**: Master `pyproject.toml`, lockfiles (`uv.lock`), and Semantic Versioning (SemVer) constraints.
4. **Deploy a package to PyPI (TestPyPI)**: Build standardized Source Distributions (`sdist`) and Wheels (`.whl`) with `uv build`, and securely publish a working command-line interface (`[project.scripts]`) to TestPyPI using `uv publish`.
5. **Debug like a senior engineer**: Diagnose runtime data corruptions and API payload parsing bugs using an interactive debugger without relying on `print()`.

---

## 🎲 Step 0: Find Your Assigned API Group (Deterministic Routing)

To prevent copying and guarantee diversity across the cohort, each student is assigned to one of **8 distinct live API domains** based on their EPITA login.

Open your terminal and run this command:

```bash
python3 -c "login = input('Enter your EPITA login (e.g. ismail.lachheb): ').strip().lower(); gid = sum(ord(c) for c in login) % 8; print(f'\n>>> SUCCESS: You are assigned to GROUP {gid}!')"
```

Look up your assigned group number in the table below:

| Group | Domain & Provider | Free Signup & Key URL | Env Var Key | Target Endpoint & Live Action |
| :---: | :--- | :--- | :--- | :--- |
| **0** | **Fintech & Payments**<br>*(Stripe Test Mode)* | [dashboard.stripe.com/register](https://dashboard.stripe.com/register)<br>*(Instant sandbox, no bank/business info)* | `STRIPE_API_KEY` | `GET https://api.stripe.com/v1/balance`<br>Header: `Authorization: Bearer sk_test_...`<br>*Fetches available & pending balance.* |
| **1** | **Financial Markets**<br>*(Finnhub)* | [finnhub.io/register](https://finnhub.io/register)<br>*(Instant free token)* | `FINNHUB_API_KEY` | `GET https://finnhub.io/api/v1/quote?symbol=NVDA`<br>Header: `X-Finnhub-Token: <key>`<br>*Fetches live price, high, low, open for NVDA.* |
| **2** | **Crypto Assets**<br>*(CryptoCompare)* | [min-api.cryptocompare.com](https://min-api.cryptocompare.com/)<br>*(Instant free key)* | `CRYPTOCOMPARE_API_KEY` | `GET https://min-api.cryptocompare.com/data/pricemultifull?fsyms=BTC,ETH&tsyms=USD,EUR`<br>Header: `Apikey <key>`<br>*Fetches live BTC/ETH rates, 24h market cap.* |
| **3** | **DevOps & NetOps**<br>*(GitHub REST API)* | [github.com/settings/tokens](https://github.com/settings/tokens)<br>*(Generate Classic or Fine-grained PAT)* | `GITHUB_TOKEN` | `GET https://api.github.com/user`<br>Headers: `Authorization: Bearer ghp_...`<br>`User-Agent: Epita-Student-CLI`<br>*Fetches username, public repos count, followers.* |
| **4** | **Climate & Weather**<br>*(WeatherAPI)* | [weatherapi.com/signup.aspx](https://www.weatherapi.com/signup.aspx)<br>*(Instant 1M calls/mo)* | `WEATHER_API_KEY` | `GET https://api.weatherapi.com/v1/current.json?q=Paris`<br>Query: `?key=<key>&q=Paris`<br>*Fetches temp (°C), wind (kph), condition.* |
| **5** | **Space & Astronomy**<br>*(NASA Open API)* | [api.nasa.gov](https://api.nasa.gov/)<br>*(Instant signup form)* | `NASA_API_KEY` | `GET https://api.nasa.gov/planetary/apod?api_key=<key>`<br>Query: `?api_key=<key>`<br>*Fetches Astronomy Picture of the Day metadata.* |
| **6** | **Geolocation & IP Ops**<br>*(IPInfo.io)* | [ipinfo.io/signup](https://ipinfo.io/signup)<br>*(Instant free token)* | `IPINFO_TOKEN` | `GET https://ipinfo.io/json`<br>Header: `Authorization: Bearer <key>`<br>*Fetches client IP, City, Region, Country, ASN/Org.* |
| **7** | **Media & News**<br>*(NewsAPI)* | [newsapi.org/register](https://newsapi.org/register)<br>*(Instant developer key)* | `NEWS_API_KEY` | `GET https://newsapi.org/v2/top-headlines?country=us&pageSize=5`<br>Header: `X-Api-Key: <key>`<br>*Fetches top breaking news headlines & sources.* |

---

## ⚠️ Submission Rules & Environmental Hygiene (Read Carefully!)

You will submit your deliverables as a single compressed archive (`.zip`) by email.

> 🔴 **CRITICAL PENALTY WARNING (-5 points):**
> As an engineer, you must never distribute your local virtual environment or credentials.
> - **DO NOT include `.venv/` in your submission.**
> - **DO NOT commit or include your real `.env` file or TestPyPI API token in the archive.**
> - **DO NOT include `__pycache__/` or `.pytest_cache/`.**
> 
> *Submissions containing `.venv/` or leaked secret keys will automatically receive a 5-point deduction.*

---

## 📂 Expected Project Structure

```
epita_<login>_<domain>/
├── .gitignore
├── .python-version
├── .env.template
├── pyproject.toml
├── uv.lock
├── dist/
│   ├── epita_<login>_<domain>-0.1.0-py3-none-any.whl
│   └── epita_<login>_<domain>-0.1.0.tar.gz
├── src/
│   └── epita_<login>_<domain>/
│       ├── __init__.py
│       ├── config.py
│       ├── client.py
│       └── cli.py
├── scripts/
│   ├── inspect_env.py
│   └── debug_challenge.py
└── REPORT.md
```

*(Example for login `jean.dupont` in Group 1: `epita_jean_dupont_markets`).*

---

# Lab Exercises

---

## Part 1: Modern Environment Isolation with `uv` (3.5 Points)

`uv` is an extremely fast package and project manager written in Rust, unifying Python installation, virtual environment management, package resolution, and script execution.

### 1.1 Installing `uv`
If `uv` is not yet installed on your machine:
- **macOS / Linux:**
  ```bash
  curl -LsSf https://astral.sh/uv/install.sh | sh
  source $HOME/.cargo/env
  ```
- **Windows (PowerShell):**
  ```powershell
  powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
  ```
- **macOS (Homebrew):**
  ```bash
  brew install uv
  ```

Check that `uv` is available:
```bash
uv --version
```

### 1.2 Initializing the Project
Create your library project using `uv init` (substitute `<login>` with your login and `<domain>` with your assigned domain slug):

```bash
# Example: uv init --lib epita_ismail_lachheb_fintech
uv init --lib epita_<login>_<domain>
cd epita_<login>_<domain>
mkdir -p scripts
git init
```

Notice what `uv init` created:
- `pyproject.toml`: The standard project specification.
- `.python-version`: Pins your project runtime.
- `src/epita_<login>_<domain>/`: Package source directory.

### 1.3 Creating and Inspecting the Isolated Environment
1. Create a virtual environment using `uv`:
   ```bash
   uv venv
   ```
2. Create the diagnostic script `scripts/inspect_env.py`:
   ```python
   # scripts/inspect_env.py
   import sys
   import os

   def print_section(title, data):
       print(f"\n=== {title} ===")
       if isinstance(data, list):
           for item in data:
               print(f"  - {item}")
       else:
           print(f"  {data}")

   if __name__ == "__main__":
       print_section("OS Executable Path ($PATH)", os.environ.get("PATH", "").split(os.pathsep)[:4])
       print_section("Active Python Binary (sys.executable)", sys.executable)
       print_section("Python Version", sys.version.split()[0])
       print_section("Virtual Env Prefix (sys.prefix)", sys.prefix)
       print_section("Base Prefix (sys.base_prefix)", sys.base_prefix)
       print_section("Is in Virtual Environment?", sys.prefix != sys.base_prefix)
       print_section("Search Paths for Imports (sys.path)", sys.path[:5])
   ```
3. Run the script **globally** and via **`uv run`**:
   ```bash
   python3 scripts/inspect_env.py > global_env_output.txt
   uv run python scripts/inspect_env.py > uv_env_output.txt
   ```

### 1.4 Deliverable for Part 1 (Answer in `REPORT.md`)
1. Compare `sys.executable` and `sys.prefix` between both runs. Explain how `uv run` dynamically isolates the environment without requiring you to manually run `source .venv/bin/activate`.
2. Configure `.gitignore`: add `.venv/`, `*.txt`, `dist/`, and `__pycache__/`. Commit `.gitignore` and `scripts/inspect_env.py`.

---

## Part 2: Live API Connection & 12-Factor Secret Hygiene (4.5 Points)

In this section, you will connect your project to your assigned live system while adhering to **Factor III (Config in the environment)** of the 12-Factor App methodology.

### 2.1 Adding Base Dependencies with `uv`
Add `requests` and `python-dotenv`:
```bash
uv add requests python-dotenv
```

### 2.2 Writing the Configuration Module (`config.py`)
Create `src/epita_<login>_<domain>/config.py`. Ensure it loads `.env` and validates that your domain's API key is defined:

```python
# src/epita_<login>_<domain>/config.py
import os
from dataclasses import dataclass
from dotenv import load_dotenv

# Ingest variables from local .env
load_dotenv()

@dataclass(frozen=True)
class Settings:
    app_env: str = os.getenv("APP_ENV", "development")
    api_key: str = os.getenv("API_KEY", "") # Or your assigned variable name, e.g. STRIPE_API_KEY
    debug: bool = os.getenv("DEBUG", "False").lower() in ("true", "1", "yes")

    def validate(self):
        """Defensive check: fail immediately if credentials are missing."""
        if not self.api_key or self.api_key.startswith("your_"):
            raise ValueError(
                "CRITICAL SECURITY / CONFIG ERROR: API key is not configured!\n"
                "Ensure your .env file exists and contains a valid API key."
            )

config = Settings()
```

*(Customize `api_key` to read your group's specific variable name: `STRIPE_API_KEY`, `FINNHUB_API_KEY`, `GITHUB_TOKEN`, `WEATHER_API_KEY`, `NASA_API_KEY`, `IPINFO_TOKEN`, `NEWS_API_KEY`, or `CRYPTOCOMPARE_API_KEY`).*

### 2.3 Implementing the Live API Client (`client.py`)
Create `src/epita_<login>_<domain>/client.py` tailored to your assigned group.

**Example for Group 0 (Stripe):**
```python
import requests
from epita_<login>_<domain>.config import config

def fetch_live_data() -> dict:
    config.validate()
    url = "https://api.stripe.com/v1/balance"
    headers = {"Authorization": f"Bearer {config.api_key}"}
    resp = requests.get(url, headers=headers, timeout=10)
    resp.raise_for_status()
    data = resp.json()
    avail = data.get("available", [{}])[0]
    return {
        "status": "LIVE_SUCCESS",
        "currency": avail.get("currency", "usd").upper(),
        "available_balance": avail.get("amount", 0) / 100.0,
        "livemode": data.get("livemode", False)
    }
```

**Example for Group 1 (Finnhub):**
```python
import requests
from epita_<login>_<domain>.config import config

def fetch_live_data(symbol: str = "NVDA") -> dict:
    config.validate()
    url = f"https://finnhub.io/api/v1/quote?symbol={symbol}"
    headers = {"X-Finnhub-Token": config.api_key}
    resp = requests.get(url, headers=headers, timeout=10)
    resp.raise_for_status()
    data = resp.json()
    return {
        "status": "LIVE_SUCCESS",
        "symbol": symbol,
        "current_price": data.get("c"),
        "high_of_the_day": data.get("h"),
        "low_of_the_day": data.get("l"),
        "previous_close": data.get("pc")
    }
```

*(Implement the corresponding client logic for your group according to the table in Step 0).*

### 2.4 Implementing Secret Hygiene
1. Create `.env.template` (this file **MUST** be committed to Git):
   ```ini
   # .env.template - Public configuration blueprint (NO REAL SECRETS)
   APP_ENV=development
   DEBUG=True
   # Set your group's key name here:
   API_KEY=your_dummy_api_key_here
   ```
2. Copy it to `.env` (private, untracked):
   ```bash
   cp .env.template .env
   ```
3. Edit `.env` and paste your real API key.
4. Update `.gitignore` so that `.env` can never be tracked:
   ```gitignore
   .venv/
   .env
   __pycache__/
   dist/
   *.egg-info/
   *.txt
   ```
5. Test your live connection:
   ```bash
   uv run python -c "from epita_<login>_<domain>.client import fetch_live_data; print(fetch_live_data())"
   ```

### 2.5 Deliverable for Part 2 (Answer in `REPORT.md`)
1. Run `git status` and paste the output. Confirm `.env` is omitted and `.env.template` is tracked.
2. Paste the redacted response from your live API call.
3. Why is storing secrets in environment variables superior to putting them in configuration files like `config.json` or `settings.yaml`? Give two security reasons.

---

## Part 3: Dependency Locking with `uv` & Semantic Versioning (3.5 Points)

### 3.1 Adding Direct & Development Dependencies
1. Add `rich` for terminal UI formatting:
   ```bash
   uv add rich
   ```
2. Add `pytest` as a development-only dependency:
   ```bash
   uv add --dev pytest
   ```

### 3.2 Visualizing the Dependency Tree
1. Run `uv tree`:
   ```bash
   uv tree
   ```
2. In `REPORT.md`, paste the tree output and clearly identify:
   - Your **direct dependencies**.
   - The **transitive dependencies** pulled in by `requests` and `rich`.
3. Open `uv.lock`. Explain how the cryptographic SHA-256 hashes inside `uv.lock` protect your build against supply-chain attacks.

### 3.3 Semantic Versioning (SemVer) Analysis (Answer in `REPORT.md`)
1. Explain the version ranges matched by:
   - `requests = "^2.31.0"`
   - `urllib3 = "~=2.2.1"`
   - `rich = "==13.7.1"`
2. If an upstream author releases version `3.0.0` with breaking changes, will `uv lock --upgrade` install it given `^2.31.0`? Why or why not?
3. What is the fundamental difference between `pyproject.toml` (declaration) and `uv.lock` (pinning)?

---

## Part 4: Building & Deploying the Live CLI to TestPyPI (4.5 Points)

You will now package your application as an installable Command-Line Tool and publish it live to **TestPyPI**.

### 4.1 Implementing the CLI Interface (`cli.py`)
Create `src/epita_<login>_<domain>/cli.py` to display your live API data in a styled terminal dashboard using `rich`:

```python
# src/epita_<login>_<domain>/cli.py
from rich.console import Console
from rich.panel import Panel
from rich.table import Table
from epita_<login>_<domain>.client import fetch_live_data

console = Console()

def main():
    try:
        data = fetch_live_data()
        table = Table(title=f"Live System Telemetry: {data.get('status')}")
        table.add_column("Property", style="cyan", no_wrap=True)
        table.add_column("Value", style="magenta")

        for key, value in data.items():
            table.add_row(str(key), str(value))

        console.print(Panel(table, title="EPITA Engineering Tools — Session 2", expand=False))
    except Exception as e:
        console.print(f"[bold red]Execution Error:[/bold red] {e}")

if __name__ == "__main__":
    main()
```

### 4.2 Exposing the CLI Entrypoint in `pyproject.toml`
Open `pyproject.toml` and configure the project scripts entrypoint so the tool can be invoked directly from the command line:

```toml
[project.scripts]
epita-<login>-cli = "epita_<login>_<domain>.cli:main"
```

Test running your command locally with `uv run`:
```bash
uv run epita-<login>-cli
```

### 4.3 Building Distribution Artifacts with `uv build`
Build your package:
```bash
uv build
```
Verify that `dist/` contains:
- `epita_<login>_<domain>-0.1.0-py3-none-any.whl` (Built distribution: Wheel)
- `epita_<login>_<domain>-0.1.0.tar.gz` (Source distribution: sdist)

### 4.4 Publishing to TestPyPI with `uv publish`
1. Create a free account on **TestPyPI**: [https://test.pypi.org/account/register/](https://test.pypi.org/account/register/)
2. Verify your email address.
3. In TestPyPI, go to **Account Settings** > **API tokens** > **Add API token**:
   - Token name: `epita-tp2-token`
   - Scope: `Entire account`
4. Copy the token (`pypi-AgENdGVzdC5weXBpLm9yZw...`).
5. Publish your package to TestPyPI:
   ```bash
   uv publish --publish-url https://test.pypi.org/legacy/ --token "YOUR_TESTPYPI_TOKEN"
   ```

### 4.5 Live Verification of Your Published Package
Verify that your deployed package can be installed and executed from anywhere:

```bash
# In a fresh shell or temporary directory:
cd /tmp
uv run --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ epita-<login>-cli
```

### 4.6 Deliverable for Part 4 (Answer in `REPORT.md`)
1. **Public URL:** Provide the direct link to your published package on TestPyPI (e.g. `https://test.pypi.org/project/epita_<login>_<domain>/`).
2. Why are Wheels (`.whl`) faster and safer to install on client machines than Source Distributions (`.tar.gz`)?

---

## Part 5: Professional Debugging Challenge (No `print()`!) (4.0 Points)

> 🛑 **MANDATORY RULE:**
> You are strictly forbidden from adding any `print()` statements to find and fix this bug.
> You must use an **interactive debugger**:
> - VS Code Debugger (`Run and Debug` with `F5`), or
> - PyCharm Debugger, or
> - Interactive CLI debugger: `uv run python -m pdb scripts/debug_challenge.py`.

### 5.1 The Broken Script
Create `scripts/debug_challenge.py`:

```python
# scripts/debug_challenge.py
"""
EPITA CM2 Debugging Challenge: API Payload & Batch Aggregator.
This script contains runtime type mismatches and faulty aggregations.
DO NOT USE print()! Use your interactive debugger.
"""

def parse_api_record(raw_entry: str) -> dict:
    # Simulates records: "record_id,entity_name,raw_volume,status,weight_multiplier"
    parts = raw_entry.strip().split(",")
    return {
        "id": parts[0],
        "entity": parts[1],
        "volume": parts[2],        # <-- Hint: What runtime type is this?
        "status": parts[3],
        "weight": parts[4]          # <-- Hint: What runtime type is this?
    }

def compute_weighted_metric(volume, weight):
    return volume * weight

def aggregate_telemetry(payload: list[str]) -> dict:
    total_volume = 0
    successful_entries = 0
    high_impact_count = 0

    for idx, row in enumerate(payload):
        record = parse_api_record(row)

        if record["status"] == "SUCCESS":
            # Set a conditional breakpoint here when idx == 2
            impact = compute_weighted_metric(record["volume"], record["weight"])

            if impact > 500:
                high_impact_count += 1

            total_volume += record["volume"]
            successful_entries += 1

    avg_volume = total_volume / successful_entries if successful_entries > 0 else 0
    return {
        "successful_entries": successful_entries,
        "total_volume": total_volume,
        "avg_volume": avg_volume,
        "high_impact_count": high_impact_count
    }

if __name__ == "__main__":
    api_payload = [
        "REC_001,service_alpha,120,SUCCESS,2",
        "REC_002,service_beta,0,FAILURE,1",
        "REC_003,service_gamma,450,SUCCESS,3",
        "REC_004,service_delta,80,SUCCESS,1",
        "REC_005,service_epsilon,1500,PENDING,4"
    ]
    
    # Expected output for SUCCESS records (REC_001, REC_003, REC_004):
    # - total_volume: 120 + 450 + 80 = 650.0
    # - avg_volume: 650 / 3 = 216.67
    # - high_impact_count (impact > 500):
    #     REC_001: 120 * 2 = 240
    #     REC_003: 450 * 3 = 1350 (>500)
    #     REC_004: 80 * 1 = 80
    #     Total high impact = 1
    
    result = aggregate_telemetry(api_payload)
    print("Aggregate Result:", result)
```

If you run this code unmodified, Python crashes with a `TypeError` or performs invalid string concatenations.

### 5.2 Debugging Protocol
1. Open `scripts/debug_challenge.py` in your IDE.
2. Set a **Breakpoint** on line 31 (`impact = compute_weighted_metric(...)`).
3. Set a **Conditional Breakpoint** configured to trigger only when `idx == 2`.
4. Launch the debugger.
5. In the debugger:
   - Check the **Variables / Scope window**: what are the Python data types of `record['volume']` and `record['weight']`?
   - Step into (`F11`) `compute_weighted_metric`.
   - Inspect the **Call Stack** window showing `<module>` -> `aggregate_telemetry` -> `compute_weighted_metric`.
6. Fix the bug cleanly in `scripts/debug_challenge.py`.

### 5.3 Deliverable for Part 5 (Answer in `REPORT.md`)
1. **Diagnosis:** What were the exact runtime types of `record["volume"]` and `record["weight"]`, and why did the original code fail?
2. **Call Stack description:** Describe the frames visible in the Call Stack when paused at the breakpoint.
3. **The Fix:** Provide the code diff or snippet of your correction.
4. **Validation:** Run the fixed script with `uv run python scripts/debug_challenge.py` and paste the verified output dictionary.

---

# 📝 Grading Rubric (Barème / 20 points)

| Section | Evaluation Criteria | Points |
| :--- | :--- | :---: |
| **Part 1: Environment & `uv` Mechanics** | Working `uv` setup; detailed answers in `REPORT.md` on `$PATH` and `sys.prefix`; proper `.gitignore` configuration. | **3.5 pts** |
| **Part 2: Live API & 12-Factor Hygiene** | Live API successfully integrated; `.env.template` properly tracked; `.env` strictly untracked; defensive validation in `config.py`. | **4.5 pts** |
| **Part 3: Dependency Hygiene & SemVer** | Clean `uv.lock` & `pyproject.toml`; `uv tree` analysis in `REPORT.md`; accurate answers to SemVer scenarios. | **3.5 pts** |
| **Part 4: TestPyPI Package Deployment** | Valid Wheel and sdist built via `uv build`; working `[project.scripts]` CLI; **active public link to TestPyPI package**. | **4.5 pts** |
| **Part 5: Professional Debugging** | Quality of the investigation report in `REPORT.md` (no prints used, variable inspection, Call Stack explanation, working fix). | **4.0 pts** |
| **Hygiene Penalty** | **IMMEDIATE DEDUCTION:** Submitting `.venv/`, committing personal `.env`, or leaking API tokens in the archive. | **-5.0 pts** |
| **TOTAL** | | **/ 20 pts** |

---

# ✉️ Email Submission Guidelines

- **Recipient:** `ismail.lachheb@epita.com`
- **Subject line:** `[EPITA][ENG_TOOLS][TP2] LASTNAME_Firstname`  
  *(Example: `[EPITA][ENG_TOOLS][TP2] DUPONT_Jean`)*
- **Attachment:** A single zip file named `TP2_LASTNAME_Firstname.zip`.

### Preparing your archive:
Run the following commands to safely package your project without leaking environments or tokens:

```bash
# Check git status to ensure untracked temporary files are clean
git status

# Move to the parent directory
cd ..

# Create the submission zip excluding virtualenvs, secrets, and caches
zip -r TP2_LASTNAME_Firstname.zip epita_<login>_<domain> \
  -x "*/.venv/*" \
  -x "*.env" \
  -x "*/__pycache__/*" \
  -x "*/.pytest_cache/*" \
  -x "*/.git/*"
```

Verify your zip archive before sending:
```bash
unzip -l TP2_LASTNAME_Firstname.zip | grep -E ".venv|\.env$"
```
*(If the command outputs nothing, your submission is clean and ready to send!)*
