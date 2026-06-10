# Resume Sourcing Pilot

A Python pipeline that discovers publicly available résumés and professional
profiles of U.S. **cleared** and **cleared-adjacent** talent, then de-duplicates
them into a single, auditable dataset.

The project implements the four deliverables of the pilot contract:

| Deliverable | What it does | Where it lives |
|-------------|--------------|----------------|
| **1. Search-API discovery** | Issues keyword queries against a SERP provider, extracts direct résumé links (PDF/DOCX/DOC/HTML/TXT), downloads them, and preserves the raw files. | `sources/serpapi/`, `sources/web/` |
| **2. GitHub profile extractor** | Uses the GitHub REST API to find users by clearance/employer/location/role signals and captures their profile fields and résumé repositories. | `github/`, `sources/code_hosts/github.py` |
| **3. Personal-site scraper** | Shallow, GitHub chained check of each discovered personal site (`/resume`, `/cv`, `/about`, …) for directly downloadable docs. No login, no headless rendering. | `sources/personal_site.py` |
| **4. LinkedIn feasibility memo** | A written risk/feasibility memo plus a compliant public-SERP discovery path (`site:linkedin.com/in`). No direct scraping. | `docs/LINKEDIN_FEASIBILITY_MEMO.md`, `sources/serpapi/linkedin.py` |

A lightweight filtering and de-duplication step runs across all sources to remove
job postings, templates, course material, duplicate URLs, duplicate files and broken links.

## How it works

The pipeline is deliberately **search-broad, filter-narrow**:

1. **Build queries.** Keyword buckets from `config/search_task.yaml`
   (clearance, employers, locationsroles etc) are combined into tiered search engine dorks and GitHub query sets.
2. **Run sources in parallel.** Each source runs in its own thread, so a source
   sleeping on a rate limit never blocks the others.
3. **Classify every hit.** SERP results are classified before becoming records 
   only personal profiles and resume pages become people, and only genuine resume
   docs are downloaded. Job boards, company pages, and government PDFs are
   dropped.
4. **Parse documents.** Downloaded PDFs/DOCX are text-extracted so the résumé
   filter inspects real content (Experience / Education / Skills, contact details),
   not just the filename. A non-resume blacklist vetoes documents *about* resume,
   SOPs, legal filings, and templates.
5. **De-duplicate people.** Records from all sources are merged into one canonical
   `PersonRecord` per person. **GitHub is the reference source**: when another
   source matches a GitHub person, the GitHub URL stays the primary link and the
   rest are appended.
6. **Write output.** A clean client deliverable plus optional analyst artifacts
   (see [Output](#output)).

## Requirements

- Python 3.11+
- A GitHub personal access token
- A SerpApi key for `serpapi`, `bing`, and `linkedin` sources

## Setup

```powershell
# from the project root
.\.venv\Scripts\python -m pip install -r requirements.txt
Copy-Item .env.example .env
# edit .env: set GITHUB_TOKEN=<your PAT> and, if you have one, SERPAPI_KEY=<key>
```

## Quick start

The main entry point is the `search` subcommand. Put `src` on the path first.

```powershell
$env:PYTHONPATH = "src"

# Preview the queries that would run, no network calls:
.\.venv\Scripts\python -m src.cli search --all --dry-run

# Run a single source, or a combination, or the full reliable set:
.\.venv\Scripts\python -m src.cli search --source linkedin --query config\search_task.yaml
.\.venv\Scripts\python -m src.cli search --sources serpapi,bing,github --query config\search_task.yaml
.\.venv\Scripts\python -m src.cli search --all --query config\search_task.yaml
```

Python API:

```python
from src.pipeline import SearchPipeline
result = SearchPipeline(task).run_sources(["github", "linkedin", "serpapi"])
```

## Sources

`github`, `gitlab`, `serpapi` (Google, primary SERP), `bing` (SerpApi Bing
engine), `linkedin`, `duckduckgo`, `searxng`.

- `serpapi`, `bing`, and `linkedin` read `SERPAPI_KEY` from the environment or local `.env`.
- `github` works anonymously but is far more productive with a token.
- `gitlab` works anonymously; `GITLAB_TOKEN` raises its limits.
- `searxng` needs `SEARXNG_BASE_URL` and disables itself cleanly if unset.
- `duckduckgo` uses the free `ddgs` backend, which throttles heavily and is **not**
  part of `--all`; add it explicitly only if needed.

LinkedIn is the single strongest source of relevant people. GitHub provides the
largest raw volume.

## Reaching the 1000 record target

1000 combined records as a best effort goal. The harvest loop runs source rounds with a widening query budget until the target, the round cap, or the SerpApi budget is reached.

```powershell
cd C:\Users\1\PycharmProjects\parser
$env:PYTHONIOENCODING = "utf-8"; $env:PYTHONPATH = "$PWD\src"
$env:GITHUB_FAIL_FAST_ON_RATE_LIMIT = "1"; $env:GITHUB_SOURCE_CAPTURE_DOCS = "0"
$env:GITHUB_SOURCE_MAX_RESULTS = "400"; $env:GITHUB_SOURCE_MAX_QUERIES = "60"

.\.venv\Scripts\python.exe -m src.cli search `
  --sources linkedin,serpapi,bing,github `
  --query config\search_task.yaml `
  --target 1000 --max-rounds 8 --serp-budget 1500 `
  --serp-results-per-query 20 --serp-tier all --serp-mode all `
  --jobs 5 --no-personal-sites --full-output --out output\run_1000
```

What matters for yield:

- **Breadth of `config/search_task.yaml`.** Every additional employer, location,
  or agency multiplies the number of distinct dorks and therefore distinct people.
  This is the cheapest lever — widen the lists before anything else.
- **`serp-results-per-query 20`** doubles the people returned per query.
- **`target` + `max-rounds` + `--serp-budget`** drive the harvest loop. Raise
  the budget and rounds if a run plateaus before the target.
- **`no-personal-sites`** skips the slowest stage during volume collection; run a
  final pass without it to pull the actual résumé files.
- **GitHub coverage** via `GITHUB_SOURCE_MAX_RESULTS` / `GITHUB_SOURCE_MAX_QUERIES`.

## Key parameters

| Flag | Purpose |
|------|---------|
| `source` / `sources` / `all` | One source, a comma list, or the reliable default set. |
| `query` | Path to the search-task YAML (default `config/search_task.yaml`). |
| `target` | Harvest until N unique people are found (`0` = single pass). |
| `max-rounds` / `serp-budget` | Harvest round cap and total SerpApi query budget. |
| `max-results` / `max-queries` | Per-round people cap and query cap. |
| `serp-results-per-query` | Organic results fetched per dork (default 15; 20 for more yield). |
| `serp-tier` / `serp-mode` | Dork tiers (`t1/t2/t3`) and modes (`docs/pages/personal/profiles/employers`). |
| `jobs` | Parallel source workers (`0` = auto, `1` = sequential). |
| `no-personal-sites` | Skip the personal-site scraper (the slowest stage). |
| `no-docs` | Skip document capture entirely. |
| `full-output` | Also write the analyst artifacts (JSONL/CSV/per-source). |
| `clean-raw` / `--no-clean-raw` | Clear or keep `output/raw/` before a run. |
| `mock <dir>` | Run offline against saved fixtures (no network). |
| `dry-run` | Print the queries and exit. |

Performance tuning via environment variables: `DOC_WORKERS` and
`PERSONAL_SITE_WORKERS` (document parsing and site scraping fan-out, default 8
each); `GITHUB_REQUIRE_US` (keep the US-relevance gate on, recommended).

## Output

By default a run writes a clean client deliverable to `output/last_result/`:

- `people.xlsx` — relevant people, trimmed columns, ranked by confidence.
- `documents.xlsx` — résumé documents with audit fields.
- `resumes/` — the résumé files themselves.
- `summary.json` — machine-readable run metrics.

`output/raw/` keeps every downloaded file in its original format. Add
`--full-output` for the per-record schema artifacts required by the contract:
`people.jsonl` / `people.csv` (profile records) and `documents.jsonl` /
`documents.csv` (document records), plus `people_simple.json` and per-source
workbooks.

For handover, run with `--full-output` and archive the output folder:

```powershell
Compress-Archive -Path output\run_1000\* -DestinationPath delivery.zip
```


## Tests

```powershell
$env:PYTHONPATH = "src"
.\.venv\Scripts\python -m pytest unit_test -q
```

The suite is offline (no network) and covers the core filtering, scoring,
clearance, and identity logic.

## Docker

See `DEPLOY.md` for the containerized build and run.
