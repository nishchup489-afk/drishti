# Drishti

**An open-source, polite, history-aware web crawler.**

![drishti logo](assets/logo.png)


 Drishti fetches pages, remembers every version of them, and tells you what changed, when, and what each page said before.
Drishti is the crawler behind [Developer's Eye](https://developers-eye.vercel.app), a developer-focused search engine. Drishti 1.0 powers it today. Drishti 2.0 is in active development, building on 1.0's codebase rather than replacing it.

> **Status:** 1.0 is stable and running in production. 2.0 started on Nov 26, 2026. Everything marked 🛠 below is planned, not shipped.

---

## Why Drishti

Most crawlers answer one question: *what does this page say right now?* Drishti is built to answer the ones that come after it:

- What changed on this page, and when?
- What did it say six months ago?
- Which pages change often, and which never change at all?
- Was a fetch blocked, rate-limited or refused, and by whom?

It is designed for one person or a small team crawling a focused slice of the web well: thousands of sites and millions of pages, not billions.

### What Drishti will not do

- **No anti-bot evasion.** No fingerprint spoofing, CAPTCHA solving or residential proxy rotation. Drishti identifies itself honestly, and when a site blocks it, Drishti records the block and backs off.
- **No logged-in, paywalled or social-platform crawling.**
- **No whole-web crawling.** Common Crawl already does that.

---

## Features

### Shipped in 1.0 ✅

| Area | What it does |
|---|---|
| **Frontier** | Persistent PostgreSQL URL frontier with leases (`lease_until`), `skip_locked` claiming, retry backoff and recovery of expired leases |
| **Politeness** | robots.txt with status-aware handling, Crawl-delay, per-host politeness, `Retry-After` parsing, 429/5xx backoff |
| **Safety** | DNS/IP policy to block private and internal addresses, redirect policy, capped response body size |
| **Identity** | Honest user agent: `DevelopersEyeBot` |
| **Discovery** | Curated source registry (docs, APIs, blogs, feeds), sitemap discovery, RSS/Atom feeds, JSON API ingestion |
| **Parsing** | HTML text, links and metadata extraction; language detection |
| **URLs** | Normalization, canonical URLs, scope and depth limits |
| **Change detection** | SHA-256 content fingerprint labels every fetch `new`, `updated` or `unchanged` |
| **Recrawl** | Fixed-interval revisits per target |
| **Indexer** | Separate worker that builds PostgreSQL full-text search, with sharding and heartbeats |
| **Operations** | Telemetry, monitoring, crawl and indexer reports, worker recovery |

**Measured:** 100,374 documents stored in about one hour with 37 workers and a 97.9% fetch success rate (local run).

### Planned for 2.0 🛠

#### Page history (the core of 2.0)

1. **Conditional GET.** Send `If-None-Match` and `If-Modified-Since` so servers can answer `304 Not Modified`. The 304 path already exists in 1.0; only the request headers are missing.
2. **Page versions.** Keep every version of a page instead of overwriting one row per URL.
3. **Diffs between versions,** with text and DOM-region diffs that separate "cookie banner changed" from "price changed".
4. **Raw page storage (WARC).** Store the original response so extraction can be re-run without re-crawling.
5. **Adaptive recrawl.** Pages that change often are checked often; pages that never change are checked rarely.

#### Quality and coverage

6. **Near-duplicate detection** with SimHash, on top of exact hashing.
7. **Block detection and classification:** challenge pages, 402, 403, soft blocks. Recorded and reported, never evaded.
8. **PDF parsing** in isolated worker processes with hard memory and time limits, keeping page boundaries.
9. **Headless browser fallback,** used only when a page needs JavaScript to render.
10. **Structured data:** JSON-LD, schema.org and OpenGraph.
11. **Crawl definitions as config files** (YAML/TOML), versioned and validated.
12. **Plugin architecture** for fetchers, parsers, extractors, change classifiers and storage sinks.
13. **Extraction quality tests:** a golden set with measurable extraction quality, and alerts when extraction drifts.
14. **llms.txt / ai.txt parsing,** plus per-crawl cost and budget tracking (pages, bytes, render-seconds).

#### Infrastructure

- **Node-based crawling.** Workers on multiple machines lease work from one frontier. They run on machines we control first; public contributor nodes come later (see [Roadmap](#roadmap)).
- **Adaptive concurrency.** Each worker scales its concurrency to the device's CPU and memory.
- **Control panel.** Live view of the frontier, workers, hosts, blocks and crawl jobs. Read-only first, then controls.
- **Multiple crawl modes,** chosen per job: sitemap/feed-driven, incremental change monitor, focused (vertical), single-site deep crawl, and on-demand API fetch.
- **Pluggable storage.** One storage interface, shipped with PostgreSQL and filesystem / S3-compatible implementations.
- **URLFrontier-compatible API.** Drishti's frontier will speak the [URLFrontier](https://github.com/crawler-commons/url-frontier) API so other crawlers can use it.
- **Index improvements:** better compression, and vector search through pgvector.

---

## Architecture

```
Seed → Frontier → Fetch → Policy → Parse → Extract → Diff → Sink
         ▲                                                   │
         └──────────── recrawl schedule ◄────────────────────┘
```

| Stage | 1.0 | 2.0 |
|---|---|---|
| **Seed** | Source registry, sitemaps, feeds | + config-defined seed sources |
| **Frontier** | Postgres, leases, retries | + adaptive revisit schedule, URLFrontier API, multi-node leasing |
| **Fetch** | Async HTTPX | + conditional GET, browser fallback as a plugin, adaptive concurrency |
| **Policy** | Host, DNS, robots, politeness, redirect | + block classification, robots/ToS snapshot recorded per fetch |
| **Parse** | HTML, RSS/Atom, sitemap, JSON | + PDF (isolated), JSON-LD/schema.org, llms.txt |
| **Extract** | Text, links, metadata | + extraction contracts and drift alerts |
| **Diff** | Content hash only | + versions, text/DOM diffs, change classification |
| **Sink** | PostgreSQL documents | + WARC raw storage, pluggable sinks, pgvector |

### Design rules

- **Stages pass immutable records** (`FetchRecord`, `Document`, `Version`, `Change`). Plugins never touch the frontier directly.
- **Replay without network.** With raw responses stored, Parse → Sink can be re-run from WARC alone.
- **Config controls crawl policy; code controls extraction logic.** YAML should never become a programming language.
- **HTTP first, structured data next, browser last.**
- **Build what a real workload needs.** A feature goes into Drishti when a product built on it needs it.

### Planned plugin interface (draft)

```python
class Fetcher(Protocol):
    async def fetch(self, request: FetchRequest) -> FetchRecord: ...

class Parser(Protocol):
    def accepts(self, record: FetchRecord) -> bool: ...
    def parse(self, record: FetchRecord) -> ParsedDocument: ...

class ChangeClassifier(Protocol):
    def classify(self, old: Version, new: Version) -> list[Change]: ...

class StorageSink(Protocol):
    async def write(self, item: Document | Version | Change) -> None: ...
```

Draft API: names and signatures may change.

### Planned crawl definition (draft)

```yaml
job:
  name: docs-changelog-watch
  mode: incremental

seeds:
  - sitemap: https://docs.example.com/sitemap.xml
  - feed: https://example.com/changelog.rss

scope:
  domains: [docs.example.com, example.com]
  include: ["/docs/**", "/changelog/**", "/pricing"]
  max_depth: 4
  max_pages: 50000

politeness:
  per_host_concurrency: 2
  honor_crawl_delay: true

revisit:
  strategy: adaptive
  min_interval: 6h
  max_interval: 30d

fetch:
  primary: http
  conditional_get: true
  browser_fallback: auto   # never | auto | always

versions:
  keep: true
  diff: semantic

storage:
  metadata: postgres
  raw: warc
  index: postgres_fts

budget:
  max_bytes: 20GB
```

This schema is a proposal, not an implemented API.

---

## Getting started (1.0)

**Requirements:** Python 3.14+, Poetry, PostgreSQL. Drishti currently runs inside the Developer's Eye repo as a submodule and depends on its `backend` package.

```bash
git clone --recurse-submodules https://github.com/nishchup489-afk/developers_eye.git
cd developers_eye

# Database
cd backend
poetry install
poetry run alembic -c alembic.ini upgrade head

# Crawler
cd ../drishti
poetry install
poetry run python -m crawler.main bootstrap                 # register sources, queue start URLs
poetry run python -m crawler.worker.runner --forever --concurrency 2

# Indexer
poetry run python -m indexer.worker.runner --forever --batch-size 100 --concurrency 2 --poll-seconds 10
```

Commands:

| Command | What it does |
|---|---|
| `crawler.main seed` | Register sources from the registry |
| `crawler.main discover` | Find initial crawl targets (sitemaps, feeds) |
| `crawler.main bootstrap` | Both of the above |
| `crawler.worker.runner --forever` | Run a crawl worker |
| `indexer.worker.runner --forever` | Run an indexing worker |

### Tests

```bash
cd drishti
poetry run pytest tests/crawler
poetry run pytest tests/indexer
```

---

## Roadmap

The full task-by-task checklist, with done-when criteria for each item, is in [CHECKLIST.md](CHECKLIST.md).

### Phase 1: History 🛠 *(current)*
Conditional GET · page versions · diffs · WARC raw storage · adaptive recrawl · config files · plugin interfaces

*Inspired by:* Scrapy (plugins) · Heritrix (archival storage) · Nutch (recrawl scheduling) · changedetection.io (change tracking)

### Phase 2: Quality
SimHash near-duplicates · block classification · isolated PDF parsing · browser fallback · JSON-LD · extraction quality tests · llms.txt · cost tracking · pgvector · developer-facing API

*Inspired by:* BUbiNG (near-duplicate detection) · Crawl4AI (extraction strategies) · Firecrawl (API design)

### Phase 3: Scale
Adaptive concurrency · multi-node workers on trusted machines · control panel · URLFrontier-compatible frontier · link-based crawl scoring

*Inspired by:* Crawlee (adaptive concurrency) · URLFrontier (independent frontier) · StormCrawler (distributed processing) · Nutch (scoring)

### Phase 4: Open network
Public contributor nodes with signed submissions and verification by spot-check re-fetching · globally coordinated politeness · high-throughput frontier

*Inspired by:* Mwmbl · Plumb · BUbiNG

### Research only, not on the roadmap
**.onion crawling** is not part of any release. No code is written until it has passed a legal review. If it ever proceeds: a hand-checked allowlist only, metadata only (no page bodies stored), and never enabled by default.

---

## Crawling policy

Drishti is built to be a good citizen of the web:

- It identifies itself with a clear user agent.
- It obeys robots.txt, including per-agent rules and Crawl-delay.
- It limits requests per host and backs off on 429, 503 and `Retry-After`.
- It refuses private and internal network addresses.
- It never tries to bypass access controls, paywalls or bot protection.

If you run Drishti, you are responsible for what you crawl and how you use the data. Public pages are not automatically free to republish: copyright, terms of service, privacy law and local rules still apply. Store and show diffs and snippets with links back to the source rather than mirroring whole pages.

---

## Contributing

Drishti 2.0 is looking for contributors, especially in Phase 1 (versioning, diffs, WARC) and Phase 2 (parsers and extraction tests).

1. Open an issue describing what you want to work on.
2. Keep pull requests focused, with tests.
3. Explain design choices in the PR. This project values the "why" as much as the code.

See `CONTRIBUTING.md` in the Developer's Eye repo for setup details.

---

## Acknowledgements

Drishti learns from the crawlers that came before it: Scrapy, BUbiNG, Crawlee, Apache Nutch, Heritrix, StormCrawler, Crawl4AI, URLFrontier, changedetection.io, Firecrawl, Mwmbl and Marginalia.

## License

[MIT](LICENSE) © 2026 Nish