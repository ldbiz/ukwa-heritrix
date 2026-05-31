# 06 — Tests and Fixtures

## Test Frameworks

| Framework | Scope | Location |
|---|---|---|
| JUnit 4 | Unit and integration tests for Java modules | `src/test/java/` |
| Robot Framework | End-to-end integration tests via Docker Compose | `integration-test/robot/` |

## Running Tests

**Unit tests:**
```
mvn clean install
```

**Integration tests (requires Docker):**
```
source source-setup-crawl-uid.sh
docker-compose build
docker-compose up
```

The `robot` container runs automatically when `docker-compose up` is invoked. It submits URLs via Kafka and then polls the Heritrix REST API until the crawl completes.

## Unit Test Files

| Test class | Behaviour covered |
|---|---|
| `ExtractorJsonTest` | `ExtractorJson` — whether JSON content types and `.json` URLs trigger link extraction; whether null-valued JSON fields are handled safely |
| `GovUkContentAPIExtractorTest` | GOV.UK Content API URL inference — correct API URL generation from page URLs |
| `RobotsTxSitemapExtractorTest` | Sitemap URL extraction from `robots.txt` |
| `AnnotationMatchesListRegexDecideRuleTest` | Annotation-based decide rule — correct ACCEPT/REJECT based on URI annotations |
| `ExternalGeoLocationDecideRuleTest` | GeoIP decide rule — IP-to-country lookup and country-code matching |
| `CompressibilityDecideRuleTest` | Compressibility-based reject rule — high-entropy (random) content is accepted; highly compressible (spam) content is rejected |
| `FixedSizeCacheUriUniqFilterTest` | LRU-cache-backed URI uniqueness filter |
| `RedisRecentlySeenUriUniqFilterTest` | Redis-backed URI uniqueness filter (requires Redis) |
| `WrenderProcessorTest` | Browser rendering processor (requires live Webrender) |
| `ViralContentProcessorTest` | ClamAV scanning (requires live clamd; `@Ignore`d by default) |
| `AMQPIndexableCrawlLogFeedTest` | AMQP crawl log feed (legacy) |
| `OutbackCDXClientTest` | OutbackCDX client — confirms that malformed URLs are not submitted for lookup |
| `RedisFrontierSelfTest` | Redis frontier end-to-end crawl (disabled: `runTest` is overridden to no-op) |

## Integration Test

**File:** `integration-test/robot/tests/crawl-test-site.robot`

This is a Robot Framework test suite that exercises the full crawl pipeline inside a Docker Compose environment. The test sequence is:

1. Wait for Heritrix to reach the `EMPTY` (idle) state.
2. Submit two seed URLs to the `fc.tocrawl` Kafka topic using the `crawl-streams submit` CLI.
   - `http://acid.matkelly.com/` — the Archival Acid Test site (a local Docker container).
   - `http://crawl-test-site.webarchive.org.uk` — the UKWA crawl test site (a local Docker container).
3. Wait for the crawl to reach the `RUNNING` state.
4. Wait (up to 5 minutes) for the crawl queue to empty (state `EMPTY`).

The test currently validates crawl state transitions only. The README notes that the CDX server could be queried to verify crawled URLs and their digest/content type, but this is not yet implemented as test assertions.

## Fixtures and Sample Data

| File / directory | What it represents |
|---|---|
| `testdata/selftest/conf/selftest-crawler-beans.cxml` | Minimal Spring job config used by the disabled `RedisFrontierSelfTest` |
| `testdata/selftest/conf/heritrix.properties` | Properties file for the selftest |
| `testdata/init.cdx` | CDX initialisation data |
| `testdata/seed.json` | Sample Kafka URL message format (JSON seed entry) |
| `src/test/resources/eicar.com.txt` | EICAR test virus file used by `ViralContentProcessorTest` |
| `docker/shared/domain-surts.txt` | Example SURT allowlist for a domain crawl |
| `jobs/frequent/surts.txt` | Default seed SURT list for the `frequent` job |
| `jobs/frequent/exclude.txt` | Default SURT denylist for the `frequent` job |
| `jobs/frequent/url.shorteners.txt` | Known URL shortener domains |

## Coverage Observations

- The scope/decide rule logic and extractors have reasonable unit test coverage.
- The OutbackCDX integration is tested indirectly via `OutbackCDXClientTest` (URL validation only) and through the integration test crawl.
- The Kafka URL receiver has no dedicated unit test; it is covered only by the integration test.
- The Redis frontier self-test is disabled.
- Tests that require live external services (ClamAV, Webrender, Redis) are either skipped or marked `@Ignore`.
