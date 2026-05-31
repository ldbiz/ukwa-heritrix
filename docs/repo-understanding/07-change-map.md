# 07 — Change Map

A guide to where to look before making common categories of change.

---

## If I Need to Change Startup or Runtime Behaviour

**Likely files:**
- `docker/bin/start` — startup sequence, job launch/resume logic, SIGTERM handling
- `Dockerfile` — image build, installed tools, exposed ports, default environment variables
- `docker-compose.yml` — local integration environment, service wiring, volume mounts

**Why those files matter:**
`start` is the container entrypoint and controls everything before Heritrix loads its job. `Dockerfile` sets the build-time defaults. `docker-compose.yml` defines the full multi-service environment used for testing.

**Caveats:** The `LAUNCH_AUTOMATICALLY` env var controls whether and how the job auto-starts. The 45-second sleep is a fixed delay hardcoded in `start`.

---

## If I Need to Change Crawl Configuration

**Likely files:**
- `jobs/frequent/crawler-beans.cxml` — the primary place for any change to crawl behaviour: scope rules, chain processors, quotas, Kafka topics, WARC prefixes
- `jobs/frequent/sheets.xml` — per-site overrides (politeness, quotas, hop limits)
- `jobs/frequent/surts.txt` / `exclude.txt` — SURT allowlist and denylist (these can be changed on disk at runtime)
- `jobs/frequent/frontier-bdb.xml` / `frontier-redis.xml` — frontier backend settings
- `jobs/frequent/uriuniqfilter-*.xml` — URI uniqueness filter backend settings

**Why those files matter:**
`crawler-beans.cxml` is the Spring context that defines every component the crawler uses. Most configuration changes belong here. Scope-file changes can be applied without restart. Frontier/filter changes require selecting the correct XML file via environment variable.

**Caveats:** All bean properties read from env vars have defaults in the SpEL expressions. Adding a new env var requires updating both the `crawler-beans.cxml` and the `docker-compose.yml` (for integration testing).

---

## If I Need to Change Scope Logic

**Likely files:**
- `jobs/frequent/crawler-beans.cxml` — the `scope` bean and its `rules` list
- `src/main/java/uk/bl/wap/modules/deciderules/` — custom decide rule implementations
- `jobs/frequent/surts.txt` / `exclude.txt` — SURT prefix files

**Why those files matter:**
Scope rules are ordered in `crawler-beans.cxml`. The order matters — the last non-`NONE` decision wins. New rule types require a Java class in the `deciderules` package.

**Caveats:** Adding a new decide rule class requires rebuilding the Docker image. Changing SURT files does not.

---

## If I Need to Change the Fetch or Extraction Chain

**Likely files:**
- `jobs/frequent/crawler-beans.cxml` — the `fetchProcessors` bean list
- `src/main/java/uk/bl/wap/modules/extractor/` — custom link extractors
- `src/main/java/uk/bl/wap/crawler/processor/` — custom fetch-chain processors (WrenderProcessor, annotators)

**Why those files matter:**
The `fetchProcessors` list in `crawler-beans.cxml` defines what runs and in what order during the fetch phase. Custom extractors (JSON, sitemaps, GOV.UK API) are in `src/main/java/uk/bl/wap/modules/extractor/`.

**Caveats:** The `WrenderProcessor` exits the chain early on success (`ProcessResult.FINISH`). Processors after it are skipped for rendered URLs.

---

## If I Need to Change Post-Fetch Behaviour (WARC, Kafka, CDX)

**Likely files:**
- `jobs/frequent/crawler-beans.cxml` — the `dispositionProcessors` bean list
- `src/main/java/uk/bl/wap/crawler/postprocessor/` — `KafkaKeyedCrawlLogFeed`, `ForgettingFrontierProcessor`, `ViralContentProcessor`
- `src/main/java/uk/bl/wap/modules/writer/ExtendedWARCWriterProcessor.java` — WARC writing
- `src/main/java/uk/bl/wap/modules/recrawl/OutbackCDXPersistStoreProcessor.java` — CDX recording

**Why those files matter:**
The disposition chain is ordered, and each processor runs for every fetched URL. Changes to Kafka output format, WARC structure, or CDX recording logic all live here.

---

## If I Need to Change External Service Integration

**For Kafka:**
- `src/main/java/uk/bl/wap/crawler/frontier/KafkaUrlReceiver.java` — inbound URL feed
- `src/main/java/uk/bl/wap/crawler/postprocessor/KafkaKeyedCrawlLogFeed.java` — outbound crawl log
- `src/main/java/uk/bl/wap/crawler/postprocessor/KafkaKeyedToCrawlFeed.java` — outbound candidate feed
- `src/main/java/uk/bl/wap/crawler/postprocessor/KafkaKeyedDiscardedFeed.java` — outbound discarded feed

**For OutbackCDX:**
- `src/main/java/uk/bl/wap/util/OutbackCDXClient.java` — shared HTTP client
- `src/main/java/uk/bl/wap/modules/recrawl/OutbackCDXPersistStoreProcessor.java` — write path
- `src/main/java/uk/bl/wap/modules/deciderules/OutbackCDXRecentlySeenDecideRule.java` — read/scope path

**For Webrender:**
- `src/main/java/uk/bl/wap/crawler/processor/WrenderProcessor.java`

**For ClamAV:**
- `src/main/java/uk/bl/wap/crawler/postprocessor/ViralContentProcessor.java`
- `src/main/java/uk/bl/wap/util/ClamdScanner.java`

---

## If I Need to Change Tests or Fixtures

**For unit tests:**
- `src/test/java/uk/bl/wap/` — add or modify JUnit 4 tests

**For the integration test:**
- `integration-test/robot/tests/crawl-test-site.robot` — Robot Framework test suite
- `integration-test/robot/Dockerfile` / `requirements.txt` — test runner image

**For test fixtures:**
- `testdata/seed.json` — example Kafka message format
- `testdata/selftest/` — selftest Spring config (used by disabled `RedisFrontierSelfTest`)
- `src/test/resources/` — test resource files (e.g. EICAR test virus file)

**Caveats:** Several tests require live external services and are disabled by default (`@Ignore`). The integration test runs inside Docker Compose and requires building the full image first.
