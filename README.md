
## Requirements

- Python 3.11+
- A GitHu

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

ariables: `DOC_WORKERS` and
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
