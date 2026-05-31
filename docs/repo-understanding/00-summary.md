# 00 — Repository Summary

## Purpose

`ukwa-heritrix` is the UK Web Archive's customised distribution of [Heritrix3](https://github.com/internetarchive/heritrix3), the Internet Archive's open-source web crawler. It extends Heritrix3 with UKWA-specific Java modules, a pre-built job configuration, and Docker packaging so that UKWA can run its legal-deposit web crawls in production.

The output is a Docker image. That image is the artefact shipped and operated.

## Technology Stack

| Layer | Technology |
|---|---|
| Crawler engine | Heritrix3 (v3.4.0) |
| Language / build | Java 8, Maven |
| Job configuration | Spring XML (`.cxml`) |
| URL feed (input) | Apache Kafka |
| Deduplication store | OutbackCDX |
| Frontier / URI uniqueness | BerkeleyDB, RocksDB, or Redis (selectable) |
| Virus scanning | ClamAV (clamd) |
| Browser rendering | Webrender Puppeteer + Warcprox |
| Monitoring | Prometheus (JMX exporter), Filebeat / ELK (optional) |
| Integration testing | Robot Framework |
| Packaging | Docker, Docker Compose |

## Main Runtime Model

The crawler runs as a single long-lived Heritrix3 process inside a Docker container. There are no traditional seed files; instead, URLs arrive continuously from a Kafka topic. The crawler fetches each URL, extracts outlinks, applies scope rules to decide which outlinks to re-enqueue, and writes WARC files. Results (crawl log entries) are streamed back to Kafka for downstream consumers.

## Main External Services / Dependencies

| Service | Role |
|---|---|
| Apache Kafka | URL input queue (`uris.tocrawl`) and output streams (`uris.crawled`, `uris.inscope`, etc.) |
| OutbackCDX | Records what has been crawled and when; used for de-duplication |
| ClamAV (`clamd`) | Virus-scans downloaded content |
| Webrender + Warcprox | Browser-based rendering of seed URLs, capturing via WARC proxy |
| Redis / RocksDB | Alternative frontier or URI-uniqueness-filter backends |

## Main Entry Points

- **`docker/bin/start`** — Shell script that is the container's `CMD`. Starts Heritrix, optionally launches or resumes a crawl job, and handles SIGTERM shutdown.
- **`jobs/frequent/crawler-beans.cxml`** — Spring context defining the entire crawl job: scope rules, fetch chain, write chain, Kafka wiring, OutbackCDX integration.
- **`src/main/java/uk/bl/wap/`** — Java source for all UKWA-specific modules (frontier adapters, extractors, decide rules, post-processors).

## Architecture Summary

UKWA-Heritrix is assembled from three layers:

1. **Heritrix3 base engine** — downloaded as a Maven dependency and unpacked into the Docker image. Provides the crawler framework, WARC writing, and HTTP fetching.

2. **UKWA Java modules** — compiled from `src/main/java/uk/bl/wap/` and placed in Heritrix's `lib/`. These replace or extend Heritrix's default components: a Kafka URL receiver replaces the seeds file; OutbackCDX processors replace the default BDB-based recrawl store; custom decide rules control scope; a virus-scan post-processor routes infected content to separate WARCs.

3. **Job configuration** — `jobs/frequent/crawler-beans.cxml` is a Spring XML file that wires all the beans together, reading runtime parameters from environment variables. Supporting XML fragments (`frontier-*.xml`, `uriuniqfilter-*.xml`) are imported to make the frontier and URI uniqueness filter swappable at deploy time.

## Read These First

| File / Directory | Why |
|---|---|
| `jobs/frequent/crawler-beans.cxml` | The complete crawl job definition — everything the crawler does is wired here |
| `docker/bin/start` | Container startup: how Heritrix is launched and how jobs are triggered |
| `src/main/java/uk/bl/wap/crawler/frontier/KafkaUrlReceiver.java` | How URLs enter the crawl from Kafka |
| `src/main/java/uk/bl/wap/modules/deciderules/` | Scope decision logic — what the crawler accepts or rejects |
| `src/main/java/uk/bl/wap/modules/recrawl/` | OutbackCDX integration for deduplication |
| `jobs/frequent/frontier-bdb.xml` / `frontier-redis.xml` | Selectable frontier backends |
| `docker-compose.yml` | Full integration environment definition |
| `Dockerfile` | How the image is built |
| `integration-test/robot/tests/crawl-test-site.robot` | Integration test that validates end-to-end crawl behaviour |
