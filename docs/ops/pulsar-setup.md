# Pulsar Setup — Retention & Backlog Quotas

This document lists the operator commands that configure the Pulsar broker for Little Trader's `market-data.*` topics.

These commands must be run **once per environment** (local dev, staging, production) before any market-data backfill. Without them, the default broker policy will truncate backfilled messages after the default retention window (commonly a few days).

## Architectural context

See [ADR 0117 — Market data sourced from cached Pulsar topic consumption](../adr/0117-market-streams-sourced-from-pulsar-cache.md). The short version:

- fs-worker is the sole producer of market streams.
- Pulsar is the authoritative event log; topics are kept for ≥1 year locally (offloaded to GCS by [#107](https://github.com/4coders-com-br/little-trader/issues/107) once Tiered Storage lands).
- The server reads from Pulsar into an in-memory warm cache — no Datomic persistence for bars.

## Target namespace

The `market-data.*` topics live under the `public/default` namespace by default (see `resources/fast-streaming/topology.edn` — `:tenant "public"`, `:namespace "default"`).

If/when a dedicated namespace is adopted for market data, substitute it below.

## Infinite retention

```bash
pulsar-admin namespaces set-retention public/default \
  --size -1 --time -1
```

- `--time -1` — no time-based retention cap (never delete by age).
- `--size -1` — no size-based retention cap (never delete by total size).

Verify:

```bash
pulsar-admin namespaces get-retention public/default
# Expected: { "retentionTimeInMinutes": -1, "retentionSizeInMB": -1 }
```

## Disable backlog quota truncation

Prevents the broker from truncating unacked backlog when a consumer lags:

```bash
pulsar-admin namespaces set-backlog-quota public/default \
  --limit -1 --policy producer_exception
```

- `--limit -1` — no backlog size cap.
- `--policy producer_exception` — if a cap is ever reintroduced, fail producers with an exception rather than silently dropping messages.

Verify:

```bash
pulsar-admin namespaces get-backlog-quota public/default
```

## Applying to a dedicated namespace (future)

If/when `public/market-data` is created:

```bash
pulsar-admin namespaces create public/market-data
pulsar-admin namespaces set-retention     public/market-data --size -1 --time -1
pulsar-admin namespaces set-backlog-quota public/market-data --limit -1 --policy producer_exception
```

Update `resources/fast-streaming/topology.edn` `:defaults :namespace` to match.

## Tiered storage (future — #107)

Once GCS-backed Tiered Storage is enabled, the retention targets above remain but the broker will offload older segments transparently. No change is required in producers or consumers; the Pulsar Reader/Consumer APIs replay from offloaded segments without any client code change.

## Operational reminders

- Run these commands before any backfill — including before re-running a backfill in a fresh environment.
- Backfill is **not idempotent** at the Pulsar level. Re-running it produces duplicate messages per `:time`. The consumer-side warm cache (issue #119) folds duplicates on insert.
- Current `:retention` key in `topology.edn` (`:ttl-hours 720`) is **documentation only** — it is not applied by the Clojure code to the broker. Treat it as a historical hint; the broker commands above are the real source of truth.
