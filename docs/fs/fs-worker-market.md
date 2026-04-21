# fs-worker Market Stack — Operator Manual

End-to-end, terminal-only procedure to go from a clean slate to a **local dev app pulling 3-year daily OHLCV from cloud Pulsar**, with REPL instructions for manual verification.

Scope: CI/CD reset, GCP provisioning, fs-worker deploy, Pulsar retention, 3-year backfill, tunnel, local dev wire-up, REPL probes.

Authoritative references:
- [ADR 0117 — Pulsar as source of truth](../adr/0117-market-streams-sourced-from-pulsar-cache.md)
- [docs/ops/pulsar-setup.md](../ops/pulsar-setup.md)
- [docs/ops/cloud-pulsar-walkthrough.md](../ops/cloud-pulsar-walkthrough.md) (quick-reference version of the local-dev half)
- Tickets [#110](https://github.com/4coders-com-br/little-trader/issues/110), [#117](https://github.com/4coders-com-br/little-trader/issues/117), [#119](https://github.com/4coders-com-br/little-trader/issues/119), [#107](https://github.com/4coders-com-br/little-trader/issues/107)

---

## 0. Prerequisites & environment

Before touching anything, verify your shell:

```bash
# 0.1 — tools
gcloud --version                          # Google Cloud SDK 4xx+
docker --version && docker compose version
gh --version                              # GitHub CLI, authenticated
clojure --version                         # Clojure CLI (local REPL work)

# 0.2 — GCP auth
gcloud auth list
gcloud config get-value project            # should be little-trader (or your project)
gcloud config get-value compute/zone       # us-central1-a
gcloud auth application-default login      # for Terraform/admin SDK calls if needed

# 0.3 — repo
cd ~/4coders/little-trader
git status                                 # clean; on main
git fetch origin && git reset --hard origin/main   # *** destructive — only on a dedicated workstation ***
```

Export the variables this manual uses everywhere:

```bash
export GCP_PROJECT_ID=little-trader
export GCP_PROJECT_NUMBER=2632317954
export GCP_REGION=us-central1
export GCP_ZONE=us-central1-a
export GITHUB_REPO=4coders-com-br/little-trader
export LT_INFRA_VM=datomic-transactor       # Datomic + Postgres VM
export LT_FS_VM=fs-worker-staging           # Pulsar + fs-worker VM
```

**Before any destructive action**, save current state:

```bash
mkdir -p /tmp/lt-snapshot && cd /tmp/lt-snapshot
gcloud compute instances list        > vms.txt
gcloud run services list             > cloudrun.txt
gcloud compute firewall-rules list   > firewalls.txt
gcloud artifacts repositories list   > gar.txt
gh secret list --repo $GITHUB_REPO   > secrets.txt
cd -
```

---

## 1. NUKE — tear the current environment down to zero

⚠️ **Everything below deletes production-like resources.** Skip this section if you only want to bring up a missing piece; jump to §2.

### 1.1 — Cloud Run services

```bash
gcloud run services delete little-trader-staging --region=$GCP_REGION --quiet 2>/dev/null || true
gcloud run services delete little-trader        --region=$GCP_REGION --quiet 2>/dev/null || true
```

### 1.2 — Compute Engine VMs (fs-worker + datomic-transactor)

```bash
gcloud compute instances delete $LT_FS_VM    --zone=$GCP_ZONE --quiet 2>/dev/null || true
gcloud compute instances delete $LT_INFRA_VM --zone=$GCP_ZONE --quiet 2>/dev/null || true
```

### 1.3 — Persistent disks and static IPs

```bash
gcloud compute addresses list --filter="name~'(fs-worker|datomic)'" \
  --format="value(name,region)" | while read name region; do
    gcloud compute addresses delete "$name" --region="$region" --quiet 2>/dev/null || true
done

gcloud compute disks list --filter="zone:$GCP_ZONE AND name~'(fs-worker|datomic)'" \
  --format="value(name,zone)" | while read name zone; do
    gcloud compute disks delete "$name" --zone="$zone" --quiet 2>/dev/null || true
done
```

### 1.4 — VPC connector + firewall rules + subnet

```bash
gcloud compute networks vpc-access connectors delete lt-vpc-connector \
  --region=$GCP_REGION --quiet 2>/dev/null || true

gcloud compute firewall-rules delete allow-cloudrun-to-datomic --quiet 2>/dev/null || true

gcloud compute networks subnets delete cloud-run-subnet \
  --region=$GCP_REGION --quiet 2>/dev/null || true
```

### 1.5 — Artifact Registry images

```bash
# Delete ALL images in the repo — rebuild on next CI run.
gcloud artifacts docker images list \
  $GCP_REGION-docker.pkg.dev/$GCP_PROJECT_ID/little-trader \
  --format="value(package)" | while read img; do
    gcloud artifacts docker images delete "$img" --delete-tags --quiet 2>/dev/null || true
done
```

*(Optional, rarely needed)* delete the GAR **repository** itself:

```bash
gcloud artifacts repositories delete little-trader --location=$GCP_REGION --quiet
```

### 1.6 — Workload Identity and service account

Only if you're truly starting from zero on this GCP project. Deleting these breaks CI until §2 re-runs.

```bash
gcloud iam workload-identity-pools providers delete github-provider \
  --location=global --workload-identity-pool=github-actions-pool --quiet 2>/dev/null || true

gcloud iam workload-identity-pools delete github-actions-pool \
  --location=global --quiet 2>/dev/null || true

gcloud iam service-accounts delete \
  github-actions-deployer@$GCP_PROJECT_ID.iam.gserviceaccount.com --quiet 2>/dev/null || true
```

### 1.7 — GitHub secrets (optional; only if changing project)

```bash
for s in GCP_PROJECT_ID GCP_WORKLOAD_IDENTITY_PROVIDER GCP_SA_EMAIL \
         DATOMIC_URI_STAGING PULSAR_SERVICE_URL_STAGING \
         FS_WORKER_URL_STAGING JWT_SECRET ADMIN_EMAIL ADMIN_PASSWORD \
         BLOB_ENDPOINT BLOB_BUCKET BLOB_ACCESS_KEY BLOB_SECRET_KEY BLOB_REGION; do
  gh secret delete "$s" --repo $GITHUB_REPO 2>/dev/null || true
done
```

### 1.8 — Verify clean

```bash
gcloud run services list --region=$GCP_REGION       # expected: no services
gcloud compute instances list                        # expected: none named {datomic,fs-worker}
gcloud compute firewall-rules list | grep datomic    # expected: empty
```

---

## 2. CI/CD baseline — GCP project bootstrap

### 2.1 — Enable APIs + create GAR repo + service account + Workload Identity

```bash
./cloud-run/setup.sh
```

What this does (read the script for the verbatim sequence):

- Enables APIs: `run`, `artifactregistry`, `cloudbuild`, `iam`, `compute`, `vpcaccess`.
- Creates Artifact Registry repo `little-trader` in `$GCP_REGION`.
- Creates service account `github-actions-deployer@$GCP_PROJECT_ID.iam.gserviceaccount.com`.
- Grants roles: `run.admin`, `artifactregistry.writer`, `iam.serviceAccountUser`, `compute.instanceAdmin.v1`.
- Creates Workload Identity pool `github-actions-pool` and provider `github-provider` bound to `$GITHUB_REPO`.

At the end it prints a block of `gh secret set` commands — save them to a file:

```bash
./cloud-run/setup.sh | tee /tmp/lt-setup-out.log
```

### 2.2 — Commit GitHub secrets

```bash
gh secret set GCP_PROJECT_ID            --body "$GCP_PROJECT_ID"
gh secret set GCP_REGION                --body "$GCP_REGION"
gh secret set GCP_ZONE                  --body "$GCP_ZONE"
gh secret set GCP_WORKLOAD_IDENTITY_PROVIDER --body "$(grep WORKLOAD_IDENTITY /tmp/lt-setup-out.log | awk -F= '{print $2}')"
gh secret set GCP_SA_EMAIL              --body "github-actions-deployer@$GCP_PROJECT_ID.iam.gserviceaccount.com"
gh secret set JWT_SECRET                --body "$(openssl rand -base64 32)"
gh secret set ADMIN_EMAIL               --body "admin@littletrader.dev"
gh secret set ADMIN_PASSWORD            --body "$(openssl rand -base64 24)"
```

---

## 3. Infrastructure provisioning — VPC + VMs

### 3.1 — Datomic/Postgres VM

```bash
export POSTGRES_PASSWORD="$(openssl rand -hex 24)"
./cloud-run/setup-infra.sh
```

What this does:

- Creates subnet `cloud-run-subnet` (10.10.0.0/26) in the default network.
- Creates firewall rule `allow-cloudrun-to-datomic` (TCP 4334, 5432, 6650, 8090 from the subnet, tagged `datomic-transactor` + `fs-worker`).
- Creates the `datomic-transactor` VM (e2-micro, ubuntu-2204-lts, 30 GB).
- SCPs `Dockerfile.transactor`, `docker-compose.infra.yml`, `config/transactor.properties`, `scripts/init-pg-schema.sh`.
- Runs `docker compose up -d` on the VM: Postgres + Datomic transactor.

Capture the VM's internal IP:

```bash
export LT_INFRA_IP=$(gcloud compute instances describe $LT_INFRA_VM \
  --zone=$GCP_ZONE --format='value(networkInterfaces[0].networkIP)')
echo "datomic-transactor IP: $LT_INFRA_IP"

gh secret set POSTGRES_PASSWORD --body "$POSTGRES_PASSWORD"
gh secret set DATOMIC_URI_STAGING --body \
  "datomic:sql://little-trader?jdbc:postgresql://$LT_INFRA_IP:5432/datomic?user=datomic&password=$POSTGRES_PASSWORD"
```

### 3.2 — fs-worker VM (Pulsar + worker)

```bash
./cloud-run/setup-worker.sh
```

What this does:

- Creates the `fs-worker-staging` VM (e2-small, ubuntu-2204-lts, 40 GB).
- Installs Docker.
- SCPs `Dockerfile.fast-streaming`, `docker-compose.worker.yml`, `.env`, `deps.edn`, `build.clj`, and recursively `src/` + `resources/` + `scripts/`.
- Runs `docker compose -f docker-compose.worker.yml up -d --build`, which brings up:
  - `pulsar` (standalone, :6650 broker, :8081 admin HTTP)
  - `pulsar-init` (creates all `market-data.*` topics)
  - **`pulsar-retention-init`** (applies infinite retention — ADR #117)
  - `fs-worker` (port 8090)

Capture the IP + commit to secrets:

```bash
export LT_FS_IP=$(gcloud compute instances describe $LT_FS_VM \
  --zone=$GCP_ZONE --format='value(networkInterfaces[0].networkIP)')
echo "fs-worker-staging IP: $LT_FS_IP"

gh secret set PULSAR_SERVICE_URL_STAGING --body "pulsar://$LT_FS_IP:6650"
gh secret set FS_WORKER_URL_STAGING      --body "http://$LT_FS_IP:8090"
```

### 3.3 — VPC connector for Cloud Run

```bash
gcloud compute networks vpc-access connectors create lt-vpc-connector \
  --region=$GCP_REGION \
  --subnet=cloud-run-subnet \
  --min-instances=2 --max-instances=3 \
  --machine-type=e2-micro
```

Verify connectivity from an SSH session on one VM to the other (IP × port × expected response):

```bash
gcloud compute ssh $LT_FS_VM --zone=$GCP_ZONE --command "nc -zv $LT_INFRA_IP 5432 && nc -zv 127.0.0.1 6650"
# Expected: both 'succeeded!'
```

### 3.4 — Blob storage (MinIO / GCS) secrets

Use the existing GCS bucket `lt-fast-streaming-staging` if it already exists, otherwise create it:

```bash
gsutil ls -b gs://lt-fast-streaming-staging || \
  gsutil mb -p $GCP_PROJECT_ID -l $GCP_REGION gs://lt-fast-streaming-staging

gh secret set BLOB_ENDPOINT   --body "https://storage.googleapis.com"
gh secret set BLOB_BUCKET     --body "lt-fast-streaming-staging"
gh secret set BLOB_ACCESS_KEY --body "<HMAC key>"     # gsutil hmac create for the SA
gh secret set BLOB_SECRET_KEY --body "<HMAC secret>"
gh secret set BLOB_REGION     --body "us-east-1"
```

---

## 4. First deploy via CI

Push (or re-push) `main` to trigger the pipeline, or run it manually:

```bash
gh workflow run main.yml --ref main
gh run watch          # tail the latest run in real time
```

The pipeline:

1. **test** (`clojure -M:test`) + **lint** (`clj-kondo`) + **cljs-test**.
2. **build-and-push** — builds `little-trader:{SHA}` and `fs-worker:{SHA}`, pushes to GAR.
3. **deploy-staging** — `gcloud run deploy little-trader-staging` with `--vpc-connector=lt-vpc-connector` and `PULSAR_SERVICE_URL=$PULSAR_SERVICE_URL_STAGING`.
4. **update-fs-worker** — SCPs the new `docker-compose.worker.yml`, SSH's in, `docker compose pull && up -d --force-recreate fs-worker`.

After it completes, hit health:

```bash
gcloud compute ssh $LT_FS_VM --zone=$GCP_ZONE --command "curl -s http://127.0.0.1:8090/health"
# expected: {"status":"ok","worker-status":"running","timestamp":...}

gcloud compute ssh $LT_FS_VM --zone=$GCP_ZONE --command "curl -s http://127.0.0.1:8090/status | jq"
# expected: .status == "running", .topics > 0
```

---

## 5. Pulsar readiness on the VM

`pulsar-retention-init` already ran as part of `docker compose up`. Verify:

```bash
gcloud compute ssh $LT_FS_VM --zone=$GCP_ZONE --command "
  cd ~/fs-worker &&
  sudo docker compose -f docker-compose.worker.yml exec -T pulsar \
    bin/pulsar-admin --admin-url http://pulsar:8080 \
    namespaces get-retention public/default
"
# expected: retentionTimeInMinutes: -1, retentionSizeInMB: -1
```

If retention is wrong (upgrade from an older compose file), re-apply manually from your workstation after opening the tunnel (§6):

```bash
make pulsar-tunnel-up
make pulsar-configure-cloud
```

---

## 6. Open the SSH tunnel to cloud Pulsar

Run from your workstation:

```bash
make pulsar-tunnel-up
# equivalent: ./lt pulsar-tunnel up
```

This forwards:

- `localhost:6650` → Pulsar broker binary protocol
- `localhost:8081` → Pulsar admin HTTP

Smoke test:

```bash
curl -fsS http://localhost:8081/admin/v2/clusters
# expected: ["standalone"]
```

---

## 7. Trigger the 3-year backfill

The worker reads `FS_BACKFILL_DAYS` (default **1095**) and `FS_BACKFILL_RESOLUTION` (default **60**) from env at call time. Triggering the backfill is a plain HTTP POST with no body:

```bash
# Via the tunnel (add :8090 to the tunnel first if you want direct local access).
# Simplest: invoke it on the VM itself.
gcloud compute ssh $LT_FS_VM --zone=$GCP_ZONE --command "curl -sX POST http://127.0.0.1:8090/backfill"
# expected: {"message":"Backfill triggered","markets":[...],"days":1095,"resolution":"60"}
```

The worker enters `status=:backfilling`. Deribit caps each chunk at ~5000 bars, so a 3-year 1H backfill per market is roughly:

- 3 y × 8,760 h ≈ 26,300 bars / ~5 chunks × 200 ms sleep ≈ **2–3 minutes per market**
- For all 7 configured price markets: **15–25 minutes total**

Tail progress:

```bash
gcloud compute ssh $LT_FS_VM --zone=$GCP_ZONE --command \
  "cd ~/fs-worker && sudo docker compose logs -f fs-worker --tail=100"
```

Check topic fill from your workstation (via the tunnel):

```bash
docker run --rm --network host apachepulsar/pulsar:2.11.4 \
  bin/pulsar-admin --admin-url http://localhost:8081 \
  topics stats persistent://public/default/market-data.btc.ohlcv.1h \
  | grep -E 'msgInCounter|storageSize'
# expected: msgInCounter >= 26000, storageSize > 0
```

Poll status:

```bash
gcloud compute ssh $LT_FS_VM --zone=$GCP_ZONE --command "curl -s http://127.0.0.1:8090/status | jq '.status, .last-backfill'"
# expected eventually: "running", {"markets":[...],"total":...,"elapsed-ms":...}
```

### 7.1 — Daily aggregator

The `ohlcv_agg` module inside the worker subscribes to the 1H topic and publishes to the 1D topic. It publishes a day's bar when the **next day's** first 1H bar arrives. For a 3-year backfill, ~1,095 daily bars land in a minute or two after the 1H backfill finishes:

```bash
docker run --rm --network host apachepulsar/pulsar:2.11.4 \
  bin/pulsar-admin --admin-url http://localhost:8081 \
  topics stats persistent://public/default/market-data.btc.ohlcv.1d \
  | grep msgInCounter
# expected: ≥1000
```

---

## 8. Local dev app wired to cloud Pulsar

From the same workstation (tunnel still up from §6):

```bash
make dev-cloud-pulsar
```

This composite target:

1. Re-checks / re-opens the tunnel.
2. Re-applies retention against the cloud broker (no-op on re-run).
3. Starts **only** the local `server` + `shadow-watch` containers with `APP_PULSAR_SERVICE_URL=pulsar://host.docker.internal:6650`.

Services up on the workstation:

| Endpoint | Purpose |
|---|---|
| `localhost:7888` | nREPL |
| `localhost:3000` | shadow-cljs UI (cljs hot reload) |
| `localhost:8080` | Ring HTTP server |
| `localhost:6650` (tunnel) | Cloud Pulsar broker |
| `localhost:8081` (tunnel) | Cloud Pulsar admin |

### 8.1 — Verify bars flow

```bash
# REST: last 5 BTC 1H bars
curl -s 'http://localhost:8080/api/fast-streaming/ohlcv?market=btc&timeframe=1h&limit=5' | jq '{count, first:.bars[0].time, last:.bars[-1].time}'

# REST: last 30 BTC daily bars
curl -s 'http://localhost:8080/api/fast-streaming/ohlcv?market=btc&timeframe=1d&limit=30' | jq '.count'

# Open the workspace — BTC 1H candles with 3y of history
open http://localhost:8080          # macOS
xdg-open http://localhost:8080      # linux
```

First fetch per topic triggers the server's warm-cache scan (a few seconds for 26k 1H bars). Subsequent fetches are instant.

---

## 9. Manual REPL verification

Connect from your editor (Emacs CIDER / Cursive / Calva) to `nrepl://localhost:7888`, or from a terminal:

```bash
clojure -Sdeps '{:deps {nrepl/nrepl {:mvn/version "1.1.2"}}}' -M -m nrepl.cmdline --connect --host localhost --port 7888
```

### 9.1 — Smoke test the Pulsar history cache

```clojure
(require '[com.little-trader.fast-streaming.history :as h])

;; Seed the warm cache — this is what the REST handler does.
(def btc-1h "persistent://public/default/market-data.btc.ohlcv.1h")
(def btc-1d "persistent://public/default/market-data.btc.ohlcv.1d")

(time (count (h/get-bars btc-1h)))         ; first call: Pulsar scan
;; => "Elapsed time: 2300 msecs" ;; ≈ 26000
(time (count (h/get-bars btc-1h)))         ; second call: warm cache
;; => "Elapsed time: 0.2 msecs"

(take 3 (h/get-latest-bars btc-1h 3))
;; => ({:time 1713535200 :open 64100.5 :high 64200.0 :low 64000.0 :close 64150.0 :volume ...} ...)

(h/get-bars-since btc-1d (- (quot (System/currentTimeMillis) 1000) (* 30 86400)))
;; => last 30 daily bars
```

### 9.2 — Inspect cache internals

```clojure
;; (the exact var name depends on what #119 renamed it to; grep for
;;  'topic-cache' or 'cache' in history.clj if this line errors)
@#'h/topic-cache
;; => {"persistent://public/.../btc.ohlcv.1h" {:bars [...] :consumer ... :client ...}
;;     "persistent://public/.../btc.ohlcv.1d" {:bars [...] ...}}

(for [[topic {:keys [bars]}] @#'h/topic-cache]
  [topic (count bars) (:time (first bars)) (:time (last bars))])
```

### 9.3 — Drive the EQL resolver directly

```clojure
(require '[com.little-trader.components.parser :as p])

(def parser (p/make-parser))

(parser {}
  [{[:fs/ohlcv {:market "btc" :timeframe :1h :limit 10}]
    [:count :first-time :last-time :topic]}])
;; => {[:fs/ohlcv {...}] {:count 10 :first-time ... :last-time ... :topic "..."}}

(parser {}
  [{[:fs/ohlcv {:market "eth" :timeframe :1d :limit 365}]
    [:count :first-time :last-time]}])
```

### 9.4 — Manually trigger a bounded backfill (if you need to seed a new market)

```clojure
(require '[com.little-trader.fast-streaming.backfill :as bf])

;; Note: this runs *on the workstation JVM*, publishing to the tunneled
;; broker at pulsar://localhost:6650 — your local JVM is a Pulsar PRODUCER
;; in this case. For production backfills, always drive the fs-worker VM
;; via POST /backfill (§7). This REPL path is for quick ad-hoc top-offs.

(bf/backfill-market! :market "btc" :days 7 :resolution "60"
                     :pulsar-url "pulsar://localhost:6650")
;; => {:market "btc" :instrument "BTC-PERPETUAL" :total 168 ...}
```

### 9.5 — Inspect Pulsar directly from the REPL

```clojure
(import '(org.apache.pulsar.client.api PulsarClient Schema SubscriptionType))

(def client (-> (PulsarClient/builder)
                (.serviceUrl "pulsar://localhost:6650")
                (.build)))

(def reader (-> client
                (.newReader Schema/BYTES)
                (.topic "persistent://public/default/market-data.btc.ohlcv.1h")
                (.startMessageId org.apache.pulsar.client.api.MessageId/earliest)
                (.create)))

(loop [n 0]
  (when (.hasMessageAvailable reader)
    (let [msg (.readNext reader)]
      (when (zero? (mod n 5000))
        (println "bar" n "@" (.getEventTime msg)))
      (recur (inc n)))))

(.close reader)
(.close client)
```

### 9.6 — Force a cache invalidation (when you backfill mid-session)

```clojure
;; The per-topic warm cache doesn't TTL. If you just re-backfilled, flush
;; the entry so the next read re-scans.
(h/invalidate-topic! "persistent://public/default/market-data.btc.ohlcv.1h")
(time (count (h/get-bars "persistent://public/default/market-data.btc.ohlcv.1h")))
```

---

## 10. Tear-down (daily)

```bash
make stop                     # stops local containers
make pulsar-tunnel-down       # closes SSH tunnel
```

---

## 11. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `make dev-cloud-pulsar` fails with `No admin endpoint on localhost:8081` | Tunnel down | `make pulsar-tunnel-up` |
| Tunnel drops after ~10 min idle | SSH timeout | Add `ServerAliveInterval 30` to your `~/.ssh/config` for the VM host; re-run tunnel |
| `host.docker.internal` unresolvable inside local server container | Linux host | Edit `docker-compose.yml` service `server` → `extra_hosts: ["host.docker.internal:host-gateway"]` |
| REST returns `{"count":0}` | Topic empty — backfill didn't run | Check §7 worker status and re-trigger; check `FS_BACKFILL_DAYS` env on fs-worker |
| 1H topic has 26k, 1D has 0 | `ohlcv_agg` not consuming | `gcloud compute ssh $LT_FS_VM --command "cd ~/fs-worker && sudo docker compose logs ohlcv-agg --tail 50"` or restart worker |
| Retention command rejects `-1` | Pulsar < 2.8 | Upgrade Pulsar image, or use `--size 9999999 --time 525600` (1y in minutes) |
| CI `deploy-staging` fails with `Permission denied` on VPC connector | Service account missing `vpcaccess.user` | `gcloud projects add-iam-policy-binding $GCP_PROJECT_ID --member=serviceAccount:github-actions-deployer@$GCP_PROJECT_ID.iam.gserviceaccount.com --role=roles/vpcaccess.user` |
| CI `update-fs-worker` fails pulling image | VM service account missing `roles/artifactregistry.reader` | `gcloud projects add-iam-policy-binding $GCP_PROJECT_ID --member=serviceAccount:<VM_SA> --role=roles/artifactregistry.reader` |
| Local Datomic refuses connections | Transactor on `$LT_INFRA_VM` not healthy | `gcloud compute ssh $LT_INFRA_VM --command "cd ~/infra && sudo docker compose ps"` — restart if `Exit` |

---

## 12. Appendix — key file map

| File | Role |
|---|---|
| `Makefile` | Targets: `pulsar-tunnel-up`, `pulsar-configure-cloud`, `pulsar-stats`, `dev-cloud-pulsar`, `cloud-deploy-staging` |
| `scripts/lt` | CLI: `pulsar-tunnel`, `pulsar-configure`, `pulsar-admin`, `dev`, `cloud` |
| `scripts/pulsar-configure-retention.sh` | Idempotent retention applier (used by init container + operator) |
| `docker-compose.yml` | Local dev stack (core, market, monitor) |
| `docker-compose.cloud.yml` | Overlay pointing local containers at cloud resources |
| `docker-compose.worker.yml` | fs-worker VM stack (Pulsar + fs-worker + retention-init) |
| `cloud-run/setup.sh` | GCP bootstrap (APIs, GAR, SA, Workload Identity) |
| `cloud-run/setup-infra.sh` | Datomic/Postgres VM provisioning |
| `cloud-run/setup-worker.sh` | fs-worker VM provisioning |
| `.github/workflows/main.yml` | CI/CD pipeline |
| `src/com/little_trader/fast_streaming/worker.clj` | Main polling loop (live) |
| `src/com/little_trader/fast_streaming/backfill.clj` | Deribit→Pulsar historical backfill |
| `src/com/little_trader/fast_streaming/ohlcv_agg.clj` | 1H → 1D rollup |
| `src/com/little_trader/fast_streaming/history.clj` | Server-side warm cache (#119) |
| `src/com/little_trader/components/fs_resolvers.clj` | EQL resolver `fs-ohlcv-history` |

---

## 13. What to read next

1. [ADR 0117](../adr/0117-market-streams-sourced-from-pulsar-cache.md) — why Pulsar and not Datomic.
2. [docs/ops/pulsar-setup.md](../ops/pulsar-setup.md) — retention command reference.
3. [docs/ops/cloud-pulsar-walkthrough.md](../ops/cloud-pulsar-walkthrough.md) — shorter operator card (section 6–9 of this doc).
4. Ticket [#119](https://github.com/4coders-com-br/little-trader/issues/119) — consumer/UI work that lands on top of this.
