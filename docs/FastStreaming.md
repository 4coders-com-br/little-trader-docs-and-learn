# Fast Streaming v2 (fs-core)

> **Status:** Fully implemented and merged. `make dev` starts the complete stack locally.
> **Replaces:** Pulsar + fs-worker + fs-worker-api + SSH bridge (~4,300 lines deleted).

---

## What it is

fs-core is a single JVM process that replaces the old Pulsar-based fast-streaming stack.
It handles real-time Deribit market data ingest, in-process pub/sub via `core.async`,
Parquet persistence to MinIO/S3, and exposes a symmetric live/history subscription API.
An isolated `auth-node` sidecar handles all order submission over a Unix domain socket.

## Quick start

```bash
make dev        # Starts everything: postgres, transactor, minio, ring app,
                # shadow-cljs, AND fs-core. Zero cloud dependency.
```

Services started by `make dev`:

| Service            | Port(s)    | Notes |
|--------------------|------------|-------|
| PostgreSQL         | 5432       | Datomic SQL storage, data in `./postgres/` |
| Datomic transactor | 4334       | |
| MinIO              | 9100/9101  | `fs-core-data` bucket auto-created, data in `./minio/` |
| Ring app + nREPL   | 8080/7888  | |
| shadow-cljs        | 3000       | |
| **fs-core**        | **8091/9091** | Management API / Prometheus metrics |

Check fs-core health:

```bash
curl http://localhost:8091/health
# → {"status":"ok","uptime-ms":12345}

bb fs-core-status
# → shows uptime, active sources, gap count
```

---

## Architecture

```
  ┌──────────────────────────────────────────────────────────────────────┐
  │                        fs-core JVM process                           │
  │                                                                      │
  │  ┌──────────────────────────────────────────────────────────────┐   │
  │  │                     Source adapters                          │   │
  │  │  DeribitTicker (WS)  DeribitTrades (WS)  DeribitBook (WS)   │   │
  │  │  DeribitOptions (REST, polled hourly)                        │   │
  │  └──────────────────────────┬───────────────────────────────────┘   │
  │                             │ Envelope{ts,source,type,key,          │
  │                             │          payload,ingest-ts,seq}       │
  │  ┌──────────────────────────▼───────────────────────────────────┐   │
  │  │               fs-core Bus  (core.async)                      │   │
  │  │  dispatch-ch  sliding-buffer 65536                           │   │
  │  │  per-sub channels  sliding-buffer 4096                       │   │
  │  │  wildcard routing:  * = one segment   > = trailing           │   │
  │  │                                                              │   │
  │  │  market.deribit.<instrument>.ticker                          │   │
  │  │  market.deribit.<instrument>.trades                          │   │
  │  │  market.deribit.<instrument>.book-l2                         │   │
  │  │  option.deribit.<currency>.chain                             │   │
  │  │  system.gaps                                                 │   │
  │  └──┬──────────────────┬───────────────────┬─────────────────┘    │
  │     │                  │                   │                        │
  │  ┌──▼─────────┐  ┌─────▼──────────┐  ┌────▼─────────────────┐    │
  │  │ Gap Detector│ │ Storage Sink   │  │  Management + Metrics│    │
  │  │ monitors   │  │ market.> →     │  │  :8091  /health      │    │
  │  │ market.>   │  │ Parquet batch  │  │         /status      │    │
  │  │ publishes  │  │ → S3/MinIO     │  │         /gaps        │    │
  │  │ system.gaps│  │ 100k rows OR   │  │         /sources/:n  │    │
  │  └────────────┘  │ 128MB OR 1h    │  │             /restart │    │
  │                  └───────┬────────┘  │  :9091  /metrics     │    │
  │                          │           └──────────────────────┘    │
  └──────────────────────────│──────────────────────────────────────┘
                             │
          ┌──────────────────▼─────────────────┐
          │           MinIO / S3               │
          │  feed=ticker/exchange=deribit/      │
          │  symbol=btc-perpetual/              │
          │  year=2025/month=05/<uuid>.parquet  │
          └────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────┐
  │                       auth-node JVM process                          │
  │  Unix domain socket  /tmp/lt-deribit-auth.sock                       │
  │  submit-order · cancel-order · get-fills · get-positions             │
  │  Audit log (append-only EDN) · gap-state tagging                     │
  └──────────────────────────────────────────────────────────────────────┘
```

---

## Namespace map

```
src/com/little_trader/
  fs_core/
    main.clj              Entry point, wires everything, SIGTERM drain ≤ 10s
    schemas.clj           Malli registry: Envelope + 8 payload schemas
    bus.clj               core.async pub/sub, wildcard dispatch (* and >)
    source.clj            Source protocol: connect!, stream-or-poll!, source-close!
    adapters/
      deribit_ticker.clj  WS ticker subscription, topic: market.deribit.<instr>.ticker
      deribit_trades.clj  WS trade stream,         topic: market.deribit.<instr>.trades
      deribit_book.clj    WS L2 order book,        topic: market.deribit.<instr>.book-l2
      deribit_options.clj REST poll (1h),           topic: option.deribit.<ccy>.chain
    ingest/
      ring_buffer.clj     Per-feed sliding window ≥ 60s
      gap_detector.clj    Monitors market.>, emits :gap/:gap-recovered on system.gaps
    storage/
      parquet_writer.clj  Column buffer → tc/write! (100k rows | 128MB | 1h)
      s3.clj              cognitect-aws S3 client, Hive partition key builder
      sink.clj            Subscribes market.>, routes to per-topic writer atoms
    replay/
      scanner.clj         Downloads Parquet partitions → rows onto core.async channel
    api/
      subscription.clj    subscribe!(b :live opts) / subscribe!(b :history opts)
      management.clj      Ring+Reitit on :8091
    metrics/
      registry.clj        Prometheus counters/gauges/histograms
      http_server.clj     /metrics on :9091
  auth_node/
    main.clj              Entry point, loads config, starts Unix socket server
    transport.clj         JDK-21 UnixDomainSocketAddress, length-prefixed JSON
    deribit_api.clj       submit-order, cancel-order, get-fills, get-positions
    audit_log.clj         fsync-per-entry append-only EDN log
    gap_state.clj         Tags N audit entries with :post-gap after gap-recovered
```

---

## Message envelope

Every message on every topic has this shape (validated by `fs-core.schemas/Envelope`):

```clojure
{:ts         1234567890000000000   ; exchange epoch nanos (or ingest-ts)
 :source     :deribit
 :type       :ticker               ; see payload schemas below
 :key        "btc-perpetual"       ; instrument or currency
 :payload    { ... }               ; type-specific map
 :ingest-ts  1234567890001000000   ; wall-clock nanos at fs-core receive
 :seq        42}                   ; monotonic per source, resets on restart
```

**Payload schemas** (`:type` → schema):

| `:type`            | Key fields |
|--------------------|-----------|
| `:ticker`          | `instrument-name`, `last-price`, `mark-price`, `bid-price`, `ask-price`, `timestamp` |
| `:trade`           | `trade-id`, `instrument-name`, `price`, `amount`, `direction`, `timestamp` |
| `:book-l2-snapshot`| `instrument-name`, `bids [[price amount]]`, `asks [[price amount]]` |
| `:book-l2-delta`   | `instrument-name`, `bids [["new"/"change"/"delete" price amount]]`, `asks [...]` |
| `:option-chain`    | `currency`, `items [...]`, `timestamp` |
| `:ohlcv-1h`        | `instrument-name`, `open`, `high`, `low`, `close`, `volume`, `timestamp` |
| `:gap`             | `topic`, `since-ms` |
| `:gap-recovered`   | `topic`, `gap-ms` |

---

## Topic taxonomy

```
market.deribit.btc-perpetual.ticker
market.deribit.btc-perpetual.trades
market.deribit.btc-perpetual.book-l2
market.deribit.eth-perpetual.ticker
...
option.deribit.btc.chain
option.deribit.eth.chain
system.gaps
```

Wildcard subscription examples:

```clojure
(bus/subscribe! b "market.>"         handler)   ; all market messages
(bus/subscribe! b "market.*.btc-perpetual.>" h) ; BTC perp, any feed
(bus/subscribe! b "system.gaps"      handler)   ; gap/recovered events only
```

---

## Subscription API

**Live** (taps the in-process bus — no disk):

```clojure
(require '[com.little-trader.fs-core.api.subscription :as sub])

(def ch (sub/subscribe! bus :live {:pattern "market.deribit.btc-perpetual.ticker"}))
(async/go-loop []
  (when-let [msg (async/<! ch)]
    (println (:payload msg))
    (recur)))
```

**Historical replay** (downloads Parquet from S3, same envelope format):

```clojure
(def ch (sub/subscribe! bus :history
          {:s3-client s3-client
           :bucket    "fs-core-data"
           :feed      "ticker"
           :exchange  "deribit"
           :symbol    "btc-perpetual"
           :year      2025 :month 5}))
```

The subscriber code is identical for both modes.

---

## Storage: Parquet on S3/MinIO

### Dependencies

```clojure
;; deps.edn
scicloj/tablecloth {:mvn/version "7.029.2"}
org.scicloj/dataset-io {:mvn/version "1.0.0"}   ; brings parquet-hadoop + Arrow JARs
```

`parquet_writer.clj` and `scanner.clj` both `require` `tech.v3.libs.parquet` at namespace
load time, which self-registers the Parquet format handler in `tech.v3.dataset.io`.

### Partition layout (Hive-compatible)

```
fs-core-data/
  feed=ticker/
    exchange=deribit/
      symbol=btc-perpetual/
        year=2025/
          month=05/
            <uuid>.parquet
```

Queryable by DuckDB, Spark, Polars, Athena, BigQuery external tables — without custom code.

### Batch triggers (whichever fires first)

| Trigger        | Default   | Config key       |
|---------------|-----------|-----------------|
| Row count     | 100,000   | `:batch-rows`   |
| Estimated size| 128 MB    | `:batch-bytes`  |
| Time open     | 1 hour    | `:batch-ms`     |

### Query with DuckDB

```sql
-- DuckDB can query MinIO directly
INSTALL httpfs; LOAD httpfs;
SET s3_endpoint='localhost:9100';
SET s3_url_style='path';
SET s3_access_key_id='littletrader';
SET s3_secret_access_key='littletrader123';
SET s3_use_ssl=false;

SELECT ts, json_extract(payload, '$.last_price') AS price
FROM read_parquet('s3://fs-core-data/feed=ticker/exchange=deribit/**/*.parquet')
ORDER BY ts DESC LIMIT 100;
```

---

## Configuration (`resources/fs-core.edn`)

```clojure
{:exchange  :deribit
 :testnet?  true
 :instruments ["BTC-PERPETUAL" "ETH-PERPETUAL"]
 :sources   [:ticker :trades :book-l2 :option-chain]

 :bus       {:dispatch-buffer 65536 :subscription-buffer 4096}

 :storage   {:enabled?    true
              :bucket      "fs-core-data"
              :prefix      ""
              :batch-rows  100000
              :batch-bytes 134217728
              :batch-ms    3600000
              :local-tmp-dir "/tmp/fs-core"}

 :s3        {:endpoint   "http://minio:9000"   ; MinIO in dev, real S3 endpoint in prod
              :region     "us-east-1"
              :access-key "littletrader"
              :secret-key "littletrader123"}

 :api       {:port 8091}
 :metrics   {:port 9091}

 :auth-node {:socket-path  "/tmp/lt-deribit-auth.sock"
              :audit-log    "/tmp/lt-audit.edn"}

 :gap       {:threshold-ms  30000
              :post-gap-tag-count 10}}
```

Override config file via env: `FS_CORE_CONFIG=my-config.edn`.

---

## Management API (`:8091`)

| Method | Path                    | Description |
|--------|-------------------------|-------------|
| GET    | `/health`               | `{"status":"ok","uptime-ms":N}` |
| GET    | `/status`               | Uptime + bus stats (topics, messages, dropped, subscriptions) |
| GET    | `/gaps`                 | Current gap set |
| POST   | `/sources/:name/restart`| Restart a source: `ticker`, `trades`, `book-l2`, `option-chain` |
| POST   | `/instruments`          | Add instrument at runtime |

The Ring app proxies `/api/admin/workers/status` → `GET /status` + `GET /gaps`,
and `/api/admin/workers` POST → `/sources/:name/restart`, surfaced in the admin panel.

---

## Prometheus metrics (`:9091/metrics`)

| Metric                          | Type      | Labels |
|---------------------------------|-----------|--------|
| `fs_ingest_messages_total`      | Counter   | feed, exchange, symbol |
| `fs_bus_dropped_messages_total` | Counter   | pattern |
| `fs_ingest_lag_ms`              | Gauge     | feed |
| `fs_gap_state`                  | Gauge     | feed (1=gap, 0=ok) |
| `fs_ring_buffer_size`           | Gauge     | feed |
| `fs_storage_write_latency_ms`   | Histogram | — |
| `fs_auth_order_ack_latency_ms`  | Histogram | — |

---

## Auth node

Runs as a separate JVM process adjacent to fs-core. Exchange API keys never leave it.

```
executor → Unix socket /tmp/lt-deribit-auth.sock → auth-node → Deribit REST
```

**Wire protocol:** length-prefixed JSON (4-byte big-endian length + UTF-8 body).

**Actions:** `submit-order`, `cancel-order`, `get-positions`, `gap-notify`, `health`.

**Gap tagging:** after a gap-recovered event, the next N audit entries (default 10, configurable
via `:gap.post-gap-tag-count`) are tagged `{:post-gap true}` for auditability.

**Audit log:** `/tmp/lt-audit.edn` — append-only, fsync per entry.

Start auth-node standalone:

```bash
clojure -M:run-auth-node
```

---

## Gap detection

The gap detector subscribes to `market.>` and tracks `last-seen` per topic.
A periodic go-loop (every `threshold-ms / 6`, min 5s) checks for silence:

- Topic silent > `threshold-ms` (default 30s) → publish `:gap` on `system.gaps`
- Next message arrives → publish `:gap-recovered`

Gap detection is **informational** — order submission is not suspended.
The auth node tags post-gap audit entries (AC7.8) so activity during a gap is auditable.

---

## Dockerfile and build

```bash
# Build fs-core uberjar (AOT-compiles fs-core.main only, not the Ring app)
make build-fs-core
# → clojure -T:build uber-fs-core → target/little-trader.jar

# Or let docker build do it
docker build -f Dockerfile.fs-core -t fs-core .
```

The `Dockerfile.fs-core` uses a two-stage Temurin-21 build. The uberjar target
is `uber-fs-core` in `build.clj` (not `uber`, which compiles the full Ring app).

---

## Tests

```bash
bb test              # runs all suites including :fs-core
clojure -M:test --focus :fs-core
```

Zero-I/O tests in `test/com/little_trader/fs_core/`:
- `schemas_test.clj` — malli validation for all 8 payload types
- `bus_test.clj` — wildcard dispatch, drop-on-full, unsubscribe

---

## What was retired

The old Pulsar-based stack is fully gone:

| Component | Replacement |
|-----------|-------------|
| Apache Pulsar (broker) | core.async in-process bus |
| fs-worker (Pulsar consumer) | fs-core source adapters |
| fs-worker-api (REST) | management.clj on :8091 |
| Blob Account / fs-worker-api storage | Parquet → MinIO/S3 directly |
| SSH VPN bridge | Unix domain socket (same host) |
| Datomic materialized views of OHLCV | Deribit REST historical endpoint |

All Pulsar namespaces were deleted or stubbed. `pulsar-client` dep removed.
Cloud infra (Cloud Run, Cloud SQL, VMs, GCS buckets, VPC connector) decommissioned.
Full inventory: `todo/FUTURE_CLOUD_INFRA.md`.

---

## Roadmap (deferred)

- **Alert channel** for gap-recovery warning (Telegram, email, CLI). Interface exists, concrete channel TBD.
- **Binance adapter** (v2.1) — protocol is designed for additive adapters
- **MT5 auth-node** (v2.2) — likely Windows host + mTLS bridge
- **Apache Iceberg / Delta Lake** — can be layered on existing Parquet files without rewriting
- **Cloud staging** — fresh GCP or R2 re-provisioning once local stack is solid (see `todo/FUTURE_CLOUD_INFRA.md`)
