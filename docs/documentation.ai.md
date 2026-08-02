# Vesta — AI/LLM reference

> S3-compatible object storage server, written in Rust, single binary,
> optional Raft-replicated cluster (custom or openraft), separate admin
> console, S3-native admin API, read-only MCP server for agent access.

This file is written for coding agents/LLMs, not humans — see
[the standard this follows](https://github.com/iwasoftcom/vesta) and its
rationale. It is complete on its own: install, call the S3 API, and
configure the admin surface from this file alone.

**Honest status:** early development (v0.1.0). Not yet independently
security-audited. No production mileage yet.

## Install / run

```bash
# Docker (recommended)
docker run -p 9000:9000 -p 9500:9500 iwasoftcom/vesta:0.1.0

# Debian/Ubuntu
sudo dpkg -i vesta_0.1.0_amd64.deb   # installs vesta.service (systemd), disabled by default
sudo systemctl enable --now vesta

# RHEL/Fedora
sudo rpm -i vesta-0.1.0.x86_64.rpm
sudo systemctl enable --now vesta

# Windows (native service)
vesta.exe service-install   # registers with SCM, LocalSystem, AutoStart
sc start Vesta

# Bare binary (dev/test)
VESTA_DATA_DIR=./vesta-data VESTA_ADDR=127.0.0.1:9000 vesta
```

Two ports, two processes:

| Process | Default address | Purpose |
|---|---|---|
| `vesta` (storage node) | `127.0.0.1:9000` (`0.0.0.0:9000` in the Docker image) | S3 API + `/_vesta/admin/*` control-plane API |
| `vesta-admin` (management app) | `0.0.0.0:9500` | Web UI, proxies to a storage node — stateless, holds no data itself |

`vesta-admin` is a separate binary/process from `vesta`. Run both if you want
the web console; `vesta` alone is a complete, fully functional S3 server.

## Configuration (environment variables)

All on the `vesta` binary unless noted.

| Variable | Default | Meaning |
|---|---|---|
| `VESTA_ADDR` | `127.0.0.1:9000` | Listen address for the S3 + admin API |
| `VESTA_DATA_DIR` | `./vesta-data` | All persistent state: metadata WAL, blocks, raft log |
| `VESTA_ACCESS_KEY` / `VESTA_SECRET_KEY` | unset | **Not read by the `vesta` server binary** (a stale doc claim existed in earlier README revisions — do not rely on it). These two are consumed only by the `vesta-mcp` client, to sign its outgoing requests. To require SigV4 on the server, create at least one identity via the admin API — see "Admin console" below; enforcement then turns on automatically |
| `VESTA_ADMIN_USER` / `VESTA_ADMIN_PASS` | unset | Bootstrap the first admin console user at startup (argon2-hashed); optional — see "Admin console" below for the no-env path |
| `VESTA_LOG` | `info` | `tracing_subscriber::EnvFilter` syntax, e.g. `debug`, `vesta_server=debug` |
| `VESTA_REGION` | `us-east-1` | SigV4 signing region |
| `VESTA_ERASURE` | unset (plain filesystem blocks) | `k+m` e.g. `4+2` — Reed–Solomon erasure coding |
| `VESTA_ENCRYPTION_KEY` | unset (no encryption) | Any string → AES-256-GCM encryption at rest (dedup-safe) |
| `VESTA_TLS_CERT` / `VESTA_TLS_KEY` | unset (plain HTTP) | PEM paths → serve HTTPS instead |
| `VESTA_CORS_ORIGIN` | unset (no CORS headers) | Seed value only — after first boot this is a live admin-console setting, not an env var (see below) |
| `VESTA_RATE_LIMIT_RPS` / `VESTA_RATE_LIMIT_BURST` | unset / seed `1000`/`2000` | Seed values only — after first boot, rate limiting is a live admin-console setting |
| `VESTA_GC_INTERVAL_SECS` | seed `300` | Seed value only — after first boot, a live admin-console setting (block GC mark-sweep interval) |
| `VESTA_NODE_ID` + `VESTA_PEERS` | unset (standalone) | Static-peer cluster mode: `VESTA_NODE_ID=0` (ordinal), `VESTA_PEERS=http://n0:9000,http://n1:9000,http://n2:9000` (all node URLs, in order) |
| `VESTA_SELF_URL` + `VESTA_SEED` | unset | Dynamic-join bootstrap mode: node starts alone at `VESTA_SELF_URL`; `VESTA_SEED=1` on exactly one node to become the initial leader, others join later via the admin console |
| `VESTA_CONSENSUS` | unset (custom raft) | `openraft` → use the openraft-backed consensus engine instead of the built-in one. Only valid with `VESTA_NODE_ID`+`VESTA_PEERS` |
| `VESTA_ADMIN_ADDR` | `0.0.0.0:9500` | `vesta-admin` binary: its own listen address |
| `VESTA_ADMIN_NODES` | `http://127.0.0.1:9000` | `vesta-admin` binary: comma-separated storage node URLs it proxies to |
| `VESTA_MCP_NODE` | `http://127.0.0.1:9000` | `vesta-mcp` binary: which node to query |
| `VESTA_MCP_TRANSPORT` | `stdio` | `vesta-mcp` binary: `stdio` (MCP standard) |
| `VESTA_MCP_ADDR` | `127.0.0.1:9600` | `vesta-mcp` binary: only used if an HTTP transport is selected |
| `VESTA_MCP_ALLOW_WRITE` | unset (read-only) | `vesta-mcp` binary: `1` enables write tools; default is strictly read-only |

**"Seed value" env vars**: `VESTA_RATE_LIMIT_*`, `VESTA_GC_INTERVAL_SECS`,
`VESTA_CORS_ORIGIN` only take effect the very first time a node starts (no
`settings.json` yet in `VESTA_DATA_DIR`). After that, they are live settings
mutated through the admin API/console and replicated cluster-wide; the env
vars are ignored on subsequent restarts. Do not expect changing the env var
and restarting to change behavior after first boot — use the admin API.

## S3 API quickstart

Any S3 client works (aws-cli shown; SigV4 required only if
`VESTA_ACCESS_KEY`/`VESTA_SECRET_KEY` were set):

```bash
aws --endpoint-url http://127.0.0.1:9000 s3 mb s3://photos
aws --endpoint-url http://127.0.0.1:9000 s3 cp ./x.jpg s3://photos/x.jpg
aws --endpoint-url http://127.0.0.1:9000 s3 ls s3://photos
aws --endpoint-url http://127.0.0.1:9000 s3 rm s3://photos/x.jpg
```

Raw HTTP also works without any SDK, but only in **anonymous mode** — i.e.
only on a node where no API key has ever existed (see the enforcement note
below; a node you've already created a key on, e.g. by following "Admin
console" below, will reject these with 403):

```bash
curl -X PUT http://127.0.0.1:9000/photos              # create bucket
curl -X PUT http://127.0.0.1:9000/photos/x.jpg --data-binary @x.jpg
curl http://127.0.0.1:9000/photos/x.jpg -o x.jpg       # get object
curl -X DELETE http://127.0.0.1:9000/photos/x.jpg
```

**SigV4 enforcement is dynamic, not a boot-time flag.** A fresh node with no
API keys is anonymous. The instant ANY identity exists — via
`VESTA_ACCESS_KEY`/`VESTA_SECRET_KEY` at startup, or a key created later
through the admin API/console — SigV4 becomes mandatory for the S3 API,
immediately, no restart. This is one-directional: there is no config to have
some keys exist while anonymous access still works.

Implemented: PutObject/GetObject/HeadObject/DeleteObject, ListObjectsV2,
range GET, conditional GET/PUT, CopyObject, multipart upload (all
operations), batch DeleteObjects, bucket versioning + delete markers, Object
Lock/WORM (GOVERNANCE/COMPLIANCE + legal hold), object tagging, bucket
policy + canned ACLs, SigV4 (headers + presigned URLs + streaming chunks),
`?select` (S3 Select, CSV SQL), `?search=k=v` (native metadata search),
`?inventory` (CSV report), `?lifecycle`, `?notification`.

## Admin console — address, bootstrap, and the operations you'll do most

**Address:** `http://<VESTA_ADMIN_ADDR>` — default `http://localhost:9500`.
It's a browser UI (React) that proxies to a storage node
(`VESTA_ADMIN_NODES`, default `http://127.0.0.1:9000`) — it holds no data of
its own.

Every node also exposes the same operations directly as a JSON API at
`http://<node>:9000/_vesta/admin/*` (what the console itself calls). Auth:
header `x-vesta-user` / `x-vesta-pass` (a dashboard user's credentials — see
bootstrap below). While zero dashboard users exist, these endpoints are
**open, unauthenticated, and act as admin** — this is the bootstrap window,
not a bug: create the first user immediately and it closes.

**Getting the first admin credential (two ways):**

1. Env var at node startup: `VESTA_ADMIN_USER=admin VESTA_ADMIN_PASS=...` —
   creates an argon2-hashed dashboard admin before anything else can connect.
2. No env var: open the console (or call the API) while no dashboard users
   exist yet — you're implicitly admin. Create your real user immediately:

```bash
curl -X POST http://127.0.0.1:9000/_vesta/admin/users \
  -H 'content-type: application/json' \
  -d '{"username":"admin","password":"change-me","role":"admin"}'
# role: "admin" or "viewer". Once >=1 user exists, the bootstrap window is closed;
# all further calls need x-vesta-user/x-vesta-pass headers.
```

**Create a bucket** (owner/tenant optional — omit for a tenant-less bucket):

```bash
curl -X POST http://127.0.0.1:9000/_vesta/admin/buckets \
  -H 'x-vesta-user: admin' -H 'x-vesta-pass: change-me' \
  -H 'content-type: application/json' \
  -d '{"name":"photos","owner":""}'
```
(Or just use the S3 API directly: `aws s3 mb` — no admin auth needed in
anonymous mode. The admin endpoint is for when you want it tenant-scoped or
you're driving everything through one API.)

**Create a tenant** (required before creating an API key — keys must belong
to an existing tenant):

```bash
curl -X POST http://127.0.0.1:9000/_vesta/admin/tenants \
  -H 'x-vesta-user: admin' -H 'x-vesta-pass: change-me' \
  -H 'content-type: application/json' \
  -d '{"name":"acme-corp"}'
```

**Create an API key** (SigV4 access/secret key pair, scoped to a tenant;
omit `access_key`/`secret_key` to auto-generate both):

```bash
curl -X POST http://127.0.0.1:9000/_vesta/admin/keys \
  -H 'x-vesta-user: admin' -H 'x-vesta-pass: change-me' \
  -H 'content-type: application/json' \
  -d '{"tenant":"acme-corp"}'
# → {"access_key":"VESTA...","secret_key":"...","tenant":"acme-corp"}
```

**Live runtime settings** (rate limit, block-GC interval, CORS — replicated
cluster-wide, no restart):

```bash
curl -X POST http://127.0.0.1:9000/_vesta/admin/settings/ratelimit \
  -H 'x-vesta-user: admin' -H 'x-vesta-pass: change-me' \
  -H 'content-type: application/json' \
  -d '{"enabled":true,"rps":50,"burst":100}'

curl -X POST http://127.0.0.1:9000/_vesta/admin/settings/gc-interval \
  -H 'x-vesta-user: admin' -H 'x-vesta-pass: change-me' \
  -H 'content-type: application/json' -d '{"secs":60}'

curl -X POST http://127.0.0.1:9000/_vesta/admin/settings/cors \
  -H 'x-vesta-user: admin' -H 'x-vesta-pass: change-me' \
  -H 'content-type: application/json' -d '{"origin":"*"}'
```

Reading current values: `GET /_vesta/admin/settings` (same auth).

## Architecture facts that affect integration

- **Content-addressed, deduplicated storage.** Blocks are keyed by SHA-256 of
  their content; identical bytes anywhere are stored once. Encryption and
  erasure coding both operate below this layer and don't break dedup.
- **Consensus is pluggable, not decorative.** Cluster mode (`VESTA_NODE_ID`+
  `VESTA_PEERS`) replicates buckets/objects/IAM/settings through a real
  Raft log — either the built-in implementation or `openraft` (`VESTA_CONSENSUS=
  openraft`). A write is committed cluster-wide (quorum ack) before the HTTP
  response returns — reads immediately after a write are consistent.
- **Metadata durability is per-node AND cluster-wide.** Each node persists
  its own append-only WAL (`VESTA_DATA_DIR/metadata.json` + `.wal`)
  independent of cluster membership; a node that rejoins after downtime
  catches up via normal log replication, not a special recovery path.
  Restarting the entire cluster (all nodes) does not lose data.
  There is no automatic snapshot-based catch-up mechanism yet — a node that
  falls arbitrarily far behind still catches up correctly, just via replaying
  the full log (fine at this project's current scale; a future limitation to
  know about, not a current one).
- **Multi-tenancy is real, not cosmetic.** A `tenant` is a first-class quota
  + isolation boundary: SigV4 keys belong to exactly one tenant, and bucket
  quotas are enforced per-tenant. A bucket's `owner` field ties it to a
  tenant for those quota checks.
- **Graceful shutdown is real.** SIGTERM (Linux/Docker/K8s), Windows Service
  Stop, and Ctrl-C all drain in-flight requests before exiting — safe to
  restart under a process supervisor or orchestrator without truncating
  writes.
- **Rate limiting is per-tenant, not global**, and returns a proper S3
  `SlowDown` error (not a dropped connection) — clients following normal S3
  SDK retry/backoff behavior handle it correctly without special-casing.

## Prometheus metrics & health

- `GET /metrics` — Prometheus text format 0.0.4: request counts/latency
  (avg/p50/p95), bucket/object/data-size gauges, `vesta_is_leader`.
- `GET /healthz` — liveness (process up).
- `GET /readyz` — readiness: `200` only when this node is the leader or
  knows the leader (`503` during an election) — use this, not `/healthz`,
  for a Kubernetes/load-balancer readiness probe.

## MCP server (agent tool access)

`vesta-mcp` is a separate, stateless, **read-only by default** MCP server —
gives an agent native search and S3 Select as tools, scoped to a tenant via
the SigV4 key it's given (tenant isolation comes free from the key, no
extra config).

```bash
VESTA_MCP_NODE=http://127.0.0.1:9000 \
VESTA_ACCESS_KEY=VESTA... VESTA_SECRET_KEY=... vesta-mcp
```

Transport is MCP-standard stdio; point any MCP-compatible agent host
(Claude Desktop, Claude Code, etc.) at the `vesta-mcp` binary. Set
`VESTA_MCP_ALLOW_WRITE=1` to also expose write tools (off by default —
read-only is the safe default for giving an agent access to production data).

## Downloads & source

- Downloads (Windows x64 `.zip`, Debian `.deb`, RedHat `.rpm`) and the
  Docker image: `https://iwasoft.com` → Products → Vesta. Source code is
  not part of the downloads (closed-source; this doc + a public README/docs
  mirror live at `github.com/iwasoftcom/vesta`).
- Docker: `docker pull iwasoftcom/vesta:0.1.0` (also `:latest`).
- Implements: the AWS S3 API (SigV4, REST). Not a fork of MinIO or any other
  existing object store — an independent Rust implementation.
