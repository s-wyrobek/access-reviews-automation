# Access Reviews Automation

> Production system automating monthly IT access reviews across 25+ systems

![Python](https://img.shields.io/badge/Python-3.8+-blue)
![Status](https://img.shields.io/badge/status-production-brightgreen)
![Systems](https://img.shields.io/badge/systems-25+-orange)
![License](https://img.shields.io/badge/license-MIT-green)

---

## Problem

In any organization with hundreds of people, there's a recurring need to verify who has access to which tools — and whether they still should. Without automation, this means exporting spreadsheets from a dozen systems, opening an HR roster, and manually comparing emails row by row.

**In practice: 3 days of work across 3 people, every month. Error-prone and hard to audit.**

---

## Solution

A script that takes a CSV export from any tool, compares it against the license inventory (Snipe-IT) and HR database, and produces a formatted Excel report ready for review. One command per system — or one command for everything.

```
             ┌─────────────┐
  CSV export │             │  XLSX report
  ──────────►│  compare.py │──────────────►  Review /
             │             │
             └──────┬──────┘
                    │
          ┌─────────┼──────────┐
          ▼         ▼          ▼
      Snipe-IT    HR data   Exceptions
     (licenses)  (active)   (config)
```

**Result: review time reduced from 3 days (3 people) → half a day (1 person). ~95% reduction in effort across thousands of license checks per month.**

---

## Usage

```bash
# Single system
python3 compare.py github

# Predefined system group
python3 compare.py atlassian-stack

# All systems at once
python3 compare.py --all

# Dry run — does not overwrite production output
python3 compare.py github --test

# Monthly summary report
python3 generate_monthly_summary.py
python3 generate_monthly_summary.py 202601   # specific month
```

---

## Project Structure

```
access-reviews/
│
├── compare.py                   # Main entry point
├── systems_config.py            # Per-system config (columns, filters, mappings)
├── config.py                    # Data folder paths (fill in for your environment)
├── core_functions.py            # Internal logic (Excel output, normalization, HR check)
├── generate_monthly_summary.py  # Monthly summary across all systems
└── run_all.sh                   # Run all systems + generate report in one command
```

### Input / Output Layout

```
data/
│
├── Snipe Export/
│   ├── Licenses.xlsx              # Date-stamped sheets (YYYY.MM.DD) — auto-picks latest
│   └── Exceptions-Snipe.xlsx     # License exceptions (service accounts, global or per-system)
│
├── HR Export/
│   ├── HR-main.xlsx               # Main employee database (date-stamped sheets)
│   ├── HR-secondary.xlsx          # Second entity / external contractors base
│   ├── Exceptions-HR-Internal.xlsx  # Employees without a full HR record
│   └── Exceptions-HR-External.xlsx  # External collaborators
│
├── Mappings/
│   └── GitHubMapping.xlsx         # GitHub login → email → full name
│
├── Exports {System}/              # One folder per system — script auto-picks latest CSV
│
└── Reviews/
    ├── {System}_Reviews.xlsx      # Per-system results (date-stamped sheets)
    └── Summary_Report.xlsx        # Monthly aggregate report
```

---

## Report Output

Each run appends a new date-stamped sheet to `{System}_Reviews.xlsx`.

| Column | Description |
|---|---|
| Email | User email address |
| User | Full name |
| GitHub Username | Login *(GitHub systems only)* |
| Role (System) | Role in the system (Owner / Admin / Member etc.) |
| Role (Snipe) | License name in Snipe-IT |
| System–Snipe Match | `True` / `False` / `TRUE - Exception` / `No mapping` |
| HR Match | `YES` / `NO` / `YES - Exception \| Internal` / `YES - Exception \| External` |
| Discrepancy Description | Auto-generated description of the issue |
| Actions Taken | Filled in manually after review |

Report is color-coded and sorted — discrepancies at the top, OK at the bottom.

---

## Adding a New System

Adding a system is a dozen lines in `systems_config.py` — no changes to core logic:

```python
'new_system': {
    'name': 'NewSystem',
    'export_folder': NEW_SYSTEM_FOLDER,
    'snipe_license_name': '[NewSystem]',
    'email_column': 'Email',

    # optional
    'role_column': 'Role',
    'filter_by_status': True,
    'status_column': 'Status',
    'status_value': 'Active',
    'exclude_license_patterns': ['Copilot'],
}
```

For systems without emails in the export (e.g. GitHub) — use a login → email mapping:

```python
'name_email_mapping': '/path/to/GitHubMapping.xlsx',
'name_email_mapping_name_col': 'GitHub Username',
'name_email_mapping_sheet': 'Username Mapping',
```

---

## Example Terminal Output

```
============================================================
ACCESS REVIEW: GitHub [org]
============================================================

1. Loading source data...

   [Snipe-IT]
   >> Auto-selected latest sheet: 2026.01.01
   OK Loaded 300+ rows

   [HR – exceptions]
   Loaded: Exceptions-HR-Internal.xlsx → 10+ exceptions
   Loaded: Exceptions-HR-External.xlsx → 5+ exceptions
   Loaded 15+ exceptions total

2. Filtering Snipe by license 'GitHub'...
3. Latest export from 'Exports GitHub/'...
   OK Latest export: github_org_2026-01-15.csv

   GitHub [org]: 80+ users
   Snipe-IT:     85+ users (with license)

============================================================
SUMMARY
============================================================
Total users: 85+

System–Snipe match:
  OK:           82+
  Mismatch:      3

HR match:
  YES:                        65+
  YES - Exception | Internal: 10+
  YES - Exception | External:  5+
  NO:                          0
  N/A:                         3
============================================================

DONE! Report: Reviews/GitHub [org]_Reviews.xlsx
```

---

## Key Architectural Decisions

**File-based over API-first** — cloud storage as a shared file repository. Zero additional infrastructure, works immediately, no API credentials required per system. CSV exports are a universal interface that works even for systems with no public API.

**Two-tier exception system** — distinguishes between "not in HR because exception" vs "not in HR because problem." HR exceptions are split into Internal and External — visible directly in the report so the reviewer knows *why* a person is marked OK and *where* that exception comes from.

**Auto-selection of latest sheet/file** — Snipe and HR data is exported periodically as new date-stamped sheets. The script picks the latest automatically — no config changes needed between reviews.

**Per-system config, shared logic** — all system-specific details (column names, status filters, mappings) are isolated in `systems_config.py`. The comparison logic in `compare.py` is system-agnostic — it handles GitHub, Jira, Slack, and any future system identically.

**Intentionally manual trigger** — the bottleneck is upstream: CSV exports from some systems require manual download from admin panels. Automating the trigger without automating data sourcing would create a false sense of automation. Next step: replace manual exports with API-based fetching via n8n for systems that support it.

---

## Stack

| | |
|---|---|
| Language | Python 3.8+ |
| Data processing | pandas, openpyxl |
| Input format | CSV (system exports), XLSX (HR, Snipe, exceptions) |
| Output format | Excel with conditional formatting and sorting |
| Storage | Cloud storage (local mount) |
| Versioning | Git |

---

## Results

| Before | After |
|---|---|
| 3 days of work across 3 people | ~half a day for 1 person |
| Manual spreadsheet comparison | Deterministic, repeatable output |
| No history or audit trail | Every review saved in a date-stamped sheet |
| One system at a time | 25+ systems with a single command |

**~95% reduction in effort. Thousands of license checks automated per monthly cycle.**

---

## My Role

**Concept author and system architect.**

I identified the operational problem, designed the data structure and validation flow, made the key architectural decisions (file-based approach, exception system, auto-selection), deployed the system to production, and have been iterating on it based on real usage.

Code implementation was developed with AI assistance — AI-assisted coding is today's standard, just as seniors use IDEs and documentation. The core skill doesn't change: knowing **what** to build, **how** to architect it, and **whether the result is correct**. All of those decisions were mine.

---

> **Note:** This tool is designed around a file-based workflow. You'll need to provide your own CSV exports and HR/Snipe data in the expected folder structure. See the layout above.
