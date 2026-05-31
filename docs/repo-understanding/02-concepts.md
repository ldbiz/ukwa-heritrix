# 02 — Concepts

A glossary of the domain-specific concepts used in this repository.

---

## SURT (Sort-friendly URI Reordering Transform)

A normalised form of a URL used throughout Heritrix and UKWA tooling for scope matching.

In SURT notation, domain components are reversed and parenthesised, so `http://www.bl.uk/path/` becomes `http://(uk,bl,www,)/path/`. This makes prefix matching stable and allows hierarchical domain scoping without regular expressions.

**Where it appears:** `jobs/frequent/surts.txt`, `jobs/frequent/exclude.txt`, `WatchedFileSurtPrefixedDecideRule`, `sheets.xml`

---

## Scope / Decide Rules

The set of rules that determine whether a discovered URL will be crawled or discarded.

Rules in Heritrix3 are chained in an ordered list. Each rule can return `ACCEPT`, `REJECT`, or `NONE` (pass to next). The last non-`NONE` decision wins. UKWA's `AccountableDecideRuleSequence` logs which rule made the final decision.

The main scope accepts URLs that:
- Match a SURT prefix in the allowlist (`surts.txt`)
- Are common asset extensions (CSS, JS, images)
- Are embeds or redirects from an already-accepted URL

And rejects URLs that:
- Are in the SURT exclude list (`exclude.txt`)
- Have too many hops from a seed
- Match pathological or blacklisted URL patterns
- Have been crawled recently (checked against OutbackCDX)

**Where it appears:** `jobs/frequent/crawler-beans.cxml` (the `scope` bean), `src/main/java/uk/bl/wap/modules/deciderules/`

---

## Candidate Chain, Fetch Chain, Disposition Chain

The three ordered processor pipelines that a CrawlURI passes through.

| Chain | When it runs | What it does |
|---|---|---|
| **Candidate Chain** | When a URL is first discovered | Applies scope rules; enqueues accepted URLs into the frontier |
| **Fetch Chain** | When a URL's turn comes up | Quota checks, DNS lookup, web rendering, HTTP fetch, link extraction, virus scan |
| **Disposition Chain** | After fetch | Records crawl result, writes WARCs, publishes to Kafka, processes outlinks |

**Where it appears:** `candidateProcessors`, `fetchProcessors`, `dispositionProcessors` beans in `crawler-beans.cxml`

---

## Frontier

The data structure that manages the queue of pending URLs, respecting politeness delays and per-host quotas.

UKWA supports three frontier backends, selected by environment variable:

| Backend | Config file | Notes |
|---|---|---|
| BerkeleyDB (`BdbFrontier`) | `frontier-bdb.xml` | Default; persistent, single-node |
| Redis (`SimplifiedFrontierAdaptor`) | `frontier-redis.xml` | Distributed-ready; uses Redis or KVRocks |
| (In-memory, via standard Heritrix) | n/a | Available but not configured here |

**Where it appears:** `jobs/frequent/frontier-*.xml`, imported into `crawler-beans.cxml` via `${FRONTIER_BEANS_XML:frontier-bdb.xml}`

---

## URI Uniqueness Filter

A data structure used by the frontier to track which URLs have already been scheduled, so they are not enqueued twice.

In continuous crawling, UKWA uses the `ForgettingFrontierProcessor` to _remove_ a URL from this filter after it has been fetched, enabling periodic re-crawling of the same URLs.

Available backends:

| Backend | Class | Notes |
|---|---|---|
| BerkeleyDB | `BdbUriUniqFilter` (standard H3) | Default |
| RocksDB | `RockDBUriUniqFilter` | Better performance for large crawls |
| Bloom filter | `CountingBloomUriUniqFilter` | Probabilistic; may have false positives |
| Redis | `RedisRecentlySeenUriUniqFilter` | Distributed |

**Where it appears:** `jobs/frequent/uriuniqfilter-*.xml`, `src/main/java/uk/bl/wap/modules/uriuniqfilters/`

---

## OutbackCDX

An open-source CDX (Capture InDeX) server used by UKWA to record the outcome of every crawl fetch.

UKWA uses OutbackCDX for two purposes:

1. **Deduplication / "recently seen"** — Before crawling a URL, the `OutbackCDXRecentlySeenDecideRule` queries OutbackCDX. If the URL was crawled within the recrawl interval, it is rejected.
2. **Crawl record** — After a successful fetch, `OutbackCDXPersistStoreProcessor` writes the URL, timestamp, and content digest to OutbackCDX.

**Where it appears:** `src/main/java/uk/bl/wap/modules/recrawl/`, `src/main/java/uk/bl/wap/modules/deciderules/OutbackCDXRecentlySeenDecideRule.java`, `outbackCDXClient` bean in `crawler-beans.cxml`

---

## WARC (Web ARChive format)

The archive file format in which crawled content is stored, one HTTP response per record.

UKWA writes two separate WARC streams:

- **Normal WARCs** — all clean fetched content, written by `ExtendedWARCWriterProcessor` to `/heritrix/output/<crawl_name>/<launchId>/warcs/`.
- **Viral WARCs** — content flagged by ClamAV, written by `WARCViralWriterProcessor` to `/heritrix/output/<crawl_name>/<launchId>/viral/`. These records are XOR-encoded to prevent accidental access.

**Where it appears:** `warcWriterDefault`, `warcWriterViral` beans in `crawler-beans.cxml`, `src/main/java/uk/bl/wap/modules/writer/`

---

## Kafka Topics

Kafka is used as the crawler's URL bus — all URL feeds in and out flow through Kafka topics.

| Topic (default name) | Direction | Content |
|---|---|---|
| `uris.tocrawl` | In | URLs to be crawled (seeds and rediscovered outlinks) |
| `uris.crawled` | Out | Crawl log entries (one per fetch) |
| `uris.inscope` | Out | URLs accepted by scope before fetch |
| `uris.discarded` | Out | URLs rejected by scope (if enabled) |
| `uris.candidates` | Out | All discovered outlinks (if enabled) |

Topic names are configurable via environment variables.

**Where it appears:** `KafkaUrlReceiver`, `KafkaKeyedCrawlLogFeed`, `KafkaKeyedToCrawlFeed`, `KafkaKeyedDiscardedFeed` in `crawler-beans.cxml`

---

## Sheets

A Heritrix3 mechanism for applying per-site configuration overrides without restarting.

A "sheet" is a named set of property overrides that can be associated with specific SURT prefixes. For example, UKWA uses sheets to enforce form-submission rules, extra politeness delays, or download quotas for particular domains.

**Where it appears:** `jobs/frequent/sheets.xml`

---

## Hop Path

A string encoding the sequence of link types followed from a seed to a given URL. Each character represents one hop:
- `L` — navigational link
- `E` — embed (image, script, stylesheet)
- `R` — redirect
- `P` — prerequisite (robots.txt, DNS)

The hop path appears in scope rules (e.g. "always follow embeds and redirects") and in crawl log entries.

**Where it appears:** Scope rules in `crawler-beans.cxml`, `KafkaUrlReceiver` (reads hop from incoming message)

---

## Webrender / Warcprox

Two cooperating services used for browser-based crawling of JavaScript-heavy pages.

- **Webrender** runs a headless Chromium browser via Puppeteer and renders pages via a configurable HTTP endpoint.
- **Warcprox** is an HTTPS-intercepting proxy that records all traffic the browser makes into WARC files.

When the `WrenderProcessor` is enabled, seeds (and URLs annotated `WebRenderThis`) are sent to Webrender instead of being fetched directly by Heritrix. If rendering succeeds, the rest of the Heritrix fetch chain is skipped.

**Where it appears:** `wrenderHttp` bean in `crawler-beans.cxml`, `src/main/java/uk/bl/wap/crawler/processor/WrenderProcessor.java`, `webrender` and `warcprox` services in `docker-compose.yml`

---

## Viral Content Processor

A post-processor that streams each fetched response body to a local ClamAV daemon for virus scanning.

If a virus is found, the URI is annotated with a `stream:<virusname>FOUND` tag. The annotation is then used by the WARC writer beans to route the infected response to the `viral/` WARC stream instead of the normal stream.

**Where it appears:** `viralContent` bean in `crawler-beans.cxml`, `src/main/java/uk/bl/wap/crawler/postprocessor/ViralContentProcessor.java`

---

## StreamScoper

An experimental standalone Java process (in `src/main/java/uk/bl/wap/scoper/`) that applies the Heritrix scope rules to a Kafka stream _outside_ the crawler. The intent is to filter candidate URLs before they reach the crawl download frontier, improving efficiency for large domain crawls.

This component is present in the codebase but is not wired into the main `docker-compose.yml` and appears to be experimental.

**Where it appears:** `src/main/java/uk/bl/wap/scoper/`
