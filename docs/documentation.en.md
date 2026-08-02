Vesta

the S3-compatible object store that closes the gaps · v0.1.0

A layered, S3-compatible object storage system written in Rust — a single binary that scales from a laptop to a Raft-replicated cluster, without switching software.

**Building with AI?** Give your coding agent/LLM this link instead of this page — a dense, machine-optimized reference (install, every env var and default, exact API calls) it can act on directly, not marketing prose it has to reverse-engineer: [documentation.ai.md](https://iwasoft.com/products/vesta/0.1.0/docs/documentation.ai.md)

## What Vesta is

Vesta targets the feature gaps found across today's object stores (S3, GCS, Azure Blob, R2, B2, Wasabi, MinIO, Ceph, SeaweedFS, Garage). It speaks the real S3 API — SigV4 signing, multipart upload, versioning, conditional requests, batch delete — and separates the **control plane** (metadata: buckets, object index, IAM) from the **data plane** (content-addressed blocks on disk) so each can be replaced or scaled independently.

## Design principles

**Control plane / data plane separation.**  
Metadata and bytes live behind separate trait boundaries. Storage engines, consensus backends and encryption layers are swapped without touching the S3 API layer.

**No admin knob left in a config file.**  
Rate limits, GC intervals, CORS, quotas and policy are runtime settings, replicated and changed live from the admin console — not environment variables that need a restart.

**Compatibility is a contract, not an approximation.**  
SigV4 (headers, presigned URLs, streaming chunks), multipart, versioning and conditional requests are exercised against real AWS SDK test suites, not hand-picked examples.

## How it differs from a typical single-binary object store

|  | Typical MinIO-style store | Vesta |
| --- | --- | --- |
| Consensus | Fixed erasure-set / gateway model | Networked Raft with dynamic membership — a battle-tested engine ([openraft](#architecture)) is a drop-in, opt-in backend behind the same write path |
| Runtime configuration | Environment variables, restart to change | Admin console mutates live settings (rate limit, GC interval, CORS, quotas) through the replicated log — no restart |
| Metadata durability | Varies by backend | Append-only WAL with snapshot compaction; every node persists independently and catches up via normal log replication |
| Multi-tenancy | Bolted on or absent | First-class tenants with per-tenant bucket quotas and SigV4 identity scoping |
| AI agent access | Not applicable | A read-only [MCP server](#more) exposes native search and S3 Select as agent tools, with per-key tenant isolation |

## Quick start

Run the server (container image, or the standalone binary):

```
# Docker
docker run -p 9000:9000 iwasoftcom/vesta:0.1.0

# or the binary
VESTA_DATA_DIR=/var/lib/vesta vesta
```

Talk to it with any S3 client:

```
aws --endpoint-url http://127.0.0.1:9000 s3 mb s3://photos
aws --endpoint-url http://127.0.0.1:9000 s3 cp ./x.jpg s3://photos/x.jpg
aws --endpoint-url http://127.0.0.1:9000 s3 ls s3://photos
```

## What's inside

**Rate limiting**  
Per-tenant token bucket, enabled and tuned live from the admin console; misbehaving callers get a proper `SlowDown`, not a dropped connection.

**Distributed consensus**  
A networked Raft with leader election, dynamic membership and durable log replication — or opt into `openraft`, a proven implementation, behind the identical write path.

**Erasure coding & encryption**  
Reed–Solomon erasure-coded storage and AES-256-GCM encryption at rest, both dedup-safe (content-addressed blocks).

**Versioning & Object Lock**  
Full version history, delete markers, and WORM retention (GOVERNANCE/COMPLIANCE) with legal hold.

**Multi-tenancy**  
Tenants are first-class: per-tenant bucket quotas, SigV4 identity scoping, bucket policy and canned ACLs.

**Search, Select & Lambda**  
Native inverted-index metadata search, S3 Select (CSV SQL), and transform-on-read (Object Lambda–style).

**Replication & events**  
Async geo-replication, a change-stream event bus, and pluggable webhook notification delivery.

**Lifecycle & inventory**  
Expiration and storage-class transition rules, plus CSV inventory reports on demand.

## Admin console

A separate, stateless management app (embedded React + MUI UI) proxies writes to a storage node — it holds no data of its own; every change is replicated through the same consensus log the S3 API uses.

<table><tbody><tr><th>Address</th><td><code>http://localhost:9500</code> (env <code>VESTA_ADMIN_ADDR</code>, default <code>0.0.0.0:9500</code>)</td></tr><tr><th>Talks to</th><td>a storage node's admin API, default <code>http://127.0.0.1:9000</code> (env <code>VESTA_ADMIN_NODES</code>)</td></tr><tr><th>Default user</th><td>none — the console acts as admin, open, until the <b>first</b> dashboard user is created (Users screen), which closes it. Or seed one at node startup: <code>VESTA_ADMIN_USER</code>/<code>VESTA_ADMIN_PASS</code></td></tr></tbody></table>

Every node also exposes the same operations as a plain JSON API at `http://<node>:9000/_vesta/admin/*` (what the console itself calls) — handy for scripting. The three things you'll do first:

```
# 1) Create a bucket
curl -X POST http://127.0.0.1:9000/_vesta/admin/buckets \
  -H 'content-type: application/json' -d '{"name":"photos"}'

# 2) Create a tenant (required before an API key)
curl -X POST http://127.0.0.1:9000/_vesta/admin/tenants \
  -H 'content-type: application/json' -d '{"name":"acme-corp"}'

# 3) Create an API key (SigV4 access/secret pair) for that tenant
curl -X POST http://127.0.0.1:9000/_vesta/admin/keys \
  -H 'content-type: application/json' -d '{"tenant":"acme-corp"}'
# → {"access_key":"VESTA...","secret_key":"...","tenant":"acme-corp"}
```

Once a dashboard user or API key exists, these calls need `x-vesta-user`/`x-vesta-pass` headers (a dashboard user's credentials) — and creating that first key automatically turns on SigV4 enforcement for the S3 API, cluster-wide, no restart.

-   **Users, keys & tenants** — dashboard accounts, SigV4 API keys, per-tenant quotas.
-   **Buckets & policy** — create/delete, bucket policy JSON, public-read toggles.
-   **Cluster** — node health, add/remove members, minority-write and auto-shrink toggles.
-   **Runtime settings** — rate limit, block-GC interval, CORS origin: changed live, replicated to every node, persisted across restarts.

## Architecture

A single binary, two network doors, and a strict layering rule: the S3 API layer never touches storage directly — everything routes through the coordinator, and every mutation that must be cluster-wide goes through the consensus log before it is visible to a read.

S3 SDKs · aws-cli SigV4 · multipart · versioning Admin console · AI agents (MCP) stateless proxy · tenant-scoped tools S3 API · :9000 Admin API · :9500 coordinator (Rust): buckets · objects · multipart · policy · lifecycle · search consensus log (custom Raft or openraft) — mutations commit before they read back metadata (WAL) · block storage (erasure-coded, encrypted, deduped)

## Downloads & source

-   **Downloads:** compiled artifacts (Windows, Debian `.deb`, RedHat `.rpm`) and the Docker image are published per version on [iwasoft.com](https://iwasoft.com) → Products → Vesta. Source code is not part of the downloads.
-   **Compatibility:** S3 API surface (SigV4, multipart, versioning, conditional requests) is exercised continuously against real AWS SDK integration tests.
-   **Honest status:** early development — not yet independently security-audited, no production mileage yet. These are disclosures, not caveats on the roadmap.

Vesta v0.1.0 · S3-compatible · Rust, content-addressed storage, networked Raft (custom or openraft). © iwasoft.
