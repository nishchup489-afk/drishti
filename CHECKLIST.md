# Drishti 2.0 Checklist

The full build checklist for Drishti 2.0. Work goes in phase order: each phase builds on the one before it.

**How to use this file**

- `[x]` done · `[ ]` not started. Use `[~]` in a PR description for "in progress".
- Each section ends with **Done when**: the test that proves it's finished.
- Paths refer to this repo. Database models and migrations currently live in the Developer's Eye `backend` package (`backend/alembic`), so schema changes go there until Phase 0 moves them.
- One rule decides scope: **if a real crawl job needs it, build it. If not, it waits.**

---

## Contents

- [1.0 baseline (already shipped)](#10-baseline-already-shipped)
- [Phase 0: Groundwork](#phase-0-groundwork)
- [Phase 1: History](#phase-1-history)
- [Phase 2: Quality](#phase-2-quality)
- [Phase 3: Scale](#phase-3-scale)
- [Phase 4: Open network](#phase-4-open-network)
- [Research only: .onion](#research-only-onion)
- [Cross-cutting (every phase)](#cross-cutting-every-phase)
- [2.0 release checklist](#20-release-checklist)

---

## 1.0 baseline (already shipped)

- [x] PostgreSQL URL frontier with states, priority, `next_fetch_at`, `lease_until`, `failure_count` (`crawler/frontier/`, `crawler/scheduler/`)
- [x] `skip_locked` claiming, retry backoff, recovery of expired leases (`crawler/scheduler/repository.py`, `crawler/worker/recovery.py`)
- [x] robots.txt with status-aware handling and Crawl-delay (`crawler/policy_and_politeness/robots_policy.py`, `robots_rules.py`)
- [x] Per-host and shared politeness, `Retry-After` parsing, 429/5xx backoff (`politeness_policy.py`, `shared_politeness.py`, `policy_coordinator.py`)
- [x] DNS/IP safety policy and redirect policy (`dns_policy.py`, `redirect_policy.py`)
- [x] Honest user agent `DevelopersEyeBot` (`crawler/config.py`)
- [x] Source registry, sitemap discovery, RSS/Atom feeds, JSON API ingestion (`crawler/seed/`, `crawler/parser/`)
- [x] HTML text, links and metadata extraction (`crawler/parser/html/`)
- [x] URL normalization, scope and depth limits (`crawler/frontier/urls.py`, `scope.py`)
- [x] SHA-256 content fingerprint labelling each fetch new / updated / unchanged (`crawler/processing/fingerprint.py`, `crawler/worker/jobs/utility.py`)
- [x] Accepts `304 Not Modified` responses (`crawler/worker/jobs/jobs.py`)
- [x] Indexer worker with sharding, heartbeats and vocabulary rebuild (`indexer/`)
- [x] Telemetry, monitoring and reporting (`crawler/worker/telemetry.py`, `monitoring.py`, `reporting.py`)
- [x] Benchmarks (`tests/benchmarks/`)

---

## Phase 0: Groundwork

Get the repo ready so 2.0 work doesn't fight the 1.0 layout.

### 0.1 Repo independence
- [ ] Decide where Drishti's database models live: still in Developer's Eye `backend`, or moved into Drishti (`drishti/db/`)
- [ ] If moving: move the crawler-owned models (`crawl_attempt`, `crawl_host`, `crawler_frontier`, `documents`, `robot_cache`, `source`) and their Alembic migrations
- [ ] Remove the `backend` path dependency from `pyproject.toml`, or make it optional
- [ ] Update the README "Getting started" for a standalone clone
- **Done when:** `git clone` + `poetry install` + `alembic upgrade head` + `pytest` work in this repo with no Developer's Eye checkout.

### 0.2 CI
- [ ] GitHub Actions workflow: lint (ruff), type-check (mypy or pyright), `pytest tests/crawler`, `pytest tests/indexer`
- [ ] Postgres service container for the integration tests
- [ ] Secret scanning (copy `secret-scanning.yml` from Developer's Eye)
- [ ] Status badge in the README
- **Done when:** every PR runs the full suite automatically and must pass before merge.

### 0.3 Baseline numbers
- [ ] Record the 1.0 baseline in `docs/benchmarks/1.0-baseline.md`: pages/hour, success rate, workers, RAM per worker, DB size per 100k docs
- [ ] Script to re-run the baseline (`tests/benchmarks/`) against `fake_web_load.py`
- **Done when:** every later phase can show before/after numbers against the same benchmark.

### 0.4 Record types
- [ ] Define versioned, immutable record types: `FetchRecord`, `ParsedDocument`, `Version`, `Change`
- [ ] Each record carries `schema_version`
- [ ] Put them in one module (`crawler/records.py`) with tests
- **Done when:** fetch, parse and write stages pass these records instead of ad-hoc dicts.

---

## Phase 1: History

The core of 2.0: Drishti stops overwriting pages and starts remembering them.

### 1.1 Conditional GET
- [ ] Add `etag` and `last_modified` columns (on the crawl target or document; pick one and document why)
- [ ] Save `ETag` and `Last-Modified` from every successful response
- [ ] Send `If-None-Match` / `If-Modified-Since` on revisits (`crawler/fetcher/fetch.py`)
- [ ] On 304: skip parse/process, update `last_fetched_at`, and record outcome `not_modified`
- [ ] Handle servers that send a 200 with identical content (fall back to the content hash)
- [ ] Handle weak ETags (`W/"..."`)
- [ ] Metric: share of revisits that return 304
- [ ] Tests: 304 path, missing validators, changed ETag, weak ETag, server ignoring the headers
- **Done when:** on a recrawl of unchanged pages, a measurable share returns 304, and bytes downloaded drop against the Phase 0 baseline.

### 1.2 Raw page storage (WARC)
- [ ] Add `warcio` (or equivalent) as a dependency
- [ ] Write each fetch as a WARC `response` record with the request, headers, body and timing
- [ ] Write a WARC `revisit` record for 304s and unchanged content
- [ ] Rotate WARC files by size (e.g. 1 GB) and gzip them
- [ ] Store a pointer on the attempt or version row: `warc_file`, `warc_offset`, `warc_length`
- [ ] Storage target behind an interface: local filesystem first, S3-compatible second
- [ ] Retention setting (keep raw for N days or N versions)
- [ ] CLI to read one record back by URL + timestamp
- [ ] Tests: write → read round-trip, rotation, corrupt or truncated file handling
- **Done when:** any stored version can be traced back to its exact raw response.

### 1.3 Replay
- [ ] CLI `drishti replay --from-warc <file> --parser html-v2`: re-runs Parse → Process → Write with no network access
- [ ] Record `extractor_version` on every document and version
- [ ] Test: replaying the same WARC twice gives identical output
- **Done when:** changing the extractor and re-running it over stored WARCs needs zero new fetches.

### 1.4 Page versions
- [ ] New table `document_version`: `id`, `document_id`, `version_no`, `fetched_at`, `content_hash`, `title`, `body`, `metadata`, `extractor_version`, raw-storage pointer, `previous_version_id`
- [ ] `document` keeps pointing to the latest version (fast search path unchanged)
- [ ] On `updated`, insert a new version; on `unchanged`, insert nothing (only update the fetch time)
- [ ] Unique constraint `(document_id, version_no)`
- [ ] Indexes for "versions of URL X" and "versions in time range"
- [ ] Retention policy (e.g. keep all versions; or keep N + one per month)
- [ ] Backfill: the current document row becomes version 1
- [ ] Migration tested up and down
- [ ] Tests: new → v1, updated → v2, unchanged → no new row, concurrent writers don't create duplicate version numbers
- **Done when:** for any URL you can list every version with its timestamp.

### 1.5 Diffs
- [ ] Text diff between consecutive versions (line or block level)
- [ ] Split the page into regions (headings, paragraphs, tables, code blocks) and diff by region
- [ ] Noise filters: ignore timestamps, cookie banners, "last updated" footers, rotating ads, CSRF tokens
- [ ] New table `change`: `id`, `document_id`, `from_version`, `to_version`, `detected_at`, `diff` (stored or computed), `size` (added/removed), `region`
- [ ] Ignore whitespace-only and noise-only changes (don't create a `change` row)
- [ ] API/CLI: `drishti diff <url> --from <v> --to <v>`
- [ ] Tests: golden pairs of pages with expected diffs, including noise-only pairs
- **Done when:** a real docs page edit shows up as a clean, readable diff, and a timestamp-only edit shows nothing.

### 1.6 Adaptive recrawl
- [ ] Per-target change stats: `checks`, `changes`, `last_changed_at`, estimated change rate
- [ ] Interval rule: shrink the interval after a change, grow it after no change, clamped to `[min_interval, max_interval]`
- [ ] Replace the fixed `crawl_interval_seconds` in `crawler/scheduler/repository.py` with the adaptive value
- [ ] Per-source overrides (e.g. pricing pages minimum 6h, archive pages maximum 30d)
- [ ] Simulation test: synthetic pages with known change rates, measure missed changes vs fetches spent
- [ ] Document the formula and trade-off in `docs/scheduler-and-leases/`
- **Done when:** on the simulation, adaptive scheduling catches at least as many changes as fixed scheduling with fewer fetches, and the numbers are published.

### 1.7 Crawl definitions in config files
- [ ] YAML/TOML job file format (see README draft): `job`, `seeds`, `scope`, `politeness`, `revisit`, `fetch`, `versions`, `storage`, `budget`
- [ ] Pydantic (or similar) schema with clear error messages
- [ ] `drishti validate <job.yaml>` command
- [ ] `drishti run <job.yaml>` command
- [ ] Save the job file's hash with every crawl it produces (reproducibility)
- [ ] Map the current Python constants in `crawler/config.py` and `crawler/fetcher/config.py` onto config fields; constants become defaults
- [ ] Example job files in `examples/`
- [ ] Rule: config controls crawl **policy**; extraction **logic** stays in code
- **Done when:** a new crawl is set up by writing a file, not by editing Python.

### 1.8 Plugin architecture
- [ ] `typing.Protocol` interfaces: `Fetcher`, `Parser`, `Extractor`, `ChangeClassifier`, `StorageSink`
- [ ] Scrapy-style middleware chain around fetch (request → middlewares → fetcher → middlewares → response)
- [ ] Plugin registry using Python entry points (`[project.entry-points."drishti.parsers"]`)
- [ ] Port the existing pieces onto the interfaces: HTTP fetcher, HTML / RSS-Atom / sitemap / JSON parsers, Postgres writer
- [ ] Plugins never touch the frontier directly
- [ ] One external example plugin in `examples/plugins/` with its own test
- [ ] Docs: `docs/plugins.md`, covering how to write, register and test a plugin
- **Done when:** a parser can be added from a separate package without editing Drishti's code.

### Phase 1 exit criteria
- [ ] All 1.1–1.8 done with tests
- [ ] A real job (e.g. 50 docs sites) runs for 7 days and produces versions and diffs
- [ ] Before/after benchmark vs Phase 0 baseline published
- [ ] Engineering blog post: "How Drishti remembers the web"

---

## Phase 2: Quality

### 2.1 Near-duplicate detection
- [ ] 64-bit SimHash over normalized text (shingles)
- [ ] Hamming-distance lookup (bucketed by bands for speed)
- [ ] Mark near-duplicates and link them to a canonical document; don't index them twice
- [ ] Threshold set by testing on a labelled sample
- [ ] Tests: identical, near-identical (boilerplate change), and unrelated pages
- **Done when:** mirror and print-version pages collapse into one result.

### 2.2 Block detection and classification
- [ ] Classify outcomes: `ok`, `not_modified`, `blocked_403`, `payment_required_402`, `rate_limited_429`, `challenge_page`, `captcha`, `soft_block` (200 with an "access denied" body), `geo_blocked`, `legal_451`
- [ ] Challenge-page detector (known markers plus a too-short body that differs from previous versions)
- [ ] On a block: back off the host, record the reason, surface it in reports. **No evasion.**
- [ ] Per-host block-rate metric and alert
- [ ] Tests with saved fixtures of each block type
- **Done when:** a reports page lists blocked hosts with their reason, and a challenge page is never stored as content.

### 2.3 PDF parsing
- [ ] PDF parser plugin (e.g. pypdf / pdfminer.six; evaluate one layout-aware option)
- [ ] Run in an isolated subprocess with hard memory and wall-clock limits
- [ ] MIME and size guards before parsing
- [ ] Keep page boundaries and page numbers; store page-level text
- [ ] Extraction warnings (scanned/no text layer, encrypted, truncated)
- [ ] OCR as an optional, separate plugin (not default)
- [ ] Tests: normal, huge, malformed, scanned, encrypted PDFs; a parser crash must not stall the worker
- **Done when:** a 500 MB or malformed PDF times out cleanly without blocking other URLs.

### 2.4 Headless browser fallback
- [ ] Browser fetcher plugin (Playwright), disabled by default
- [ ] Render-on-detect rules: empty main content, framework markers, `<noscript>` warnings
- [ ] Per-job `browser_fallback: never | auto | always`
- [ ] Block images, fonts and media in the browser
- [ ] Recycle browser contexts every N pages; hard memory cap per browser
- [ ] Metric: share of fetches rendered, render-seconds
- **Done when:** a JS-only docs site gets real content while static sites never start a browser.

### 2.5 Structured data
- [ ] JSON-LD extractor (extend `crawler/parser/structured.py`)
- [ ] Microdata / RDFa (lower priority)
- [ ] OpenGraph and Twitter card metadata
- [ ] Store as structured metadata on the version
- [ ] Tests with real-world fixtures
- **Done when:** a product or article page's structured fields are searchable.

### 2.6 llms.txt and ai.txt
- [ ] Fetch and parse `/llms.txt` and `/ai.txt` alongside robots.txt
- [ ] Store a policy snapshot per fetch: robots.txt hash, AI directives, timestamp
- [ ] Report: which hosts restrict which agents
- **Done when:** every stored version records the access policy that applied when it was fetched.

### 2.7 Extraction quality tests
- [ ] Golden set: 100+ pages (docs, blogs, changelogs, marketing pages) with hand-labelled main content
- [ ] Metrics: main-content recall, boilerplate leakage, heading/table preservation
- [ ] Runs in CI; a drop beyond a threshold fails the build
- [ ] Extraction contracts per source: expected fields, alert when null rates jump (drift)
- [ ] Publish results in `docs/benchmarks/extraction.md`
- **Done when:** extraction quality is a number that CI defends.

### 2.8 Change classification
- [ ] `ChangeClassifier` plugin: `cosmetic`, `content`, `deprecation`, `breaking`, `pricing/limits`, `removed_page`
- [ ] Rule-based first (keywords, regions, numeric changes); optional LLM classifier behind a plugin
- [ ] Labelled test set
- **Done when:** "show breaking changes this week" returns sensible results.

### 2.9 Cost and budget tracking
- [ ] Per-crawl counters: requests, bytes, render-seconds, parse-seconds, storage bytes
- [ ] Budget caps in job config (`max_pages`, `max_bytes`, `max_render_seconds`); crawl stops cleanly at the cap
- [ ] Estimated $ cost (configurable unit prices)
- **Done when:** every crawl report shows what it cost, and a budget cap is never exceeded.

### 2.10 Vector search
- [ ] Review the existing `indexer/vector.py` and its tests
- [ ] pgvector column and index (HNSW) for chunk embeddings
- [ ] Heading-aware chunking
- [ ] Embedding model choice documented (local, free)
- [ ] Hybrid retrieval: BM25 + vector, evaluated on the existing relevance metrics
- **Done when:** hybrid search beats BM25 alone on the evaluation set, or the README says it didn't.

### 2.11 Developer-facing API
- [ ] `GET /fetch?url=` → clean content + provenance (on demand)
- [ ] `GET /versions?url=` and `GET /diff?url=&from=&to=`
- [ ] `GET /changes?since=&type=&host=`
- [ ] Every response includes provenance: `fetched_at`, `fetcher`, `status`, `content_hash`, raw pointer, `extractor_version`
- [ ] OpenAPI spec, API keys, rate limits
- **Done when:** a RAG pipeline can pull fresh, versioned content with provenance in one call.

### 2.12 Inspect view
- [ ] `/inspect?url=` page: fetch tier (HTTP or browser), status, latency, block class, policy snapshot, version list, latest diff
- **Done when:** every search result links to a page explaining exactly how it was crawled.

### Phase 2 exit criteria
- [ ] All 2.1–2.12 done with tests
- [ ] Extraction and dedup numbers published
- [ ] Blog post on extraction quality or change classification

---

## Phase 3: Scale

### 3.1 Adaptive concurrency
- [ ] Worker reads available CPU and memory (container-aware: cgroup limits, not host totals)
- [ ] Scale concurrency up/down within `[min, max]` based on CPU, memory and error rate
- [ ] Per-host limits always win over worker concurrency
- **Done when:** the same worker command runs well on a laptop and on an Oracle VM without hand-tuning.

### 3.2 Multi-node workers (trusted machines only)
- [ ] Worker identity: `node_id`, version, capabilities (browser yes/no, PDF yes/no, memory)
- [ ] Workers lease from a central frontier over an API (not direct DB access)
- [ ] Capability-aware leasing (browser-needing URLs go only to browser-capable nodes)
- [ ] Node heartbeat, auto-release of leases from dead nodes
- [ ] Every fetch record carries `node_id`
- [ ] Run on two machines you control (e.g. Oracle VM + laptop)
- **Done when:** killing a node mid-crawl loses zero URLs and the crawl finishes on the other node.

### 3.3 Global politeness across nodes
- [ ] Per-host rate limits enforced centrally, not per node
- [ ] Shared robots.txt cache
- [ ] Test: two nodes never exceed the per-host limit together
- **Done when:** adding nodes makes the crawl faster overall but never hits any single host harder.

### 3.4 Control panel
- [ ] Read-only first: frontier size by state, crawl rate, nodes and heartbeats, blocked hosts, recent changes, budget usage
- [ ] Then controls: pause/resume a job, pause a host, drain a node, re-queue failures
- [ ] Auth for controls
- **Done when:** you can run and debug a crawl from the panel without SQL.

### 3.5 URLFrontier-compatible API
- [ ] Implement the URLFrontier gRPC API on top of Drishti's Postgres frontier
- [ ] Conformance tests against the URLFrontier test suite (if available)
- [ ] Docs: using Drishti's frontier from another crawler
- **Done when:** an external URLFrontier client can push and lease URLs from Drishti.

### 3.6 Link-based crawl scoring
- [ ] Store a link graph (`edges(src, dst)` in Postgres or Parquet; no graph database)
- [ ] Nutch-style scoring (e.g. OPIC) to prioritize which URLs to fetch next
- [ ] Combine with change rate and source priority in `crawler/scheduler/priority.py`
- **Done when:** a focused crawl with a fixed budget reaches more relevant pages than FIFO.

### 3.7 Indexer compression
- [ ] Measure current index and table size per 100k documents
- [ ] Compress stored bodies and versions (zstd), and store diffs instead of full copies where cheaper
- [ ] Evaluate whether a custom inverted index is worth it vs Postgres full-text search (write up the decision)
- **Done when:** storage per 100k docs drops measurably with no search-quality loss.

### Phase 3 exit criteria
- [ ] All 3.1–3.7 done with tests
- [ ] Multi-node benchmark published (throughput vs nodes)

---

## Phase 4: Open network

Public contributor nodes. Only start once Phases 1–3 are solid.

### 4.1 Identity and trust
- [ ] Contributor accounts and node keys
- [ ] Signed submissions (each fetch result signed by its node key)
- [ ] Contributor terms: what they agree to crawl, and their responsibilities

### 4.2 Verification
- [ ] Spot-check verification: re-fetch a random sample of submitted pages from a trusted node and compare hashes
- [ ] Trust score per node; lower trust → more spot-checks; failures → quarantine
- [ ] Never let one untrusted node be the only source for a document

### 4.3 Abuse controls
- [ ] Central allowlist of crawl scopes; nodes can't add arbitrary targets
- [ ] Per-node rate caps; global politeness still applies
- [ ] Kill switch per node and globally
- [ ] Abuse report channel for site owners (contact in the user agent string)

### 4.4 Throughput
- [ ] BUbiNG-style high-throughput frontier work, only if measurements show the frontier is the bottleneck

### Phase 4 exit criteria
- [ ] A poisoned-submission test is caught by verification
- [ ] Site-owner opt-out honoured network-wide within a set time
- [ ] Public contributor guide

---

## Research only: .onion

**Not part of any release.** Nothing in this section starts until the legal gate is passed.

- [ ] **Gate:** written legal review covering fetching, storing and indexing hidden-service content
- [ ] If approved, write a design doc first: hand-checked allowlist only, metadata only (no page bodies stored), off by default, separate deployment
- [ ] No code merged into `main` before the design doc is approved

---

## Cross-cutting (every phase)

### Testing
- [ ] Unit tests for every new module
- [ ] Postgres integration tests for every schema change
- [ ] Fault tests: DB down, worker killed, disk full, malformed responses
- [ ] Coverage reported in CI

### Observability
- [ ] New stages emit metrics (counts, durations, errors)
- [ ] Structured logs with `job_id`, `target_id`, `node_id`
- [ ] Crawl report includes new outcome types (304s, blocks, versions created, changes detected)

### Security and safety
- [ ] DNS/IP safety policy applies to every fetcher plugin, including the browser
- [ ] Size and time limits on every parser
- [ ] No secrets in config files; use env vars
- [ ] Dependency audit in CI (`pip-audit`)

### Documentation
- [ ] Each feature gets a page in `docs/` (what, why, trade-offs)
- [ ] Every schema change gets an entry in a changelog
- [ ] README status markers (✅ / 🛠) updated when features ship
- [ ] `CONTRIBUTING.md` in this repo

---

## 2.0 release checklist

- [ ] Phases 1 and 2 complete (Phases 3–4 can follow in 2.x releases)
- [ ] All tests green in CI
- [ ] Migrations tested on a copy of production data
- [ ] Benchmarks published: throughput, 304 rate, extraction quality, dedup rate, freshness vs fetch budget
- [ ] README and docs updated; nothing listed as shipped that isn't
- [ ] Example job files work from a clean clone
- [ ] `CHANGELOG.md` with upgrade notes from 1.0
- [ ] Tagged release `v2.0.0` with release notes
- [ ] Developer's Eye running on 2.0 in production
- [ ] Launch post (portfolio, dev.to, Hashnode)
