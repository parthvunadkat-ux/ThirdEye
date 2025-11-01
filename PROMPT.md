SUPREME MASTER BUILD PROMPT — Third Eye — The Brand Prophet

(Open Source • Global • One Input • Evidence-First • Resilient • A11y • Privacy-Safe)

You (the agent) are a world-class cross-functional team (product, backend, ML/CV, geospatial, frontend, DevOps, SecOps, MLOps, QA). Build a self-hostable, global brand intelligence terminal.
Single input: brand (string).
Output: a global, interactive intelligence dashboard + an analyst (“Brand Prophet”) that autonomously collects, geo-infers, clusters, estimates media mix/spend, maps OOH, detects events/influencers, benchmarks vs industry, summarizes with citations & confidence, and exports.
Constraint: 100% open-source stack (no OpenAI/Google APIs).
Non-negotiables: Explainability, provenance, guardrails, rate limits, quotas, resiliency, modular adapters, hard docs, comprehensive tests.


---

0) Executive Architecture (render an ASCII diagram + PNG in README)

Frontend: React + Vite + TypeScript + Tailwind + shadcn/ui + Recharts + MapLibre GL + Framer Motion.

API: FastAPI (Python 3.11), Pydantic v2, JWT auth, RBAC, CORS hardening.

Workers: Celery + Redis; Prefect for DAG orchestration/scheduling (cron + backoff).

DB: PostgreSQL + SQLAlchemy + Alembic (strict migrations).

Vector Index: pgvector (primary) and FAISS (fallback for local/offline).

Cache & RL State: Redis (token buckets/locks).

Scraping: Playwright (headless/headful + stealth), requests, BeautifulSoup, Tesseract OCR, optional undetected-chromedriver adapter.

Vision: OpenCLIP (embeddings), YOLOv8 (billboards/logos; feature flag), OpenCV face-blurring.

LLM Runtime: Ollama running Llama 3 (default) or Mistral (int4/int8 quantization).

Geocoding/Maps: Nominatim/Photon, MapLibre GL + OSM tiles (with self-host option).

Exports: WeasyPrint (HTML→PDF) + headless Chromium fallback, CSV/JSON.

Containers: Docker Compose (api, worker, frontend, db, redis, pdf service).

CI/CD: GitHub Actions (lint, tests, build, Trivy scan, SBOM via Syft).



---

1) Monorepo Layout (generate exactly, including boilerplate)

third-eye-oss/
  README.md
  .env.example
  docker-compose.yml
  infra/
    Dockerfile.api
    Dockerfile.worker
    Dockerfile.frontend
    Dockerfile.pdf
  backend/
    app/
      main.py
      core/
        config.py
        logging.py
        security.py
        flags.py
        errors.py
        constants.py
      api/v1/
        routes.py
        schemas.py
        deps.py
      services/
        ratelimit.py
        scheduler.py
        exports.py
        summarizer.py
        embeddings.py
        vision.py
        clustering.py
        estimators.py
        benchmarks.py
        evidence.py
        georesolver.py
        language_id.py
        validators.py
        persistence.py
        health.py
      connectors/
        meta_like_ads_public.py
        brand_site_news.py
        social_public_stub.py
        ugc_ooh_finder.py
        influencer_public.py
      models/
        base.py
        tenant.py
        user.py
        brand.py
        creative.py
        ooh_evidence.py
        event.py
        influencer.py
        geostat.py
        insight.py
        competitor.py
        runlog.py
        geocache.py
      db/
        session.py
        init_db.py
        alembic/
          env.py
          script.py.mako
          versions/
    tests/
      unit/
        test_georesolver.py
        test_estimators.py
        test_ratelimit.py
        test_clustering.py
        test_validators.py
        test_persistence.py
      integration/
        test_connectors_contracts.py
        test_api_contracts.py
      e2e/
        test_global_scan.py
  workers/
    worker.py
    jobs/
      scan_brand.py
      process_media.py
      generate_snapshot.py
  frontend/
    src/
      main.tsx
      App.tsx
      lib/api.ts
      lib/format.ts
      components/
        ThirdEyeBar.tsx
        MediaMixChart.tsx
        GlobalTimeline.tsx
        WorldHeatmap.tsx
        CreativeGallery.tsx
        CreativeModal.tsx
        OOHMap.tsx
        EventsTimeline.tsx
        InfluencerCards.tsx
        BenchmarksTable.tsx
        EvidenceDrawer.tsx
        ConfidencePill.tsx
        ExportButtons.tsx
        LoadingState.tsx
        EmptyState.tsx
        ErrorState.tsx
        Toast.tsx
      pages/
        Home.tsx
        Dashboard.tsx
    index.html
    tailwind.config.js
    postcss.config.js
    package.json
  config/
    flags.yaml
    limits.yaml
    cpm_tables.yaml
    er_baselines.yaml
    mobility_proxies.yaml
    attendance_multipliers.yaml
    category_baselines.yaml
    connectors_allowlist.yaml
    connectors_blocklist.yaml
  sample_data/
    creatives_demo.csv
    social_demo.csv
    benchmarks_demo.csv
    ugc_ooh_demo.csv
  scripts/
    demo_run.sh
    seed_demo_data.py
    build_sbom.sh


---

2) Legal/Ethical Guardrails & Workarounds

Robots/ToS: Implement RobotsPolicy utility reading robots.txt and connectors_allowlist.yaml / blocklist.yaml. Respect disallow; provide “demo mode” using CSV seeds when blocked.

Public Data Only: No login scraping. If a source hard-blocks, connector degrades to CSV import path with a banner in UI: “Evidence limited due to access constraints.”

PII/Privacy: Blur faces (OpenCV) if flags.blur_faces=true. Strip EXIF on persist unless needed for geo. No storage of personal emails/phones. Respect takedown by URL hash.

Evidence-first: All insights must include evidence[] + confidence. If unknown → “Insufficient public data” with confidence='L'.



---

3) Single Input → Global Autonomy (no city input)

Input brand triggers global scan for last N days (default 30).

Sources fan-out under quotas; throttled ones are rescheduled; partial snapshot is materialized so the UI is never empty.



---

4) Data Contracts (Pydantic) & DB Schema (SQLAlchemy)

Brand, RunLog, Creative, OOHEvidence, Event, Influencer, BrandGeoStat, Insight, Competitor, Benchmark, GeoCache.

EvidenceItem schema (URL, timestamp, caption, screenshot path, source tag, platform ids) with content hash for idempotency.

Invariants:

Every persisted Creative/Event/OOH row MUST carry confidence and at least one evidence item.

All lat/lon normalized to WGS84; timestamps in UTC ISO8601.




---

5) Connectors (global, rate-limited, resumable, contract-tested)

Interface

class Connector(Protocol):
    name: str
    async def fetch(self, brand: str, period: DateRange, budget: CrawlBudget) -> AsyncIterator[EvidenceItem]:
        ...

Must consume tokens from RateLimitService; obey QuotaPolicy; write checkpoints to RunLog.

On 429/403/anti-bot: fail soft, return partials, schedule remainder via Sequencer.


Implement

1. meta_like_ads_public.py

Pull public creatives (open ad library mirrors or demo CSV). Use local disk cache (24h).



2. brand_site_news.py

Respect robots; shallow crawl brand.com/news|press|stories; capture title/date/image/body; screenshot; store raw_html hash.



3. social_public_stub.py

Mode A: CSV import (default).

Mode B (flags.enable_social_scrape): allowed endpoints + hard throttle; dedupe by URL/media hash.



4. ugc_ooh_finder.py

Pattern search terms → fetch images; YOLOv8 detect billboard/logo (if flag on); OCR; geocode POIs via Nominatim/Photon; compute visibility proxy; attach media thumbnail.



5. influencer_public.py

Given handles seen in posts, scrape public bios+counts if visible; derive ER from visible post stats; map fee band using region tables; store evidence.




Contract Tests: tests/integration/test_connectors_contracts.py verifying output adheres to EvidenceItem schema, rate limit hooks, and checkpointing.


---

6) GeoResolver (no city input; multi-signal fusion)

Signals: NER place extraction (spaCy/Stanza) → Nominatim/Photon; language id (fastText/CLD3) → region priors; ccTLD; timezone hints; venue names; EXIF GPS.

Combiner: Bayesian fusion → geoscore ∈ [0,1]; thresholds: H≥0.75, M≥0.4, else L.

Rollups: compute BrandGeoStat at global/continent/country/metro; persist geoscore, confidence.


Edge Cases & Workarounds

Ambiguous toponyms (“Springfield”) → list candidates; pick via language priors + nearest POI; if tie → downgrade confidence.

Non-Latin scripts → ensure Unicode normalization (NFC), language id robust to mixed scripts.

No geosignal → store geo_country=None with confidence='L'.



---

7) Vision, OCR, Embeddings, Clustering & Dedup

Images → OCR: Tesseract; language auto-detect; store text with confidence.

Image Embeddings: OpenCLIP ViT-B/32 → pgvector.

Text Embeddings: sentence-transformers (all-mpnet-base-v2) for captions+OCR text → pgvector.

Dedup: perceptual hash (pHash); drop near-duplicates within Hamming distance threshold.

Clustering: HDBSCAN on concatenated normalized vectors; seed for reproducibility; compute cluster size, recency decay, platform/geo mix.


Workarounds:

If GPU unavailable: use CPU inference (slower) + flags.enable_vectors=false to degrade to simple grouping by pHash/time.

Store small thumbnails (256–512px) for UI; lazy-fetch originals only on demand.



---

8) Estimators & Heuristics (config-driven, globally aware)

Digital Spend Proxy:

freq = recency_weighted_cluster_count(cluster)
impressions_proxy = freq * platform_visibility_factor(platform)
spend_low/mid/high = impressions_proxy * CPM(region, platform)
confidence = H if ≥2 corroborating sources <15 days; else M/L

OOH Spend Proxy:

impressions_daily = visibility_score * mobility_proxy(region)
spend_band = (impressions_daily * 30) * CPM_OOH(region_tier) / 1000

Events Attendance:

attendance_low/mid/high = unique_posters * region_event_multiplier

Influencer Fee Band by region + follower band + ER.


Outlier Guards & Normalization

Clamp CPM to config ranges; ER to [0.1%, 25%]; log any clamp in RunLog.stats.

Convert currencies to USD and INR using offline static table (config/currency_rates.yaml) to avoid paid FX APIs; document that rates are approximate.

Confidence calculation centralized; every exposed number carries H/M/L.



---

9) Benchmarks & Industry Baselines

config/category_baselines.yaml (🔧 CATEGORY_BASELINE) with media mix% per region/continent and CPM/ER baselines (cpm_tables.yaml, er_baselines.yaml).

Deltas: label better/worse vs category by region; render on BenchmarksTable and in Third Eye summary (with evidence).


Workaround when baselines missing:

Fallback to global medians computed from your internal archive (Benchmark table). Mark confidence = L; banner “No category baselines for region”.



---

10) Brand Prophet (local analyst) via Ollama

Runtime: ollama run llama3 (document start script).

Retriever (RAG): pgvector over evidence/creatives/ooh/events/influencers/benchmarks/geostats; relevance top-k merged with structured stats.

Tools: DB readers to fetch top clusters, geostats, spend bands; formatted context window ≤ model limits.

Feedback Loop: store thumbs up/down; log which evidence supported accepted claims; nightly job samples accepted/declined outputs for prompt tuning.

Latency Workarounds:

Use quantized models (Q4_K_M).

Pre-compress long evidence (map to key facts).

Cache summaries for 24h per brand unless new evidence crosses thresholds.



System Prompt (include verbatim; same as prior version, kept strict about citations & confidences).


---

11) API Design (FastAPI, typed, documented)

GET /health

POST /brands {name} → {brand_id}

POST /scan {brand_id, period_days?=30} → {run_id}` (triggers Prefect DAG + Celery tasks)

GET /runs/{run_id} → {status, progress%, throttles, ETA? (best-effort), next_scheduled}

GET /brands/{id}/snapshot?period_days=30 → global snapshot JSON (rolled up)

GET /insights/{id} → {insight, evidence[], confidence, rationale?}

POST /import/social-csv (admin)

GET /export/{brand_id}.{pdf|csv|json} → artifacts


Security:

JWT, RBAC: roles admin|analyst|viewer.

Rate-limit per user/IP.

CORS: explicit origins; CSP for frontend.

CSRF token for any state-changing admin endpoints (CSV import).



---

12) Frontend UX (fast, accessible, resilient)

Home: brand input; “Scan” CTA; recent scans list with status chips (Running/Partial/Ready/Failed).

Dashboard:

Third Eye Bar (sticky): analyst summary with [evidence #id] (H/M/L); “Refresh” button; confidence legend.

Global Overview: Media Mix Pie (low/mid/high toggle), Budget Bands (stacked with tooltips explaining formula), Global Timeline (activity).

World Heatmap: choropleth by activity/spend; click country → drawer (top creatives/OOH/events, per-country media mix & deltas).

Creatives Gallery: clustered; filters (platform/date/theme/TG); click → modal with carousel, caption, ER proxy, impressions proxy, spend bands, date, link, Evidence Drawer; copy link to clipboard.

OOH Map: MapLibre with clustering; click pin → drawer with title/POI, impressions/day, spend band, geoscore, evidence links.

Events Timeline: attendance bands, VIP handles, source links; confidence pill.

Influencers: follower band, ER, fee band (region), sample posts; evidence pill.

Benchmarks Table: deltas vs category per region with better/worse tags.

Exports: PDF/CSV/JSON buttons; loading indicators; toast on success.



Perf Workarounds

Code splitting per page; lazy load heavy components (Map, Gallery).

Virtualized lists for galleries.

Image lazy loading with low-quality placeholders.

Avoid rerenders via memoization.

All charts/maps read precomputed JSON (fast).


A11y

Keyboard navigation, focus rings, aria labels, high-contrast mode, color-blind palette for charts; screen-reader friendly Evidence Drawer.



---

13) Exports

PDF: WeasyPrint primary; Chromium fallback container. Auto page breaks; print CSS with consistent styles; cover page; evidence thumbnails inline with links (max 3/section).

CSV: creatives.csv, ooh_evidence.csv, events.csv, influencers.csv, geostats.csv, insights.csv. UTF-8 BOM for Excel friendliness.

JSON: snapshot-{brand}-{date}.json with versioned schema.



---

14) Rate Limiting, Quotas, Sequencing, Idempotency

RateLimitService: Redis token buckets; tokens/sec, burst.

QuotaPolicy: per-connector daily {max_pages, max_requests, max_media}; per-run CrawlBudget.

Sequencer: exponential backoff with jitter; reschedule 429/403; continue other sources; write checkpoints (% complete per connector).

Idempotency: content hashes for dedup; upsert by (brand_id, external_id); run-safe re-entrancy.


Edge Workarounds

CAPTCHAs / Bot-detectors: switch to headful, slow mouse movement, randomized delays, limited viewport sizes, rotate UAs; if still blocked → degrade to CSV seeds; display UI banner.

IP Blocking: support rotating proxies (config only; do not implement paid vendors); if blocked, throttle & reschedule next day.



---

15) Data Quality, Validation, Canonicalization

Validators: ensure URLs are normalized; timestamps to UTC; text NFC; platform enums strict.

Schema Guards: null checks; lat/lon ranges; region codes ISO-3166.

Provenance Chain: evidence carries origin URL & capture timestamp; if screenshot generated, store path + hash; DB row references evidence ids.



---

16) Observability, Telemetry, and Ops

Metrics (log-based or Prometheus format): pages scraped, throttles count, average latency, % geo-resolved (country/metro), estimator clamp counts, cache hit rates.

Structured Logs: JSON with run_id, brand, connector, step, duration_ms, outcome.

Alerts: email/webhook on pipeline failure, throttling surge, geo-resolve below threshold, export failure.

Run Book in README: common failures & fixes.



---

17) Security, Secrets, Supply Chain

Secrets: no secrets needed for open stack; keep .env for service URLs; never expose server internals to frontend.

Auth: JWT, password hashing via Argon2/bcrypt; account lockout on brute force; per-IP rate limit.

Headers: CSP, X-Frame-Options, HSTS; sanitize user input.

Supply chain: pin versions; generate SBOM via scripts/build_sbom.sh; scan containers with Trivy in CI.

Backups/DR: nightly DB backup (pg_dump); keep 7-day retention in /backups; test restore monthly.



---

18) Internationalization, Timezones, Units

TZ: store UTC; display local via Intl APIs; note DST.

i18n: string maps; extensible to non-English UI.

Numerics: locale-aware separators; currency dual display (USD/INR) using offline rates.



---

19) Testing (deep)

Unit: GeoResolver (ambiguous places, multi-signal fusion), Estimators (CPM math, OOH/mobility proxies, confidence rules), RateLimiter (token bucket), Clustering (deterministic), Validators (schema).

Integration: connector contract tests (inputs/outputs under RL), API contracts (OpenAPI schema), RAG retrieval correctness (top-k includes expected items).

E2E: seeded demo → /scan → snapshot with ≥2 countries; insights present with evidence; exports OK.

Property-based & Fuzz: URLs, timestamps, lat/lon.

Performance tests: time to first snapshot; dashboard render under mocked large datasets.

Red Team: prompts that try to make Brand Prophet hallucinate—assert it replies “Insufficient public data.”



---

20) Performance Budgets & Fallbacks

First dashboard render from cached snapshot < 2s.

LLM inference budget: < 3s on cached context; fallback to cached summary if model cold.

Geocoding batched; geocache table; TTL 90 days.

Heavy steps behind flags (enable_yolo, enable_vectors, enable_social_scrape).



---

21) UX for Partial/Degraded Data

If a connector throttles/blocked: show partial results with banner “Completing in background—data limited due to access constraints.”

Confidence pills visible everywhere; tooltips explain formula & source count.

Evidence Drawer always shows at least one URL/timestamp or states “No public evidence”.



---

22) Explanations & Transparency

Every chart has a tooltip “How this was computed” with formula, tables used, and confidence.

Third Eye Bar summary: each bullet ends with [evidence #id] (H/M/L). Clicking an id opens the Evidence Drawer.



---

23) Deployment & DevX

scripts/demo_run.sh boots services, seeds demo data, creates brand 🔧, triggers scan, opens dashboard.

README includes: quick start, feature flags, limits tuning, troubleshooting (CAPTCHAs/headless stuck), DR restore walkthrough, performance tips.



---

24) Acceptance Criteria (must pass)

25. Entering only a brand name kicks off a global scan; partial snapshot materializes even if some connectors reschedule.


26. Dashboard shows Third Eye Bar, Media Mix Pie, Budget Bands, World Heatmap, Creatives Gallery, OOH Map, Events, Influencers, Benchmarks, Exports.


27. Every surfaced number/claim has confidence and at least one evidence item, or explicitly states “Insufficient public data.”


28. Cached snapshot loads <2s; PDF/CSV/JSON exports produce files with consistent schemas.


29. CI passes lint+tests+scan; SBOM available; README complete with architecture diagram.


30. Face-blurring and vector flags togglable; dataset size scaling tested (≥10k creatives, pagination/virtualization OK).




---

25) Fill Before Generation

🔧 CATEGORY_BASELINE (e.g., "Apparel & Footwear")

🔧 BRAND_EXAMPLE (for demo, e.g., "Puma" or your target)

(Optional) OSM tile server URL (self-host or public)

Default period: last 30 days


Generate the complete repository, code, configs, tests, seed data, CI, and README exactly per this spec.


---

Extra Engineer Notes for the Agent

Prefer simple, reliable algorithms; keep all heuristics in config/*.yaml.

Implement CSV demo paths so the entire UI is explorable offline.

All user-visible numbers show H/M/L confidence and link to Evidence Drawer.

Where scraping is blocked, do not fail the run—emit partials with clear banners and reschedule jobs automatically.

No hallucinations. If in doubt, output “Insufficient public data.”
