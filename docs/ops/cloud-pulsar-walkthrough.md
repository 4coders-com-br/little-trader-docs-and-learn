# Walkthrough — Local Dev Container → Cloud Pulsar

End-to-end sequence to take a workstation from cold start to a local dev container serving OHLCV data from a **cloud Pulsar broker**.

Rooted in [ADR 0117](../adr/0117-market-streams-sourced-from-pulsar-cache.md) and ticket [#110](https://github.com/4coders-com-br/little-trader/issues/110).

## Preconditions

- `gcloud` CLI authenticated (`gcloud auth login`, correct project selected)
- `docker` + `docker compose` running locally
- The `fs-worker-staging` VM (or whatever `LT_INFRA_VM` points at) is up and reachable via IAP / SSH
- `docker-compose.yml`, `scripts/pulsar-configure-retention.sh`, and the Makefile targets from this change have landed

## Mental model

```
┌───── your workstation ─────┐      ┌─────── cloud VM ────────┐
│                            │      │                         │
│  docker-compose server ────┼──┐   │   fs-worker (producer)  │
│                            │  │   │        │                │
│  APP_PULSAR_SERVICE_URL =  │  │   │        ▼                │
│    pulsar://host.docker    │  │   │   Pulsar broker         │
│      .internal:6650        │  │   │     :6650  :8080        │
│                            │  │   │                         │
└────────────────────────────┘  │   └────────────┬────────────┘
                                │                │
                   SSH tunnel (L 6650, L 8081)   │
                                │                │
                                └────────────────┘
```

The server container reaches `pulsar://host.docker.internal:6650`, which maps through the SSH tunnel to the broker's port `6650` inside the VM.

## Step-by-step

### 1. Open the tunnel

```bash
make pulsar-tunnel-up
# or: ./lt pulsar-tunnel up
```

Forwards `localhost:6650` → broker binary protocol, `localhost:8081` → broker admin HTTP. PID stored in `.pulsar-tunnel.pid`.

Verify:

```bash
curl -fsS http://localhost:8081/admin/v2/clusters   # should print ["standalone"]
```

### 2. Apply retention (once per environment)

```bash
make pulsar-configure-cloud
# or: ./lt pulsar-configure cloud
```

Runs the shared [`scripts/pulsar-configure-retention.sh`](../../scripts/pulsar-configure-retention.sh) against the tunneled admin endpoint. Safe to re-run; idempotent.

Verify manually if you want:

```bash
docker run --rm --network host apachepulsar/pulsar:2.11.4 \
  bin/pulsar-admin --admin-url http://localhost:8081 \
  namespaces get-retention public/default
# → retentionTimeInMinutes: -1, retentionSizeInMB: -1
```

### 3. (One-time) Trigger backfill on fs-worker

Only needed if the cloud topics are empty or you're re-seeding. Skip if the worker has been polling live for a while and already has coverage.

SSH into the VM and run the REPL-style backfill. Option A — worker REST API:

```bash
gcloud compute ssh fs-worker-staging --zone=us-central1-a \
  --command 'curl -sX POST http://localhost:8090/backfill \
               -H "Content-Type: application/json" \
               -d "{\"markets\":[\"btc\",\"eth\"],\"days\":1095,\"resolution\":\"60\"}"'
```

Option B — nREPL into the worker container and call `bf/backfill-market-matrix!` directly.

This takes several minutes per market. Check progress:

```bash
docker run --rm --network host apachepulsar/pulsar:2.11.4 \
  bin/pulsar-admin --admin-url http://localhost:8081 \
  topics stats persistent://public/default/market-data.btc.ohlcv.1h \
  | grep -E 'msgInCounter|storageSize'
```

Target: `msgInCounter` ≥ 26,000 per market for a 3-year 1H backfill.

### 4. Start local dev container pointed at cloud Pulsar

```bash
make dev-cloud-pulsar
```

This composite target:
1. Re-checks the tunnel (opens it if stopped).
2. Re-applies retention (no-op if already in place).
3. Starts the local `server` + `shadow-watch` with `APP_PULSAR_SERVICE_URL=pulsar://host.docker.internal:6650`.

Containers up:

```
server         :8080   (Ring/Reitit + nREPL :7888)
shadow-watch   :3000   (cljs dev)
```

### 5. Verify bars return to the UI

```bash
curl -s 'http://localhost:8080/api/fast-streaming/ohlcv?market=btc&timeframe=1h&limit=5' \
  | jq '{count: .count, first: .bars[0].time, last: .bars[-1].time}'
```

Expected:

```json
{ "count": 5, "first": 1711..., "last": 1711... }
```

Or hit the UI: open `http://localhost:8080`, log in, navigate to the workspace — candlesticks should render. First load of a topic pays the Pulsar scan cost (a few seconds); subsequent loads are instant from the warm cache.

### 6. Tear down

```bash
make pulsar-tunnel-down   # close SSH tunnel
make stop                 # stop local containers
```

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `pulsar-configure-cloud` fails with "No admin endpoint" | Tunnel down | `make pulsar-tunnel-up` first |
| Tunnel opens then drops after a few minutes | Idle SSH timeout | add `ServerAliveInterval 30` to `~/.ssh/config` for the VM host |
| UI chart blank, REST returns 0 bars | Topic empty | Run step 3 backfill, or check `fs-worker` status via `gcloud compute ssh fs-worker-staging --command 'curl -s localhost:8090/status'` |
| `host.docker.internal` unresolvable in container | Linux host (feature is macOS/Windows only) | Add `extra_hosts: ["host.docker.internal:host-gateway"]` to the server service, or use `network_mode: host` |
| Retention verification shows `60` | Broker didn't accept `-1` | Check Pulsar version; `-1` requires Pulsar 2.8+. Fall back to `--size 9999999 --time 525600` (1 year in minutes) as a stopgap |
| Slow first chart load | Pulsar scans 3 years of messages | Expected for ~26k bars; subsequent reads are from the warm cache |

## Automation surface — what's where

| Piece | File |
|---|---|
| Retention script (shared) | [`scripts/pulsar-configure-retention.sh`](../../scripts/pulsar-configure-retention.sh) |
| Local Pulsar init-time retention | `pulsar-retention-init` service in [`docker-compose.yml`](../../docker-compose.yml) |
| Cloud VM Pulsar init-time retention | `pulsar-retention-init` service in [`docker-compose.worker.yml`](../../docker-compose.worker.yml) |
| Makefile targets | `pulsar-configure-{local,cloud}`, `pulsar-tunnel-{up,down}`, `pulsar-stats`, `dev-cloud-pulsar` in [`Makefile`](../../Makefile) |
| CLI subcommands | `./lt pulsar-configure [local\|cloud]`, `./lt pulsar-tunnel [up\|down\|status]` in [`scripts/lt`](../../scripts/lt) |
| Operator reference | [`docs/ops/pulsar-setup.md`](./pulsar-setup.md) |

## Related tickets

- [#110](https://github.com/4coders-com-br/little-trader/issues/110) — producer + retention (this walkthrough)
- [#119](https://github.com/4coders-com-br/little-trader/issues/119) — server warm cache + UI (completes the pipeline)
- [#107](https://github.com/4coders-com-br/little-trader/issues/107) — Tiered Storage offload
- [#117](https://github.com/4coders-com-br/little-trader/issues/117) — ADR
