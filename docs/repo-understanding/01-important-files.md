# 01 — Important Files

## Entrypoints

| File / Directory | Role | Why it matters |
|---|---|---|
| `docker/bin/start` | Container `CMD` entrypoint | Controls how Heritrix starts, how the crawl job is launched or resumed, and how SIGTERM is handled |
| `Dockerfile` | Image build recipe | Compiles UKWA modules, unpacks Heritrix, copies jobs and config into the image |
| `jobs/frequent/crawler-beans.cxml` | Spring bean context for the `frequent` crawl job | Defines every processing component and how they connect — the effective "program" the crawler runs |

## Configuration

| File / Directory | Role | Why it matters |
|---|---|---|
| `jobs/frequent/frontier-bdb.xml` | BerkeleyDB frontier bean definition | Default frontier; imported by `crawler-beans.cxml` unless overridden at runtime |
| `jobs/frequent/frontier-redis.xml` | Redis-backed frontier bean definition | Alternative frontier for large-scale or distributed crawling; selectable via env var |
| `jobs/frequent/uriuniqfilter-bdb.xml` / `uriuniqfilter-rocksdb.xml` / `uriuniqfilter-bloom.xml` | URI uniqueness filter variants | Swappable backends that control how "already scheduled" URIs are tracked |
| `jobs/frequent/sheets.xml` | Per-site crawl behaviour overrides | Associates specific SURT prefixes with named "sheets" that change politeness, quotas, or hop limits for those sites |
| `jobs/frequent/surts.txt` / `exclude.txt` | SURT allowlist and denylist seeds | Initial scope definitions; can be updated on disk while the crawler is running |
| `docker-compose.yml` | Integration environment definition | Declares all services (Heritrix, Kafka, OutbackCDX, Warcprox, Webrender, test sites, robot tester) and their connections |
| `docker/bin/` | Runtime helper scripts | `job-launch`, `job-pause`, `job-resume`, `job-stop`, `job-info`, `job-kafka-report` — all thin curl wrappers around the Heritrix REST API |

## Core Application Logic

| File / Directory | Role | Why it matters |
|---|---|---|
| `src/main/java/uk/bl/wap/crawler/frontier/KafkaUrlReceiver.java` | Kafka consumer that feeds URLs into the crawl | Replaces the traditional seeds file; the crawler's URL input mechanism |
| `src/main/java/uk/bl/wap/modules/deciderules/` | Scope-decision rules | Determines whether a discovered URL is accepted or rejected for crawling |
| `src/main/java/uk/bl/wap/modules/deciderules/WatchedFileSurtPrefixedDecideRule.java` | Live-reloading SURT scope rule | Polls a SURT file on disk and updates the crawl scope without restarting |
| `src/main/java/uk/bl/wap/modules/deciderules/OutbackCDXRecentlySeenDecideRule.java` | Deduplication via OutbackCDX | Rejects URLs that have been crawled within the configured recrawl interval |
| `src/main/java/uk/bl/wap/modules/recrawl/OutbackCDXPersistStoreProcessor.java` | Records crawl results to OutbackCDX | Writes URL+digest+timestamp after each successful fetch |
| `src/main/java/uk/bl/wap/crawler/postprocessor/KafkaKeyedCrawlLogFeed.java` | Publishes crawl log to Kafka | Streams crawl outcomes to `uris.crawled` for downstream consumers |
| `src/main/java/uk/bl/wap/crawler/postprocessor/ViralContentProcessor.java` | Virus scanning | Scans fetched content via ClamAV; annotates infected content so it is routed to separate WARCs |
| `src/main/java/uk/bl/wap/crawler/processor/WrenderProcessor.java` | Browser rendering integration | Delegates seed and annotated URLs to the Webrender service instead of fetching directly |
| `src/main/java/uk/bl/wap/crawler/postprocessor/ForgettingFrontierProcessor.java` | URI forgetting | Removes a crawled URI from the uniqueness filter, enabling re-crawl in continuous operation |
| `src/main/java/uk/bl/wap/modules/writer/ExtendedWARCWriterProcessor.java` | WARC output | Writes fetched content to WARC files; excludes virus-annotated records |
| `src/main/java/uk/bl/wap/scoper/StreamScoper.java` | Standalone scope processor | Optional standalone process that pre-filters a Kafka candidate stream using the same scope rules (experimental) |

## External Integrations

| File / Directory | Role | Why it matters |
|---|---|---|
| `src/main/java/uk/bl/wap/util/OutbackCDXClient.java` | HTTP client for OutbackCDX | Shared client used by both the deduplication rule and the persist-store processor |
| `src/main/java/uk/bl/wap/crawler/frontier/RedisSimplifiedFrontier.java` | Redis-backed frontier | Alternative frontier that uses Redis (or a compatible store like KVRocks) rather than BerkeleyDB |
| `src/main/java/uk/bl/wap/modules/deciderules/ExternalGeoLocationDecideRule.java` | GeoIP-based scope rule | Accepts URLs whose server IP resolves to a configured country (e.g. GB) |

## Tests and Fixtures

| File / Directory | Role | Why it matters |
|---|---|---|
| `src/test/java/uk/bl/wap/` | JUnit unit tests | Tests for extractors, decide rules, post-processors, and frontier components |
| `integration-test/robot/tests/crawl-test-site.robot` | Robot Framework integration test | End-to-end test: submits URLs to Kafka, waits for the crawl to complete, checks Heritrix state |
| `testdata/selftest/` | Heritrix self-test config | Spring config and site files used by the `RedisFrontierSelfTest` |
