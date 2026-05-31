# 05 — Configuration and Environment

All runtime configuration is passed to the crawler via environment variables. These are read by the Spring XML context (`crawler-beans.cxml`) using SpEL expressions of the form `#{systemEnvironment['VAR'] ?: 'default'}`.

## Core Crawl Identity

| Variable | Default | Required | Effect |
|---|---|---|---|
| `CRAWL_NAME` | `frequent` | No | Identifies the crawl; used in WARC filenames and Kafka message fields |
| `JOB_NAME` | `frequent` | No | Heritrix job name; must match the directory under `/jobs/` |
| `HERITRIX_USER` | `heritrix` | No | Heritrix web UI / REST API username |
| `HERITRIX_PASSWORD` | `heritrix` | No | Heritrix web UI / REST API password |
| `USER_AGENT_PREFIX` | `bl.uk_ldfc_bot` | No | Prefix of the HTTP `User-Agent` header sent by the crawler |

## Startup Behaviour

| Variable | Default | Required | Effect |
|---|---|---|---|
| `LAUNCH_AUTOMATICALLY` | (unset) | No | Set to `true` to launch the job 45s after startup; set to `resume` to resume from checkpoint |
| `PAUSE_AT_START` | `false` | No | If `true`, the crawler starts paused and waits for a manual resume |
| `JAVA_OPTS` | `-Xmx2g` | No | JVM options; increase `-Xmx` for large crawls |
| `MAX_TOE_THREADS` | `200` | No | Maximum number of concurrent fetch threads |
| `MONITRIX_ENABLED` | `false` | No | Enables Filebeat log shipping to Monitrix (ELK) |

## Kafka

| Variable | Default | Required | Effect |
|---|---|---|---|
| `KAFKA_BOOTSTRAP_SERVERS` | `kafka:9092` | Yes (if using Kafka) | Kafka broker address |
| `KAFKA_TOCRAWL_TOPIC` | `uris.tocrawl` | No | Topic from which URLs are consumed |
| `KAFKA_GROUP_ID` | `crawlers` | No | Kafka consumer group |
| `KAFKA_CONSUMER_ID` | `1` | No | Partition offset for this crawler instance |
| `KAFKA_CONSUMER_GROUP_SIZE` | `1` | No | Total number of consumers in the group (for partition assignment) |
| `KAFKA_SEEK_TO_BEGINNING` | `false` | No | If `true`, replay all messages from the start of the topic |
| `KAFKA_MAX_POLL_RECORDS` | `500` | No | Max records fetched per Kafka poll |
| `KAFKA_NUM_MESSAGE_THREADS` | `16` | No | Threads processing incoming Kafka messages |
| `KAFKA_CRAWLED_TOPIC` | `uris.crawled` | No | Topic for crawl log output |
| `KAFKA_INSCOPE_TOPIC` | `uris.inscope` | No | Topic for accepted (in-scope) URL log |
| `KAFKA_DISCARDED_TOPIC` | `uris.discarded` | No | Topic for rejected URL log |
| `KAFKA_DISCARDED_FEED_ENABLED` | `false` | No | Enable publishing to the discarded topic |
| `KAFKA_INSCOPE_LOG_ENABLED` | `true` | No | Enable in-scope URL feed |
| `KAFKA_CANDIDATES_LOG_ENABLED` | `true` | No | Enable candidates outlinks feed |
| `KAFKA_CANDIDATES_TOPIC` | `uris.candidates` | No | Topic for discovered outlinks |
| `KAFKA_CANDIDATES_IN_SCOPE_ONLY` | `false` | No | Only publish in-scope outlinks |
| `KAFKA_CRAWL_LOG_ENABLED` | `true` | No | Enable crawl log Kafka feed |

## Scope

| Variable | Default | Required | Effect |
|---|---|---|---|
| `SURTS_SOURCE_FILE` | `surts.txt` | No | Path to the SURT allowlist file (relative to job dir or absolute) |
| `SURTS_EXCLUDE_SOURCE_FILE` | `exclude.txt` | No | Path to the SURT denylist file |
| `SCOPE_FILE_RELOAD_INTERVAL` | `5` | No | Seconds between checks for scope file changes |
| `MAX_HOPS_DEFAULT` | `20` | No | Maximum hop distance from a seed URL |
| `RECORD_DECIDING_RULE` | `false` | No | Log which scope rule made the final decision |
| `SCOPE_LOG_ENABLED` | `false` | No | Write scope decisions to a dedicated log file |
| `GEOIP_GB_ENABLED` | `false` | No | Enable GeoIP-based acceptance of UK-hosted URLs |
| `GEOIP_LOOKUP_EVERY_URI` | `false` | No | Run GeoIP lookup even for already-accepted URLs |
| `GEOLITE2_CITY_MMDB_LOCATION` | (empty) | No | Directory containing the GeoLite2 database file |
| `RECENTLY_SEEN_LOOKUP_EVERY_URI` | `false` | No | Run CDX lookup even for already-rejected URLs |
| `RECENTLY_SEEN_USE_LAUNCH_TIMESTAMP` | `true` | No | Use `launchTimestamp` from Kafka message to control recrawl timing |

## OutbackCDX

| Variable | Default | Required | Effect |
|---|---|---|---|
| `CDXSERVER_ENDPOINT` | `http://localhost:9090/fc` | Yes | URL of the OutbackCDX server and collection |
| `MAX_OUTBACKCDX_CONNECTIONS` | `400` | No | HTTP connection pool size for OutbackCDX |

## WARC Output

| Variable | Default | Required | Effect |
|---|---|---|---|
| `WARC_PREFIX` | `BL` | No | Prefix for WARC filenames |
| `WARC_WRITER_POOL_SIZE` | `10` | No | Number of concurrent WARC writers |
| `WARC_VIRAL_WRITER_POOL_SIZE` | `5` | No | Concurrent viral WARC writers |
| `WEBRENDER_WARC_PREFIX` | `WEBRENDER` | No | Prefix for WARCs produced by Webrender/Warcprox |

## Web Rendering

| Variable | Default | Required | Effect |
|---|---|---|---|
| `WEBRENDER_ENABLED` | `false` | No | Enable browser rendering via Webrender |
| `WEBRENDER_ENDPOINT` | `http://localhost:5000/wrender` | No | Webrender service URL |
| `WEBRENDER_CONNECT_TIMEOUT_MS` | `3600000` | No | Connection timeout for Webrender requests |
| `WEBRENDER_READ_TIMEOUT_MS` | `3600000` | No | Read timeout for Webrender requests |
| `WEBRENDER_MAX_TRIES` | `1` | No | Attempts before falling back to standard HTTP |

## Virus Scanning

| Variable | Default | Required | Effect |
|---|---|---|---|
| `CLAMD_ENABLED` | `false` | No | Enable ClamAV virus scanning |
| `CLAMD_HOST` | `localhost` | No | Hostname of the clamd daemon |

## Quotas

| Variable | Default | Required | Effect |
|---|---|---|---|
| `DEFAULT_HOST_MAX_SUCCESS_KB_QUOTA` | `-1` (unlimited) | No | Per-hostname download quota in KB |
| `DEFAULT_SERVER_MAX_SUCCESS_KB_QUOTA` | `524288` (512 MiB) | No | Per-server (host+port) download quota in KB |
| `RETIRE_QUEUES` | `false` | No | If `true`, retire host queues on quota exhaustion rather than just throttling |

## Frontier and Persistence

| Variable | Default | Required | Effect |
|---|---|---|---|
| `FRONTIER_BEANS_XML` | `frontier-bdb.xml` | No | Which frontier implementation to load |
| `URIUNIQFILTER_BEANS_XML` | `uriuniqfilter-bdb.xml` | No | Which URI uniqueness filter to load |
| `REDIS_ENDPOINT` | `redis://redis:6379` | No (Redis frontier only) | Redis server URL |
| `URI_FORGETTING_ENABLED` | `true` | No | Enable URI forgetting after crawl (required for continuous recrawling) |
| `CHECKPOINT_INTERVAL_MINUTES` | `1440` | No | How often to write a checkpoint (minutes) |
| `CHECKPOINT_FORGET_ALL_BUT_LATEST` | `false` | No | If `true`, delete old checkpoints after each new one |
| `FRONTIER_JE_CLEANER_THREADS` | `1` | No | BerkeleyDB background cleaner threads |
| `MAX_RETRIES` | `10` | No | Maximum fetch retries before giving up on a URL |

## Config Files on Disk

| File | Location in container | Purpose |
|---|---|---|
| `crawler-beans.cxml` | `/jobs/frequent/` | Main Spring job configuration |
| `surts.txt` / `exclude.txt` | `/jobs/frequent/` or `/shared/` | SURT scope allow/deny lists (hot-reloadable) |
| `sheets.xml` | `/jobs/frequent/` | Per-site crawl behaviour overrides |
| `url.shorteners.txt` | `/jobs/frequent/` | Known URL shortening service domains to always accept |
| `logging.properties` | `/h3-bin/conf/` | Java logging configuration |
| `filebeat.yml` | `/etc/filebeat/` | Filebeat log shipping config (for Monitrix) |
| `heritrix-jmx-config.yml` | Mounted at `/jmx-config.yml` | JMX Prometheus exporter configuration |

## Secrets and Credentials

- `HERITRIX_USER` / `HERITRIX_PASSWORD` — default values are well-known; must be changed in production.
- No other credentials are hard-coded in the repository. OutbackCDX and Kafka endpoints are assumed to be accessible without authentication in the default configuration.
