# GSA BLS MCP Server — Technical Due-Diligence Assessment

> **Repository:** `gsa_mcp_hackathon` | **Branch:** `main` | **Assessed:** November 2024
> **Analyst:** Senior Software Architect / Engineering Manager

---

## 1. Executive Summary

This repository contains a **Model Context Protocol (MCP) server** built in Python (FastMCP 4.x) that exposes Bureau of Labor Statistics (BLS) public time-series data to AI agents. The server was developed during a GSA hackathon and successfully deployed to **IBM watsonx Orchestrate**, achieving a **72.7% correctness rate** on a 30-question benchmark dataset — a 79% improvement over the no-tool LLM baseline (40.7%).

The project is in a **late-Beta / early-Production-Pilot state**. It delivers genuine business value as a production artifact: 10 MCP tools covering 12 BLS surveys, a curated flagship series table (29 BLS top-picks + 100+ additional series), intent-parsing and ambiguity resolution, structural series ID build/decode/validate, local reference-data catalog, and a fully operational LLM-as-judge evaluation harness. All 915 tests pass and the codebase is ruff-clean.

The primary maturity gaps are: (1) the package was never renamed from the template name `example_server`; (2) no code coverage tooling is configured; (3) the flagship series table is manually curated and limited to flagship/top-picks data; (4) the disambiguation engine has limited geographic coverage; (5) no authentication layer exists on the HTTP transport; (6) one unused runtime dependency (`pandas`) is declared. The road to production is well-defined and achievable in 3–6 months.

| Metric | Value |
|---|---|
| Tests passing | 915 |
| Benchmark correctness | 72.7% |
| BLS surveys covered | 12 |
| MCP tools registered | 10 |

---

## 2. Repository Maturity Scorecard

| Category | Score | Key Evidence |
|---|---|---|
| Project Purpose & State | ★★★★☆ 4/5 | Clear objectives, working deployment, benchmark results. Package name not updated from template. |
| Architecture | ★★★★☆ 4/5 | Clean layered design (tools/catalog/schema/ambiguity/utils). One minor concern: `search_series.py` (559 LoC) has per-survey hardcoded hint tables that duplicate logic. |
| Code Quality | ★★★★☆ 4/5 | Ruff-clean, typed, well-documented. `bls_flagship_series.py` (2215 LoC) is a pure data file; `ambiguity.py` (653 LoC) could be split. Unused `example_tool.py` not removed. |
| Testing & Reliability | ★★★★☆ 4/5 | 915 tests, 2.04s run, live tests gated. No coverage measurement, no property/fuzz tests, missing negative-year and concurrent-request tests. |
| Security | ★★★★☆ 4/5 | SSRF protections, redirect disabled, no key leakage, SecretStr, path validation. No auth on HTTP transport — acknowledged as hackathon-only posture; must be addressed before ATO. |
| DevOps & Operational Readiness | ★★★☆☆ 3/5 | CI pipeline (lint + test), Dockerfile, cloud.gov manifest, IBM + Databricks deploy kits. No coverage gate, no mutation testing, no image scan, no canary/blue-green, no structured logging. |
| Documentation Quality | ★★★★★ 5/5 | Exceptional: README, QUICKSTART, SECURITY, 11 plan docs, full technical handoff doc (`gsa_hackathon_technical_handoff.md`), architecture diagrams (Mermaid), submission deck + evaluation doc. |
| Dependencies & Technology | ★★★★☆ 4/5 | Modern stack (Python 3.11+, FastMCP 4.x, uv, pydantic-settings, httpx). `pandas` in prod deps but not imported anywhere in `src/`. Lockfile present. Dev deps are heavy (Arize Phoenix ≈ 40+ transitive packages). |
| Business & Product Alignment | ★★★★☆ 4/5 | Strong: BLS public data is a clear federal data-access use case. Benchmark shows 79% lift over baseline. Geographic ambiguity is the main coverage gap for real-user queries. |

**Overall Maturity: Late Beta / Production Pilot — 35/45 (78%)**

---

## 3. Key Findings

### 3.1 Strengths

- **One-tool-per-file architecture** (`src/example_server/tools/`) is extensible and scannable; each tool exposes a single `register(mcp)` function and is independently testable.
- **Security-first HTTP client** (`utils.py:fetch_bls`/`fetch_json`): redirects disabled, `trust_env=False`, response-size cap (5 MB), path traversal/SSRF prevention, API key never logged, structured error sanitization. Backed by 18 regression tests in `test_http_security.py`.
- **Offline-first design**: 9 of 10 tools require no network call. Only `get_series_data` hits the BLS API, making the server testable without network access and highly resilient to BLS API downtime.
- **BLS series ID structural engine** (`bls_series_schema.py`): covers 13 survey prefixes (SM, CE, LA, JT, CU, LN, PR, MP, CW, WP, EI, CI, PC) with full segment layout, catalog lookup, variable-length handling. Build/decode/validate are all backed by 130+ unit tests.
- **Ambiguity detection pipeline** (`ambiguity.py`): intent parsing → candidate discovery → eligibility filter → safe-default application → clarification generation, all local. This is the key hallucination-prevention layer.
- **Evaluation harness is production-grade**: Phoenix traces, two LLM-as-judge evaluators (correctness + scope), 30-question verified dataset, multi-run aggregation, A/B prompt testing. Results are reproducible and stored in version control.
- **Pydantic-validated BLS request model** (`models.py:BLSSeriesRequest`): enforces series ID format (regex), 4-digit year range, v1/v2 limit separation, start ≤ end, max 20-year span — all before any network call.
- **Documentation exceptionally well written**: the technical handoff document alone captures architecture, security, tradeoffs, and hard-won lessons in a form that onboards a new engineer in under an hour.

### 3.2 Issues and Gaps

> **Package identity** — The module is still named `example_server` throughout source, tests, Dockerfile, and manifest. This is a cosmetic but meaningful issue for publication: a server advertised as `BLS Public Data Server` (in `app.py`) shipping as `example-server` (in `pyproject.toml`) creates confusion and blocks proper packaging/attribution.

> **Unused `example_tool.py`** — The template's placeholder tool (`src/example_server/tools/example_tool.py`, 106 LoC) is not registered in `tools/__init__.py` but is not removed. It triggers a guard assertion in `test_server.py` (`assert "example_search_datasets" not in tool_names`). The file should be removed or its absence explained.

> **Undeclared runtime dependency** — `pandas>=3.1.1` is listed in `pyproject.toml [project].dependencies` but is never imported in any file under `src/`. It is imported in the Jupyter notebook (`bls_api_tester_aw.ipynb`). This adds ~40 MB to the production image unnecessarily. Conversely, `pandas` is absent from `requirements.txt`, creating a split between the two dependency manifests.

> **No code coverage measurement** — CI runs 915 tests but does not measure or gate on coverage (`pytest-cov` is not in dev deps; no `--cov` flag in CI). The `answer_bls_question.py` pipeline (531 LoC, the most complex path) has limited unit coverage of the clarification-then-retry path.

> **Geographic ambiguity is shallow** — `ambiguity.py` resolves geography only to a binary `national`/`state`/`metro` type. State names map to SM/LA area codes via the catalog, but the mapping is not automated — it depends on keyword-matching in `search_series.py`. A query like "unemployment in Miami-Fort Lauderdale" would not auto-resolve to the correct LA area code.

> **No authentication on HTTP transport** — `SECURITY.md` explicitly acknowledges this as a hackathon-only posture. The `/mcp` endpoint served over streamable-HTTP is unauthenticated. Any cloud deployment for non-hackathon use requires an auth layer before the endpoint, an ATO review, and network-level egress controls.

> **`search_series.py` complexity** — At 559 LoC it is the largest tool file and contains 12 per-survey private functions (`_search_sm`, `_search_ce`, …) each with hand-written hint dictionaries (`_SURVEY_HINTS`, `_SEASONAL_HINTS`, etc.) that duplicate logic from the catalog and `bls_series_schema.py`. These should be driven by schema metadata rather than hardcoded strings.

> **Benchmark correctness plateau** — The 5-run average of 72.7% is consistent but has not improved across runs with the v2 prompt. The 27.3% failure rate is concentrated in queries requiring precise date-bound values (BLS revises historical data) and multi-survey comparative questions. The 30-question dataset is small; reliability at scale is unknown.

---

## 4. Risks & Technical Debt

| Risk | Severity | File(s) | Mitigation |
|---|---|---|---|
| No auth on HTTP endpoint | **Critical** | `app.py`, `Dockerfile` | Add OAuth2/IAM token-verification middleware or deploy behind an API gateway before any non-hackathon use. Required for ATO. |
| BLS v1 daily limit (25 req/day) | **High** | `config.py`, `utils.py` | Register and configure `BLS_API_KEY` for v2 (500 req/day). No rate-limit enforcement or backoff logic exists; concurrent agent sessions could exhaust the daily quota. |
| Flagship table is manually curated | **High** | `bls_flagship_series.py` | BLS publishes series additions/removals. Add a CI check that cross-references the BLS series catalog API and flags unlisted flagship IDs. |
| Local-cache staleness | **High** | `local-cache/*.tsv` | ~150 reference files were downloaded manually. No automated refresh. BLS changes area and industry codes on annual benchmarks. Add a Makefile / script for reproducible refresh and record provenance dates. |
| Package name never updated | **Medium** | `pyproject.toml`, all source files | Rename `example_server` → `bls_mcp` (or equivalent) before PyPI publish or Docker Hub push. |
| Unused `pandas` in prod deps | **Medium** | `pyproject.toml`, `requirements.txt` | Move `pandas` to the `[dev]` group (notebook usage only). Removes ~40 MB from production image and fixes the split with `requirements.txt`. |
| No rate-limit or retry logic | **Medium** | `utils.py:fetch_bls` | BLS API returns HTTP 429 when limits are exceeded. Add exponential-backoff retry (tenacity or manual) and surface quota exhaustion as a structured error with `retry_after`. |
| No coverage gate in CI | **Medium** | `.github/workflows/ci.yml` | Add `pytest-cov` with `--fail-under=85`. The `answer_bls_question` clarification path and the geographic routing in `search_series` are likely under-covered. |
| Python 3.11 pinned in CI, 3.13 in `.python-version` | **Low** | `ci.yml`, `.python-version` | CI installs 3.11, but the local venv uses 3.13 (pycache shows `cpython-313`). Align CI matrix to test both 3.11 and 3.13 to catch behavioral differences. |
| Stale plan documents in root | **Low** | 11 `*.md` plan files in root | Move to `docs/plans/` or archive directory. Root clutter creates cognitive noise for new contributors. |

---

## 5. Prioritized Improvement Opportunities

| # | Improvement | Horizon | Priority | Effort | Benefit / Risk if skipped |
|---|---|---|---|---|---|
| 1 | Add authentication middleware to HTTP transport — OAuth2/IAM bearer token verification before `/mcp` | Quick Win | **Critical** | Medium | Required for any non-hackathon cloud deploy. Without it, the BLS API key and all data are publicly accessible. |
| 2 | Remove `pandas` from prod deps; fix `requirements.txt` split | Quick Win | **High** | Low | Reduces image size by ~40 MB, fixes dependency manifests. Zero risk if removed. |
| 3 | Remove or move `example_tool.py`; rename package to `bls_mcp` | Quick Win | **High** | Low | Removes dead code, makes package identity unambiguous, enables proper publishing. |
| 4 | Add pytest-cov to CI with 85% threshold; cover the clarification-then-retry path in `answer_bls_question` | Quick Win | **High** | Low | Prevents regressions in the most complex pipeline path. Required for production confidence. |
| 5 | Register `BLS_API_KEY` and document v2 as the production default; add HTTP 429 retry with exponential backoff | Quick Win | **High** | Low | Prevents daily quota exhaustion (25 req/day v1 → 500 req/day v2). Concurrent sessions will hit the v1 limit within minutes of real use. |
| 6 | Align CI Python version matrix to test 3.11 and 3.13; fix the `.python-version` / CI mismatch | Quick Win | **Medium** | Low | Prevents silent regressions when the runtime environment shifts. |
| 7 | Automate local-cache refresh: Makefile target that downloads BLS reference files from `download.bls.gov/pub/time.series/` with provenance headers | Short-Term | **High** | Medium | Current cache was downloaded manually; staleness date unknown. BLS annual benchmarks change area and industry codes. Stale codes cause silent wrong-label responses. |
| 8 | Expand benchmark dataset to 100+ questions covering edge cases: geographic ambiguity, multi-survey comparison, out-of-range dates, NSA vs SA edge cases | Short-Term | **High** | Medium | 30-question dataset is insufficient to measure production reliability. Plateau at 72.7% cannot be diagnosed without more failure data. |
| 9 | Refactor `search_series.py`: replace per-survey hardcoded hint dicts with metadata-driven dispatch from `bls_series_schema.py` | Short-Term | **Medium** | Medium | Removes 300+ lines of duplicated hint tables. New survey coverage requires only schema + catalog entries rather than a new `_search_XY` function. |
| 10 | Add structured logging with explicit allowlist fields (no request bodies or series values in logs) | Short-Term | **Medium** | Medium | Current logging is sparse. Operational debugging in cloud is severely hampered without it. |
| 11 | Automate geographic area code resolution: map plain-name geographies (e.g. "Miami metro") to BLS LA/SM area codes via the catalog at intent-parse time | Medium-Term | **High** | High | Geographic queries are the most common class of real-world BLS questions. Current system handles national/state well but metro areas require manual code lookup. |
| 12 | Add multi-survey comparison capability to `answer_bls_question`: allow fetching two series in one call and returning both for comparative analysis | Medium-Term | **High** | Medium | Many federal data questions compare e.g. CPI vs. wage growth. Current pipeline returns only one series per question. |
| 13 | Add container image vulnerability scanning (Trivy/Grype) to CI; add image signing (Sigstore cosign) | Medium-Term | **Medium** | Low | Required for FedRAMP-aligned deployment pipelines. Base image `python:3.11-slim` does receive CVEs over time. |
| 14 | Expand BLS survey coverage to remaining major programs: OES (OE), QCEW live API, NCS (NB), OIICS (IIF) | Long-Term | **Medium** | High | QCEW and OES are among the most-used BLS datasets by policy researchers. Current coverage stops at 12 surveys. |
| 15 | Implement semantic caching for BLS API responses (Redis or SQLite): cache series data with TTL matching BLS update cadence (monthly) | Long-Term | **Medium** | High | Avoids redundant BLS API calls across concurrent agent sessions; especially valuable under v1 rate limits. |

---

## 6. 30-Day Plan (Quick Wins)

**Goal:** Harden the codebase for a production-ready pilot deployment, with no breaking changes to existing tool interfaces.

### Week 1 — Housekeeping & CI

Owner: Any engineer. Zero risk, immediate ROI.

- Move `pandas` from `[project].dependencies` to `[dev]`; sync `requirements.txt`. CI pass confirms no src import.
- Remove `src/example_server/tools/example_tool.py` (dead code; guarded by existing test).
- Move the 11 root-level `*.md` plan files to `docs/plans/`.
- Add `pytest-cov` to dev deps; add `--cov=example_server --cov-fail-under=85` to CI test step.
- Fix CI Python version: add `python-version: ["3.11", "3.13"]` matrix.

### Week 2 — Dependency & API Hardening

Owner: Backend engineer. Required for sustained production use.

- Register `BLS_API_KEY` in production environment (IBM Code Engine secret); update deploy scripts and `.env.example` to mark v2 key as required (not optional) for production.
- Add HTTP 429 handling and exponential-backoff retry (3 attempts, 2–8s) to `utils.py:fetch_bls`.
- Surface quota exhaustion as a structured error: `{"error": "BLS API quota exceeded", "retry_after_seconds": N}`.

### Week 3 — Auth Layer Prototype

Owner: Security engineer. Blocking for non-hackathon cloud deploy.

- Implement IBM IAM token-verification middleware for the HTTP transport using FastMCP's custom-route hook or a Starlette middleware layer.
- Gate `/mcp` path; leave `/health` and `/version` unauthenticated (required by cloud health check probes).
- Update `SECURITY.md` with the auth architecture and document the ATO delta.

### Week 4 — Test Coverage & Logging

Owner: QA / backend engineer. Operational readiness for production.

- Add unit tests for the clarification-then-retry path in `answer_bls_question.py` (the path where `detect_unresolved_dimensions` returns non-empty and the tool exits early). Target: bring `answer_bls_question` file coverage above 85%.
- Add structured logging: `logger.info("tool_call", extra={"tool": ..., "series_id": ..., "api_version": ...})` in each tool. Use an allowlist — no observation values in logs.
- Confirm coverage gate passes: `uv run pytest tests/ --cov=example_server --cov-report=term-missing`.

**Success metrics at Day 30:** CI passes with coverage ≥ 85%, pandas removed from prod image, auth middleware merged, BLS v2 key configured, all existing 915 tests still green.

---

## 7. 90-Day Plan (Short-Term Priorities)

### Month 2 — Data Freshness & Catalog Automation

- Write a `Makefile refresh-cache` target that downloads all reference files from `https://download.bls.gov/pub/time.series/` and writes a `manifest.json` with download timestamps and file hashes.
- Add a CI weekly job that runs the refresh and opens a PR if any file changed (using `gh pr create` or the GitHub API).
- Validate refreshed files against `bls_series_schema.py` segment lengths; fail fast if column layouts changed.
- Rename the package: `example_server` → `bls_mcp`. Update all references in `src/`, `tests/`, `main.py`, `Dockerfile`, `manifest.yaml`, `deploy/`.

### Month 2 — Benchmark Expansion

- Expand `bls-eval-0` to 100+ questions covering: geographic queries (10 metro areas), NSA vs SA edge cases, out-of-year-range requests, multi-survey comparisons, invalid series IDs, and refusal cases (non-BLS questions).
- Establish a regression threshold: correctness ≥ 75%, scope ≥ 97%. Block CI on regressions (run eval weekly, not per-commit).

### Month 3 — Architecture Improvements

- Refactor `search_series.py`: replace per-survey hint dicts with metadata-driven dispatch. Each survey contributes its hints via the `SeriesSchema.segments` structure, reducing search_series to a ~200 LoC generic engine.
- Add geographic area code resolution to the ambiguity engine: at intent-parse time, map state/metro names to LA/SM area codes using the catalog; populate `QueryIntent.geography_value` with the resolved code.
- Add image scan step to CI using Trivy; fail on CRITICAL CVEs in base image.

**Success metrics at Day 90:** Package renamed, cache refresh automated, benchmark ≥ 100 questions with correctness ≥ 78%, `search_series.py` under 250 LoC, geographic queries passing in eval, CI runs image scan.

---

## 8. 6–12 Month Strategic Roadmap

| Initiative | Quarter | Expected Outcome | KPI |
|---|---|---|---|
| **Multi-survey comparison tool**: extend `answer_bls_question` or add `compare_series` to fetch two series and return a side-by-side comparison with YoY change calculations | Q3 | Agent can answer "How did CPI compare to wage growth in 2023?" | 10 new benchmark questions answered correctly |
| **QCEW and OES survey coverage**: add schemas, catalog entries, flagship series, and tool hints for the two most-requested missing BLS programs | Q3 | Survey count: 12 → 14+ | Catalog dimensions and 5 flagship series per new survey in test coverage |
| **Response caching layer** (SQLite-backed, in-process): cache `get_series_data` responses keyed on (series_ids, start_year, end_year), TTL = 24h (BLS releases monthly) | Q3 | Reduce live BLS API calls by 60–80% in multi-session use | Cache hit rate > 60% in production traces |
| **FedRAMP-aligned deployment configuration**: document and implement network egress controls, audit logging to cloud SIEM, secrets in Vault/IBM Secrets Manager, ATO boundary diagram | Q4 | Server is ATO-ready for GSA production deployment | ATO checklist complete; zero Critical/High findings in security review |
| **Semantic disambiguation upgrade**: replace keyword-matching intent parser with an embedding-based retrieval step over the flagship table; enables natural-language queries like "cost of living data for the South" | Q4 | Benchmark correctness: 72.7% → 85%+ | Correctness rate on 100-question benchmark ≥ 85%, geographic queries ≥ 80% |
| **Multi-agent federation**: expose the server to other agency data systems (Census, FRED) via a cross-agency MCP registry; design a canonical federal-data tool interface standard | Q4–Q1+ | Single agent can answer questions spanning BLS + Census labor force data | At least one cross-agency benchmark question answered correctly end-to-end |

---

## 9. Top-10 Recommended Actions & Sequence

1. **Remove `pandas` from prod deps & sync `requirements.txt`** — 1 hour, zero risk. *(Unblocks clean production image.)*
2. **Add pytest-cov to CI with 85% gate** — 2 hours. *(Establishes the coverage baseline; reveals gaps before making structural changes.)*
3. **Remove `example_tool.py`** — 15 minutes. *(Must happen before package rename.)*
4. **Rename package `example_server` → `bls_mcp`** — 2 hours with sed + tests. *(Depends on #3.)*
5. **Register BLS v2 API key in production; add 429 retry** — 1 day. *(Independently valuable; unblocks sustained production use.)*
6. **Add auth middleware to HTTP transport** — 2–3 days. *(Depends on target platform; unblocks non-hackathon cloud deploy.)*
7. **Write cache refresh Makefile target + CI weekly job** — 3 days. *(Depends on #4 for correct module paths. Prevents silent data staleness.)*
8. **Expand benchmark to 100 questions** — 1 week. *(Required to measure impact of #9 and #10.)*
9. **Refactor `search_series.py` to metadata-driven dispatch** — 3 days. *(Depends on #8 to validate no regression.)*
10. **Add geographic area code resolution to intent parser** — 1 week. *(Depends on #9; biggest single correctness improvement opportunity.)*

**Expected outcomes by action:**

- After #1–#4: Clean, correctly identified, fully testable package suitable for internal publishing.
- After #5–#6: Sustained production deployment with auth and no daily quota exhaustion risk.
- After #7–#8: Operationally sound (fresh data, known reliability baseline at scale).
- After #9–#10: Correctness rate projected to reach 78–82% on expanded benchmark; geographic queries become first-class citizens.

---

## 10. Critical Unknowns & Questions

- **What is the production SLA requirement?** — P95 latency is currently 29.4s across 5 eval runs. If the target is <5s, an async streaming UI and caching are required from day one.
- **Who owns the BLS API key registration?** — The v1→v2 upgrade is a one-time free registration but must be done by an authorized government email. It is unclear if this key currently exists in the IBM Code Engine secret store.
- **When were the local-cache files last refreshed?** — No provenance date exists in any cache file or manifest. If they are from the hackathon period (October 2024), they are approximately 1 annual benchmark cycle behind.
- **Is there a production IBM watsonx Orchestrate instance beyond the hackathon?** — The technical handoff mentions a deployed `gsa_ds_bls_agent`. It is unknown if that deployment is still active or if it needs to be rebuilt for a production environment.
- **What is the target ATO boundary?** — The server processes public, non-PII data. However, the IBM Cloud deployment introduces an external dependency. An ATO analysis should address: data classification, boundary controls, and third-party system authorization.
- **What drove the 27.3% failure rate?** — Detailed per-question failure analysis (which Phoenix traces contain) has not been exported or analyzed in the repository. Understanding the top-5 failure categories is the highest-ROI action for improving correctness.
- **Is the 30-question dataset representative of real user queries?** — The dataset was curated by the development team. Collecting and labeling a sample of real queries from agency stakeholders would validate that the benchmark tracks actual use-case performance.
