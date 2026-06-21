# LocalHarmBench — Scam/Fraud Advisory Source Corpus

A **live, continuously-ingestible** catalog of official and authoritative sources
of scam/fraud/financial-crime advisories, organized by world region, for the
**LocalHarmBench** project — a HarmBench-style benchmark for *region-specific*
consumer harms. The corpus grounds the benchmark in real, locally-relevant scam
and fraud patterns so models can be tested on whether they:

- **empower a bad actor** to commit a locally-prevalent scam if prompted,
- **help a potential victim** recognize and avoid a scam they describe,
- correctly identify region-specific fraud typologies (PIX, M-Pesa, UPI, GCash,
  pig-butchering, digital-arrest, etc.).

The sources are catalogued — not the scam content itself. A downstream pipeline
pulls fresh advisories from each source's feed/endpoint to keep the benchmark
current.

## Regions (artifacts)

Each region is a standalone, machine-readable catalog under `sources/`:

| Artifact | Region | Countries covered |
|---|---|---|
| [`sources/brazil.yaml`](sources/brazil.yaml) | Brazil | BR |
| [`sources/latam_es.yaml`](sources/latam_es.yaml) | Spanish-speaking Latin America | MX, CO, AR, CL, PE, EC, UY, PY, BO, VE, CR, PA, GT, DO + regional |
| [`sources/africa.yaml`](sources/africa.yaml) | Africa | NG, ZA, KE, GH, EG, MA, TZ, UG, ET, RW, ZM + regional |
| [`sources/india.yaml`](sources/india.yaml) | India | IN |
| [`sources/southeast_asia.yaml`](sources/southeast_asia.yaml) | Southeast Asia | SG, MY, PH, ID, TH, VN + cross-border (KH/MM/LA) |

Schema: [`schema/source.schema.yaml`](schema/source.schema.yaml).

## How a source record is shaped (for live ingestion)

Every record carries enough to wire up a fetcher and a license gate. Key fields:

- **`feed`** — `{url, format, confirmed}`. `confirmed: true` means the endpoint
  was actually fetched and returned valid RSS/Atom/JSON during research
  (2026-06-21). `confirmed: false` means the feed is plausible/likely but was
  not byte-verified (often because the site bot-blocks automated fetchers).
- **`ingestion_mechanism`** — `rss` | `atom` | `json_api` | `html_scrape` |
  `pdf_periodic` | `manual`. Tells the pipeline *how* to pull.
- **`update_cadence`**, **`languages`**, **`content_focus`** — scheduling, NLP,
  and filtering hints.
- **`high_value: true`** — structured "alert list" sources (unauthorized/illegal
  entity registries, investor-alert lists, scam-number/blacklist feeds). These
  are the densest, most reliable signal for the benchmark.
- **`license_status`** + **`usable`** — the legal gate (see below).

## Legal usability (`usable`)

Per the project's "permissive + government public-domain only, flag the rest"
policy, every record is tagged:

| `usable` | Meaning |
|---|---|
| `yes` | Open license or government public-interest info — full reuse OK with attribution. |
| `headlines_only` | Copyrighted (most fact-checkers): ingest title + link + short summary as fair citation; **no** full-text copy or derivative corpus. |
| `flagged` | Include, but license is ambiguous / bot-blocked / needs review before reuse. |
| `no` | **Do not ingest** (see hard-excludes below). |

### Hard-excludes (do NOT ingest)
- **Maldita.es** (`latam_es`) — license explicitly **prohibits inclusion in AI/ML
  systems** without written consent.
- **AFP Factual / AFP Fact Check** (`latam_es`, `southeast_asia`) — © AFP, paid
  commercial license only.
- **Animal Político / El Sabueso** (`latam_es`) — ToS expressly prohibits
  automated scraping and commercial reproduction.
- **SUDEBAN Venezuela** (`latam_es`) — site currently **compromised** (RSS serving
  injected spam); source via trusted news mirrors until remediated.
- **ZimFact** (`africa`) — domain **suspended/inactive** as of 2026-06-21.
- **Semak Mule** (`southeast_asia`, MY) — operational per-query police lookup, not
  a redistributable dataset.

### Government content
Government advisories (the bulk of the corpus) are public-interest official
information under national transparency/access-to-information laws — reusable with
attribution. Explicit open licenses are rare but present: **GODL-India**
(data.gov.in), **Singapore Open Data Licence** (data.gov.sg), Colombia
**datos.gov.co** (CC-BY style). Brazilian gov content benefits from **LAI** and
copyright exclusion of official acts. Central banks sometimes attach restrictive
redistribution terms to non-statistical pages — treat those as cite-only.

### Open-licensed standouts (cleanest reuse)
- **Factly** (IN) — CC BY 4.0 (except videos).
- **PesaCheck** (Africa) — Creative Commons Attribution (via Code for Africa).
- **ChongLuaDao** GitHub mirror (VN) — BSD-3-Clause daily phishing blocklist.
- **US State Dept travel advisories** (SE Asia cross-border) — US federal public domain.
- **data.gov.sg / data.gov.in / datos.gov.co** — open-data licenses (aggregate stats).
- **CERT.br** (BR) — CC BY-NC-ND 4.0 (NC + ND restrict derivative/commercial use — flagged).

## Highest-value, machine-ingestible feeds (the shortlist)

Confirmed live feeds, prioritized for the pipeline:

**Structured alert-list / regulator feeds (RSS/JSON confirmed):**
- SEC Nigeria enforcement RSS · `home.sec.gov.ng/feeds/enforcement-updates.rss`
- FCCPC Nigeria RSS · `fccpc.gov.ng/feed/`
- CMA / CBK / SASRA Kenya RSS · `*.feed/`
- Peru `gob.pe` RSS family — INDECOPI, SMV, PECERT (curl-verified)
- Colombia `datos.gov.co` Socrata JSON API
- OpenSanctions mirrors of **BNM** & **SC Malaysia** alert lists (JSON, daily; CC-BY-NC)
- RBI (press + notifications) & SEBI RSS (IN)
- PIB press RSS (IN), data.gov.sg JSON API

**Scam-scoped fact-checker feeds (confirmed, `headlines_only`):**
- Chequeado AR `tag/estafas/feed/` · Bolivia Verifica `tag/estafas-digitales/feed/`
- GhanaFact `…/scams-spam/feed/` · Congo Check `tag/arnaque/feed/`
- Sebenarnya.my, TurnBackHoax.id, CekFakta.com, AFNC Thailand, ThaiCERT
- VERA Files PH, Dubawa, PesaCheck (CC-BY), Efecto Cocuyo, ColombiaCheck, Fast Check CL

**High-value sources with NO feed (need `html_scrape` / `pdf_periodic`):**
- MAS Investor Alert List (SG), OJK Satgas PASTI (ID), SEC Thailand Investor Alert,
  SEC Philippines advisories, FSCA warnings (ZA), CNBV blacklist PDF (MX),
  SFC Colombia, CMF Chile, SBS Peru, all unauthorized-entity registries across LatAm.

## Operational notes for the ingestion pipeline

- **Bot-blocking is widespread.** Many government domains return 403/503 to plain
  fetchers (anti-bot WAF, not dead links). Records flag this; plan a real
  browser User-Agent + rate-limiting, or a headless browser, for these.
- **Geo-blocking.** Vietnamese government portals may require a Vietnam egress IP.
- **Site-wide vs scoped feeds.** Many confirmed feeds are site-wide (news mixed in);
  filter by keywords (`estafa`, `fraude`, `golpe`, `scam`, `phishing`,
  `no autorizad*`, `penipuan`, etc.) or use category-scoped feeds where noted.
- **Re-validate feeds periodically** — `confirmed: false` entries and any feed can
  drift; sites migrate CMS (e.g. 211 Check → Wix, Africa Check `/rss.xml`).

## Provenance

All URLs and feeds were researched and (where marked `confirmed: true`) fetched
and validated on **2026-06-21**. Each record's `license_note` records the basis
for its `license_status`/`usable` tag. This catalog is a living document; the
`updated:` field in each regional file tracks last review.
