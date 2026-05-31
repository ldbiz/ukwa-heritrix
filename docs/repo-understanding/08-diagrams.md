# 08 — Diagrams

## 1. Runtime Flow

How the crawler starts, receives URLs, and processes them end to end.

```mermaid
flowchart TD
    A["docker/bin/start\n(container CMD)"] --> B["Heritrix3 process\nstarts"]
    B --> C["Spring context loads\ncrawler-beans.cxml"]
    C --> D["KafkaUrlReceiver\nsubscribes to\nuris.tocrawl"]
    C --> E["Job launched\nvia REST API\n(if LAUNCH_AUTOMATICALLY=true)"]

    D -->|JSON URL message| F["CrawlURI\ncreated"]
    F --> G["Candidate Chain"]

    subgraph G["Candidate Chain"]
        G1["SURT scope check\n(WatchedFileSurtPrefixedDecideRule)"]
        G2["OutbackCDX\nrecently-seen check"]
        G3["In-scope feed\n→ uris.inscope"]
        G1 --> G2 --> G3
    end

    G -->|accepted| H["Frontier\n(BDB / Redis)"]
    G -->|rejected| I["Discarded\n(optional → uris.discarded)"]

    H --> J["Fetch Chain"]

    subgraph J["Fetch Chain"]
        J1["Quota check"]
        J2["Preconditions\n(robots.txt)"]
        J3["WrenderProcessor\n(optional)"]
        J4["FetchHTTP"]
        J5["Link extractors\n(HTML/CSS/XML/JSON/sitemap)"]
        J6["ClamAV virus scan"]
        J1 --> J2 --> J3 -->|if render fails| J4 --> J5 --> J6
    end

    J --> K["Disposition Chain"]

    subgraph K["Disposition Chain"]
        K1["URI forgetting"]
        K2["Crawl log → uris.crawled"]
        K3["WARC writer\n(normal or viral)"]
        K4["OutbackCDX persist"]
        K5["Outlinks → Candidate Chain"]
        K1 --> K2 --> K3 --> K4 --> K5
    end

    K3 --> M["WARC files\n/heritrix/output"]
    K4 --> N["OutbackCDX"]
    K2 --> L["Kafka: uris.crawled"]
    K5 -->|outlinks| G
```

---

## 2. Module / File Interaction

How the main Java modules and configuration files relate to each other.

```mermaid
graph LR
    cxml["crawler-beans.cxml"]

    subgraph Frontier
        fbdb["frontier-bdb.xml\n(BdbFrontier)"]
        fredis["frontier-redis.xml\n(SimplifiedFrontierAdaptor)"]
        redisFrontier["RedisSimplifiedFrontier"]
    end

    subgraph URIFilter
        ubdb["uriuniqfilter-bdb.xml"]
        urocksdb["uriuniqfilter-rocksdb.xml\n(RockDBUriUniqFilter)"]
        ubloom["uriuniqfilter-bloom.xml"]
        redis_uq["RedisRecentlySeenUriUniqFilter"]
    end

    subgraph DecideRules
        surt["WatchedFileSurtPrefixedDecideRule"]
        cdx_rule["OutbackCDXRecentlySeenDecideRule"]
        geo["ExternalGeoLocationDecideRule"]
    end

    subgraph Processors
        kafka_in["KafkaUrlReceiver"]
        wrender["WrenderProcessor"]
        viral["ViralContentProcessor"]
        forgetter["ForgettingFrontierProcessor"]
        kafka_log["KafkaKeyedCrawlLogFeed"]
        inscope["KafkaKeyedToCrawlFeed"]
        cdx_store["OutbackCDXPersistStoreProcessor"]
        warc["ExtendedWARCWriterProcessor"]
    end

    subgraph ExternalServices
        kafka[("Kafka")]
        cdx[("OutbackCDX")]
        clamd[("ClamAV")]
        webrender_svc[("Webrender")]
    end

    cxml --> fbdb
    cxml --> fredis
    fredis --> redisFrontier
    cxml --> ubdb
    cxml --> urocksdb
    cxml --> ubloom
    cxml --> surt
    cxml --> cdx_rule
    cxml --> geo
    cxml --> kafka_in
    cxml --> wrender
    cxml --> viral
    cxml --> forgetter
    cxml --> kafka_log
    cxml --> inscope
    cxml --> cdx_store
    cxml --> warc

    kafka_in --> kafka
    kafka_log --> kafka
    inscope --> kafka

    cdx_rule --> cdx
    cdx_store --> cdx

    viral --> clamd
    wrender --> webrender_svc
```

---

## 3. Sequence Diagram: URL Crawled End-to-End

The most important single behaviour: a URL arrives from Kafka and is fetched and recorded.

```mermaid
sequenceDiagram
    participant Ops as Operator / Orchestration
    participant Kafka as Kafka (uris.tocrawl)
    participant KUR as KafkaUrlReceiver
    participant Scope as Scope / Candidate Chain
    participant CDX as OutbackCDX
    participant Frontier as Frontier (BDB/Redis)
    participant Fetch as Fetch Chain
    participant Clamd as ClamAV
    participant WARCWriter as WARC Writer
    participant KafkaOut as Kafka (uris.crawled)

    Ops->>Kafka: Publish URL message (JSON)
    KUR->>Kafka: Poll messages
    Kafka-->>KUR: URL message
    KUR->>Scope: Submit CrawlURI
    Scope->>CDX: Query: was URL crawled recently?
    CDX-->>Scope: No / yes
    alt In scope and not recently seen
        Scope->>Frontier: Enqueue URL
        Scope->>KafkaOut: Publish to uris.inscope
    else Out of scope or recently seen
        Scope-->>KUR: Discard
    end

    Frontier->>Fetch: Next URL to fetch
    Fetch->>Fetch: Check robots.txt / quotas
    Fetch->>Fetch: HTTP GET
    Fetch->>Fetch: Extract outlinks
    Fetch->>Clamd: Stream response body for scan
    Clamd-->>Fetch: Clean / VIRUS FOUND

    Fetch->>WARCWriter: Write WARC record
    Fetch->>CDX: Record URL + digest + timestamp
    Fetch->>KafkaOut: Publish crawl log (uris.crawled)
    Fetch->>Scope: Submit outlinks (Candidate Chain)
```
