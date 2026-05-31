# 04 — Behaviour Walkthroughs

## 1. URL Feed: Receiving and Scoping a URL from Kafka

**Trigger:** An external system (e.g. the UKWA crawl orchestration or the `crawl-streams` CLI tool) publishes a JSON message to the `uris.tocrawl` Kafka topic.

**Files involved:**
- `src/main/java/uk/bl/wap/crawler/frontier/KafkaUrlReceiver.java`
- `jobs/frequent/crawler-beans.cxml` (the `scope` bean and `candidateProcessors`)
- `src/main/java/uk/bl/wap/modules/deciderules/WatchedFileSurtPrefixedDecideRule.java`
- `src/main/java/uk/bl/wap/modules/deciderules/OutbackCDXRecentlySeenDecideRule.java`

**Flow:**

`KafkaUrlReceiver` runs a pool of consumer threads that continuously poll the Kafka topic. Each message is parsed as a JSON object containing the URL, whether it is a seed, hop type, and optional per-URL metadata (including `launchTimestamp` for controlling recrawl timing).

The URL is wrapped in a `CrawlURI` and submitted to the Heritrix frontier. Before it enters the download queue, it passes through the Candidate Chain, which applies scope rules in order:

1. Reject by default.
2. Accept if the URL's SURT prefix is in `surts.txt`.
3. Accept if the URL is a media/asset extension.
4. Accept if the URL is an embed or redirect from an already-accepted URL.
5. Reject if SURT prefix is in `exclude.txt`.
6. Reject if too many hops from seed.
7. Reject if recently crawled (OutbackCDX lookup).

**Inputs:** JSON message on Kafka topic (URL, isSeed, hop, forceFetch, launchTimestamp, headers).

**Outputs:** URL enqueued in the frontier (if accepted), or silently discarded. Accepted URLs are also published to `uris.inscope`.

**Side effects:** OutbackCDX is queried for each URL to check if it was recently crawled. If `KAFKA_DISCARDED_FEED_ENABLED=true`, rejected URLs are published to `uris.discarded`.

---

## 2. Fetching a URL: HTTP Download and WARC Writing

**Trigger:** The frontier selects a URL from the queue for download.

**Files involved:**
- `jobs/frequent/crawler-beans.cxml` (`fetchProcessors`, `dispositionProcessors`)
- `src/main/java/uk/bl/wap/crawler/postprocessor/ViralContentProcessor.java`
- `src/main/java/uk/bl/wap/modules/writer/ExtendedWARCWriterProcessor.java`
- `src/main/java/uk/bl/wap/modules/recrawl/OutbackCDXPersistStoreProcessor.java`

**Flow:**

The Fetch Chain runs processors in sequence:

1. Quota enforcement: if the host has exceeded its download quota, the URL is deferred.
2. Preconditions: robots.txt must be fetched before any URL on a new host.
3. Optional WebRender: if the URL is a seed or annotated `WebRenderThis`, the Webrender service is called. On success, the chain exits early and Heritrix does not fetch the URL itself (Warcprox records the traffic instead).
4. HTTP fetch via standard `FetchHTTP`.
5. Link extraction from HTTP headers, HTML, CSS, XML, sitemaps.
6. IP and country-code annotation.
7. ClamAV virus scan.

The Disposition Chain then:

1. Removes the URL from the URI uniqueness filter (so it can be re-crawled later).
2. Publishes the crawl outcome as a JSON log record to `uris.crawled`.
3. Routes to WARC writer — clean content goes to `warcs/`, virus-annotated content goes to `viral/`.
4. Records URL + SHA-1 digest + timestamp in OutbackCDX.
5. Passes discovered outlinks through the candidate chain.

**Inputs:** URL from frontier queue, HTTP response body, server headers.

**Outputs:** WARC record on disk, CDX entry in OutbackCDX, Kafka message on `uris.crawled`, outlinks enqueued in the frontier.

**External calls:** ClamAV socket, OutbackCDX HTTP API, Webrender HTTP endpoint (if enabled).

**Tests:** `ViralContentProcessorTest` (unit, requires live clamd); integration test `crawl-test-site.robot`.

---

## 3. Scope File Hot-Reload

**Trigger:** The `surts.txt` or `exclude.txt` file is modified on disk while the crawler is running.

**Files involved:**
- `src/main/java/uk/bl/wap/modules/deciderules/WatchedFileSurtPrefixedDecideRule.java`
- `jobs/frequent/surts.txt`, `jobs/frequent/exclude.txt`

**Flow:**

`WatchedFileSurtPrefixedDecideRule` extends Heritrix's standard `SurtPrefixedDecideRule` and adds a background file-watcher thread. Every `sourceCheckInterval` seconds (default: 5), it checks whether the source file's modification time has changed. If it has, it reloads the SURT prefix set into memory and applies it immediately to subsequent scope decisions.

This allows operators to add or remove domains from the crawl scope without restarting the crawler or reloading the Spring context.

**Inputs:** Modified SURT file on disk (`/shared/surts.txt` or the configured path).

**Outputs:** Updated in-memory SURT prefix set; subsequent crawl decisions reflect the new scope.

**Side effects:** None — the crawl continues uninterrupted during the reload.

---

## 4. Browser Rendering via Webrender

**Trigger:** A URL passes scope checks and either (a) is a seed, or (b) carries the `WebRenderThis` annotation.

**Files involved:**
- `src/main/java/uk/bl/wap/crawler/processor/WrenderProcessor.java`
- `docker-compose.yml` (`webrender` and `warcprox` services)

**Flow:**

`WrenderProcessor` runs early in the Fetch Chain. For eligible URLs, it makes an HTTP request to the Webrender endpoint, passing the URL. Webrender launches a headless Chromium browser that renders the page through Warcprox, which intercepts and records all HTTP/HTTPS traffic as WARC files.

If rendering succeeds, `WrenderProcessor` returns `ProcessResult.FINISH`, which skips the remaining fetch chain processors (including the standard `FetchHTTP`). The rendered WARC is written by Warcprox, not by Heritrix directly.

If rendering fails (timeout or error), the processor falls through, and Heritrix falls back to its own HTTP fetch.

Warcprox also:
- Publishes captured URLs to `uris.crawled` via a Kafka plugin.
- Records captures in OutbackCDX via its own plugin.

**Inputs:** URL eligible for rendering, Webrender HTTP endpoint.

**Outputs:** WARC files in `/heritrix/wren/` (written by Warcprox), CDX entries from Warcprox.

**External calls:** Webrender HTTP API, Warcprox as HTTPS proxy.

**Tests:** `WrenderProcessorTest` (unit, requires running Webrender).

---

## 5. Crawl Result Published to Kafka

**Trigger:** A URL completes the Disposition Chain.

**Files involved:**
- `src/main/java/uk/bl/wap/crawler/postprocessor/KafkaKeyedCrawlLogFeed.java`

**Flow:**

`KafkaKeyedCrawlLogFeed` runs early in the Disposition Chain, before WARC writing. It serialises the `CrawlURI`'s fetch outcome as a JSON object (using Heritrix's `CrawlLogJsonBuilder`) and publishes it to the `uris.crawled` Kafka topic. The record key is a hash of the URL, ensuring that messages for the same URL go to the same partition.

Extra fields such as `crawl_name` are merged into the JSON from configuration, allowing downstream consumers to distinguish records from different crawl runs.

**Inputs:** Completed `CrawlURI` with fetch status, content digest, content type, IP, hop path, and annotations.

**Outputs:** JSON message on `uris.crawled` Kafka topic.

**Side effects:** Downstream consumers (CDX indexers, document processors, analytics pipelines) read from this topic.
