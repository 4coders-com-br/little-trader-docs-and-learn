# ADR 0117 — Market data sourced from cached Pulsar topic consumption

## Status

Proposed. Blocks [#110](https://github.com/4coders-com-br/little-trader/issues/110), [#119](https://github.com/4coders-com-br/little-trader/issues/119), [#118](https://github.com/4coders-com-br/little-trader/issues/118), [#107](https://github.com/4coders-com-br/little-trader/issues/107).

## Context

Little Trader ingests high-volume market streams from Deribit — OHLCV bars, book summaries, options chains, greeks — via the fs-worker node. These streams are append-only and time-ordered; the canonical event log lives in Apache Pulsar.

Until now the project carried an ambiguous store: bars were published to Pulsar **and** intended to be persisted to Datomic as the "durable" copy. This produced:

- Schema coupling between a fluid market stream shape and RAD attributes.
- Per-bar Datomic transaction overhead on hot ingest paths.
- Producer/consumer duplication between fs-worker (Pulsar writer) and the server (would-be Datomic writer).
- Brittle upsert logic on timestamp collisions and re-run semantics.

The concrete forcing question was issue [#110](https://github.com/4coders-com-br/little-trader/issues/110) — the 1H OHLCV backfill — which exposed the ambiguity: do we write 3 years of bars to Datomic, or keep them in Pulsar? Either answer implies an architecture.

## Decision

**Pulsar is the authoritative store for bars and all high-volume market streams.** Datomic is reserved for application state — users, accounts, trades, strategies, signals — and never holds market-stream data.

The server reads from Pulsar through an **in-memory warm cache** keyed by topic URI:

- On first access to a topic, the server scans it from earliest to end, builds a sorted in-memory vector, and spawns a daemon consumer that tails the topic for new messages.
- The cache has **no TTL**. Invalidation happens only through explicit `invalidate-topic!` ops.
- Duplicate messages (same `:time`) are folded on insert.

UI consumes bars via the `fs-ohlcv-history` EQL resolver or the `/api/fast-streaming/ohlcv` REST endpoint. Both paths read the warm cache; neither touches Datomic for bar data.

fs-worker is the **sole producer** of `market-data.*` topics. The server never publishes to `market-data.*`.

## Consequences

### Required operational changes

- Pulsar namespaces hosting `market-data.*` topics must be configured with infinite retention (`-1` / `-1`) and no backlog-quota truncation until [#107](https://github.com/4coders-com-br/little-trader/issues/107) lands GCS-backed Tiered Storage. Commands documented in [`docs/ops/pulsar-setup.md`](../ops/pulsar-setup.md).
- fs-worker is the only service that needs Pulsar producer credentials for market topics. The admin-panel command topic (`system.worker.commands`) remains server-produced — control-plane, not a market stream.
- Server pods need network reachability to the Pulsar broker (already covered by `lt-vpc-connector`).

### Capacity

Memory cost per cached topic: ~26,000 bars × ~160 B × N markets × M timeframes. For BTC/ETH × {1h, 1d} this is under 20 MB — acceptable for server JVMs. Revisit if markets or timeframes expand significantly; a future ticket can add per-topic cap / windowing if the JVM pressure becomes relevant.

### Developer workflow

- Schema evolution for bar shape happens at the serialization layer (JSON envelope in `topology.edn`), not in Datomic schema migrations.
- The consumer-side cache absorbs idempotency concerns: backfill re-runs that duplicate messages at the Pulsar level are folded by `:time` on insert.
- Datomic queries still own everything transactional (trades, accounts, strategies, signals); the split is clean and testable.

### Rollback path

If a future need forces a durable SQL/Datomic representation of bars (e.g., complex analytical queries that the in-memory cache can't serve), the remediation is to add a **downstream consumer** that projects Pulsar into that store — not to revert this ADR. The architectural commitment is that Pulsar is the source; any other representation is a projection.

## Rejected alternatives

### Datomic as bars store

Rejected. Reasons:

- Schema coupling between the market stream shape and RAD attributes.
- Per-bar transaction overhead on the hot backfill path (26,000+ transactions per market per 3-year 1H backfill).
- Producer/consumer duplication — fs-worker writes Pulsar, another process would write Datomic.
- Brittle upsert logic for timestamp-collision re-runs.
- Datomic index growth from high-volume append-only data is not what the store is optimized for.

### Pure on-demand Pulsar reads (no cache)

Rejected. Every UI fetch would re-scan the topic from earliest or maintain a cursor. A cold UI fetch for a 3-year 1H topic takes several seconds at the broker level — unacceptable in the UI hot path.

### Redis / external cache

Rejected. An extra dependency that doesn't solve any problem a JVM-local atom doesn't already solve. The server is a single stateful process per region; cross-process cache coherence isn't required for this data.

## Links

- [#110](https://github.com/4coders-com-br/little-trader/issues/110) — fs-node reshape + Pulsar retention (producer side)
- [#119](https://github.com/4coders-com-br/little-trader/issues/119) — consumer cache + UI (consumer side)
- [#107](https://github.com/4coders-com-br/little-trader/issues/107) — Tiered Storage offload to GCS
- [#118](https://github.com/4coders-com-br/little-trader/issues/118) — EMA overlay (downstream consumer of the same cached stream)
- [`docs/ops/pulsar-setup.md`](../ops/pulsar-setup.md) — operator commands for retention
