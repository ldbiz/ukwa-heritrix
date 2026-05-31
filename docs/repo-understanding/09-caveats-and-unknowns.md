# 09 — Caveats and Unknowns

## External Systems Not Present in This Repo

- **UKWA crawl orchestration** — The system that decides which URLs to submit to Kafka is not part of this repository. How seeds are generated, when recrawls are triggered, and how `launchTimestamp` is set are all managed externally.
- **Monitrix / ELK stack** — The Filebeat configuration exists in the repo, but the ELK destination and log schema are not documented here.
- **Prometheus / alerting** — The JMX exporter config (`heritrix-jmx-config.yml`) is present, but the Prometheus server, dashboards, and alerting rules are external.
- **Production volume mounts** — WARC output paths, scope file locations, and state directories in production are not specified in this repo.

## Ambiguous or Uncertain Behaviour

- **`StreamScoper`** — This standalone application in `src/main/java/uk/bl/wap/scoper/` exists in the codebase but is not wired into `docker-compose.yml` or any documented deployment. Its role appears to be a pre-filter for a domain crawl, but whether it is used in production is unclear.

- **`ForceFetchBasedOnHopPath`** — This prefetch processor is present in `src/main/java/uk/bl/wap/crawler/prefetch/` but does not appear in `crawler-beans.cxml`. It is not clear whether it is intentionally disabled or simply unused.

- **AMQP integration** — `AMQPUrlReceiver` and `AMQPIndexableCrawlLogFeed` are in the source tree and have a test. The `docker-compose.yml` and `crawler-beans.cxml` comment out these beans. It is unclear whether AMQP is still used in any UKWA environment.

- **`HashingCrawlMapper`** — Present in `src/main/java/uk/bl/wap/crawler/processor/` but not referenced in `crawler-beans.cxml`. Purpose unknown from the code alone.

- **`BufferedCrawlerLoggerModule`** and `BufferedGenerationFileHandler`** — In `org.archive.crawler.reporting` / `org.archive.io`. These are in the repo under `src/main/java/org/archive/` (upstream namespace), suggesting they are backports or patches to the upstream Heritrix code. The reason for the patch is not documented.

- **`ObeyRobotsExceptTranscludedPolicy`** — A custom robots.txt policy class is present and commented-out in `crawler-beans.cxml`. The `robotsPolicyName` is currently set to `"classic"`, not this custom class.

## Behaviour Visible in Code but Not Covered by Tests

- **`KafkaUrlReceiver`** — The URL intake mechanism has no dedicated unit test. Coverage relies entirely on the integration test.
- **`ForgettingFrontierProcessor`** — URI forgetting is not tested in isolation.
- **`OutbackCDXRecentlySeenDecideRule`** — The retry-on-failure logic (3 attempts, 30-second sleep) is not tested.
- **`WatchedFileSurtPrefixedDecideRule`** — The file-watching / hot-reload mechanism is not tested.
- **`RedisSimplifiedFrontier`** — The Redis frontier self-test is explicitly disabled.
- **Viral WARC routing** — The interaction between `ViralContentProcessor` and `warcWriterViral` is not covered by any test that runs in CI.

## Questions a Maintainer Should Answer

1. Is `StreamScoper` actively deployed in production? If not, can it be removed or should it be documented?
2. Is the AMQP integration still in use, or can the AMQP source files be removed?
3. Why are `BufferedCrawlerLoggerModule` and `BufferedGenerationFileHandler` patched in this repository rather than contributed upstream?
4. What is the intended use of `HashingCrawlMapper`?
5. Is `ForceFetchBasedOnHopPath` intentionally disabled, or is it missing from `crawler-beans.cxml` by accident?
6. What Heritrix job names other than `frequent` are used in production? Is the `JOB_NAME` variable ever set to something else?
7. Is there a documented list of all Kafka topic names used in production, and do they match the defaults in `crawler-beans.cxml`?
8. The integration test only checks crawl state transitions. Are there plans to add CDX-based assertions to verify that specific URLs were actually crawled and archived?
