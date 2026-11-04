# DuckDB 2.0 and its new server mode (Quack)

Research date: 2026-11-03. Based on roughly 15 web searches and 30 page reads: DuckDB's official blog posts and docs, plus secondary coverage from InfoQ, The Stack, AlternativeTo, I Programmer, MotherDuck, Definite, GitHub and RuntimeWire. All benchmark numbers come from vendors or individual blogs. I found no independent production reviews yet, because the release is only two weeks old.

---

## 1. The short version

- **DuckDB 2.0.0 LTS, codename "Cinnamon Teal"**, shipped on **2026-10-21**. The August preview called it "Cyanoptera", which is the Latin name of the same bird (*Spatula cyanoptera*). It is a long-term-support release supported until **at least October 2027**. The team counts close to 20,000 commits since 1.5 and calls 2.0 a *feature* release. 1.0 was the *stability* release.
- **Headline feature: DuckDB can now run as a server.** The `quack` extension implements an HTTP-based native protocol. Any DuckDB process can serve its databases, and any other DuckDB (CLI, Python, Wasm in a browser, or third-party JDBC/ADBC drivers) can attach to it and run queries remotely. This is DuckDB's first official client/server mode. Its main practical win is **multiple concurrent writers to one DuckDB database**, through a single server process.
- A new **`CONNECT` / `DISCONNECT`** SQL statement points a whole session at a remote database: a Quack server, PostgreSQL or MySQL. For Postgres and MySQL, a new **remote pushdown optimizer** sends SQL to the remote engine instead of pulling tables over the wire.
- Other big 2.0 items:
  - asynchronous I/O (about 3x faster Parquet on S3 and about 20x on large CSV, by the team's own benchmarks)
  - VARIANT shredding
  - a PEG parser as the default
  - triggers
  - a CLI "agent mode"
  - C API v2
  - pinned private extension repositories
  - a new storage format (version 69) that older versions can't read
  - a set of behavior-changing breaking changes
- CEO Hannes Mühleisen (via The Stack): *"Where last year was the year of the lakehouse, this release kicks off the year of DuckDB as a server."*

## 2. Timeline

| Date | Event |
|---|---|
| 2025-09-16 | DuckDB 1.4.0 LTS "Andium" |
| 2026-03-09 | DuckDB 1.5.0 "Variegata". The PEG parser becomes available as an opt-in. |
| 2026-05-12 | Blog post "Quack: The DuckDB Client-Server Protocol". Quack is revealed as a preview. |
| 2026-05-20 | 1.5.3 "Not an Ordinary Patch Release". Quack becomes an auto-installing, auto-loading **core extension in beta**, and DuckLake can use a Quack server as its catalog. |
| 2026-06-24 | DuckCon #7, Amsterdam |
| 2026-07-31 | Blog post on asynchronous I/O |
| 2026-08-17 | "A Preview of DuckDB v2.0": CONNECT, the remote pushdown optimizer, Quack graduating to stable |
| 2026-08-20 | PEG parser post |
| 2026-09-02 | "Try DuckDB v2.0-dev". Feature freeze; a `v2.0-cyanoptera` branch is cut and an alpha channel opens. Quack moves from 0.x to 1.0. |
| 2026-10-21 | **DuckDB 2.0.0 LTS released** (Python package 2.0.0 the same day) |
| 2026-10-26 | Blog post "Gems of DuckDB 2.0.0" |
| 2026-11-17 | 1.4 LTS end of life |
| Nov 2026 | First 2.0 patch release expected; a secondary source says 2.0.1 is provisionally set for 2026-11-18. It had not shipped as of 2026-11-03. |
| Mar 2027 | 1.5 end of support (extended to give people time to migrate) |
| Oct 2027 | Earliest end of 2.0 LTS support |

## 3. Server mode in depth (Quack)

### 3.1 Why it exists
DuckDB has been in-process (embedded, like SQLite) since its start. Within one process, many threads can read and write (MVCC plus optimistic concurrency). Across processes, a database file is either opened by **one read-write process** or by **many read-only processes**. Workloads like "telemetry ingestion plus dashboards", or several services sharing one database, didn't fit that model.

People built their own workarounds: custom RPC services, Arrow Flight SQL wrappers, MotherDuck, putting Postgres in front, and pg_duckdb. The team read that as demand and, in its own words, "finally caved".

### 3.2 Protocol design
- **Transport: HTTP(S).** The team chose HTTP because existing infrastructure handles it: load balancers, firewalls, intrusion detection and auth proxies. It also lets DuckDB-Wasm in a browser speak Quack natively. Their quote: "Everyone and their little brother knows how to deal with HTTP."
- **Client-initiated request/response only.** The server never pushes.
- **Serialization:** a new MIME type, `application/duckdb`, built on DuckDB's internal serialization (the same one its write-ahead log uses). Vectors go over the wire nearly as-is, with no Arrow conversion, and complex types (nested, DECIMAL, INTERVAL, VARIANT) survive intact.
- **One round trip per query** after the handshake. Large results stream through follow-up `FETCH` requests, which can run in parallel across threads.
- **Default port 9494.** URIs use the `quack:` scheme, for example `quack:localhost`, `quack:host:9000` or `quack:[::1]:1234`.
- **Why not Arrow Flight SQL:** the team wants control over its own types and protocol messages, and Flight needs at least two round trips per query, which hurts small writes.

### 3.3 Using it

**Server (SQL):**
```sql
CALL quack_serve(token = 'my_token');           -- localhost:9494, plain HTTP
-- returns the listen URI, the HTTP URL and the auth token (generated if you don't pass one)
```

**Client (SQL):**
```sql
ATTACH 'quack:server.example.com' AS qk (TOKEN 'my_token');
CONNECT qk;                       -- route the whole session to the server
SELECT count(*) FROM events;      -- executes on the server, results stream back
DISCONNECT;                       -- back to the local database
```

There are three ways for a client to use a server:
1. **`ATTACH` + `CONNECT`.** Recommended. Plain SQL runs remotely.
2. **`ATTACH` only.** The remote is mounted as a catalog and you qualify names (`qk.events`). DDL, writes and filters work like local ones, filters are pushed to the server, and `BEGIN`/`COMMIT` are forwarded. Each attachment exposes `qk.query('...')` for full-query pushdown and a `whoami()` macro.
3. **`quack_query(uri, sql, token := ...)`.** Stateless one-off queries with no attach.

For tokens, the docs recommend a `quack` secret scoped to the server URI. A `TOKEN` passed directly on `ATTACH` overrides it.

**CLI (new in 2.0):**
```
duckdb -serve                                    # in-memory DB on localhost:9494
duckdb analytics.duckdb -serve                   # serve a file
duckdb analytics.duckdb -serve quack:analytics   # using the named secret "analytics"
duckdb -connect                                  # connect as a client (default type quack)
duckdb -connect quack:analytics
```
`-connect` can't be combined with a local database path.

**Server management functions:**
- `quack_stop(uri)`
- `quack_server_list()`: URI, port, active connection count, SSL
- `quack_active_connections()`: per-connection last query, state, cached rows
- `quack_cancel(catalog, connection_id)`
- `quack_generate_keys()`: writes `server.pem` and `private_key.pem`
- `quack_identify(...)` and `whoami()`: node identity (name, provider, region, uptime)

**Key settings and defaults (2.0 reference):**
| Setting | Default |
|---|---|
| `quack_server_max_connections` | 1024 (extra connections are refused) |
| `quack_server_keep_alive_timeout` | 300 s |
| `quack_target_batch_bytes` | 32 MiB |
| `quack_prepare_inline_rows` | 24,576 rows returned inline with PREPARE |
| `quack_server_stream_buffer_bytes` | 256 MiB per stream, capped at 1/4 of `memory_limit` |
| `quack_default_heartbeat_timeout` | 60 s |
| `quack_enable_reconnects` | false. When on, results are acknowledged and cached so clients can reconnect: `quack_result_ttl` 3600 s, `quack_cache_max_rows` 100k. |
| token / CLIENT_ID | minimum 4 characters |

### 3.4 Security model
- **Authentication:** a **shared token**. If you don't set one, a random token is generated at startup. There are no built-in users or roles.
- **Default bind is localhost only.** Binding elsewhere needs `allow_other_hostname = true`, which the release post calls "only meant for testing".
- **TLS:** local URIs use plain HTTP. Non-local servers default to HTTPS with a **self-signed certificate** unless you pass `ssl_cert_file`/`ssl_key_file`. Clients assume HTTPS for non-local hosts and can pin a self-signed certificate with `SSL_FINGERPRINT`.
- **Recommended production setup:** keep Quack on localhost and put **nginx or Caddy in front to terminate TLS**. The docs ship recipes. Gotchas from them:
  - Raise the request body limit to 256M. nginx's 1 MiB default "will fail mid-INSERT".
  - Raise the 60 s proxy timeouts (the recipe uses 600 s).
  - Use HTTP/1.1 keep-alive upstream. Quack's connection state depends on persistent connections.
  - Turn off buffering (`proxy_buffering off` in nginx, `flush_interval -1` in Caddy) so streamed FETCHes aren't held back.
  - The default port stays 9494 even behind TLS, so clients must write `quack:host:443` explicitly.
- **Pluggable auth hooks**, set with `SET GLOBAL`:
  - `quack_authentication_function (sid, client_token, server_token) -> BOOLEAN`. Runs once per connection.
  - `quack_authorization_function (sid, query) -> BOOLEAN or VARCHAR`. Runs before every query. If it returns a VARCHAR, that SQL runs **instead of** the client's query, so it can rewrite queries.
  - Both can be SQL macros. The docs' examples include a token table for multiple users and a read-only allowlist.
  - The docs warn that prefix checks like "starts with SELECT/WITH" are **not robust**: `WITH ... INSERT` gets through. For real read-only access, attach the database read-only or inspect the parsed statement type.
  - Macros can't run DML.
  - Python UDFs registered with `con.create_function` aren't visible to the callbacks, so they need to live in an extension.
- A Quack server exposes **everything its session can see**, read and write, to anyone holding the token.

### 3.5 Performance (DuckDB team's May 2026 benchmarks)
AWS m8g.2xlarge pair in the same availability zone (about 0.28 ms ping).

Bulk transfer of TPC-H lineitem (seconds, lower is better):

| Rows | Quack | Arrow Flight | PostgreSQL |
|---|---|---|---|
| 1M | 0.24 | 0.38 | 2.20 |
| 10M | 0.89 | 2.90 | 25.64 |
| 60M | 4.94 | 17.40 | 158.37 |

Single-row INSERT transactions (transactions per second, higher is better):

| Threads | Quack | Arrow Flight | PostgreSQL |
|---|---|---|---|
| 1 | 1,038 | 469 | 839 |
| 8 | 5,434 | 1,358 | 4,320 |

The team's own caveat: concurrent inserts into **one table hit a DuckDB-level ceiling above about 8 threads**, and Postgres scales better there. 2.0 claims "substantial performance and stability improvements" over the beta, but the release post gives no new Quack numbers.

Definite (an independent blog) ran 8 Quack clients each writing 250k rows to one table. That came to about 222k rows/s total versus about 192k rows/s for a single writer, with no errors. They call it a ceiling, not an SLA.

### 3.6 What Quack does *not* give you (limitations to plan around)
- **One server process holds the data.** Writes are serialized through that one DuckDB instance, so write throughput doesn't scale horizontally, and the server is both a bottleneck and a single point of failure. Clients get "multi-writer" access, but the *file* still has a single owner.
- **No replication, read replicas or HA yet.** A replication protocol over Quack is "being considered".
- **Auth is basic:** a shared token, no users, roles or SSO unless you write callbacks. MotherDuck: "as basic as possible, just a shared secret."
- **Row-level write conflicts** still fail under optimistic concurrency, and the client has to retry. Definite found that concurrent `CREATE TABLE IF NOT EXISTS` raced and was aborted, so they pre-create tables.
- **`ATTACH` binds every remote table definition.** One definition that fails to bind can fail the whole attach.
- **Crash recovery:** an open GitHub discussion (ducklake #1315, no replies yet) reports an orphaned WAL blocking server restart after an unclean shutdown.
- **Operations are on you:** process supervision, backups, TLS proxy and monitoring. 2.0 did rework metrics and logs for long-running instances.
- **Third-party drivers are alpha.** GizmoData's `quack-jdbc` (0.2.0-alpha) has no server-side bind parameters (they are substituted client-side), no FSST vector support, and limited nested type mapping. An `adbc-driver-quack` package exists on PyPI as 0.1.0a10. The Python 2.0.0 release notes don't mention Quack, but it works through SQL.

### 3.7 CONNECT beyond DuckDB
```sql
CONNECT 'postgres:dbname=mydb host=localhost';
SELECT count(*) FROM orders;   -- pushed down and run on PostgreSQL
DISCONNECT;
```
- CONNECT accepts an `ATTACH` alias or a typed connection string (`postgres:`, MySQL).
- Local in-process databases (in-memory or a file) can't be CONNECTed to.
- `DISCONNECT` with nothing connected is an error.
- It replaces the `remote.query($$...$$)` syntax shown in the early demos.
- The docs give no MySQL-specific pushdown coverage details.

### 3.8 DuckLake integration
- Since 1.5.3, a Quack server can be a **DuckLake catalog**. Clients reach the lakehouse metadata over HTTP with a token instead of each one holding a Postgres connection.
- Definite reports these advantages: no type translation (UBIGINT, nested types and VARIANT stored natively), no connection exhaustion, and faster reads of inlined data.
- Their caveats: a single writer pod with no HA, and catalog growth from inlined rows.
- DuckLake 1.1 ships with 2.0. The concurrency docs still list DuckLake with a Postgres catalog as the other route to multi-process read/write.

### 3.9 Quack vs. MotherDuck vs. Postgres (positioning)
- **Self-hosted Quack:** free, simple, single node, shared-token auth. Good for small teams, internal services, ingestion plus dashboards, and a shared DuckLake catalog.
- **MotherDuck:** positions itself as complementary. It offers what Quack lacks: per-user permissions, SSO, storage separated from compute, read scaling, time travel and zero-copy clones, and an SLA. It plans to support Quack as an endpoint type, with no firm date.
- **Postgres:** still the better fit for high-concurrency OLTP. DuckDB remains an OLAP engine, a point made repeatedly in the community discussion InfoQ summarized.

## 4. The rest of DuckDB 2.0

**Performance**
- **Asynchronous I/O** for Parquet, CSV and the native format. Separate REGULAR and ASYNC thread pools (ASYNC defaults to 4x system threads, capped at 256), a read-ahead queue, and memory governance (`read_ahead_depth`, `async_threads`).
  - Team benchmarks, TPC-H Q6 SF100 on S3: 8.23 s down to 2.84 s (about 3x), or 2.23 s tuned (about 3.7x). An 81 GB CSV went from 878 s to 45 s (about 20x). Concurrent queries finished 2.3x faster.
  - Little gain on a hot local cache. Read-ahead uses memory.
- **VARIANT** is fully integrated, with automatic **shredding** of common structure into columns (including Parquet reads and writes) and `variant_*` functions.
- Other numbers:
  - Native timezones and collations replace bundled ICU: `AT TIME ZONE` 2.2x faster, `COLLATE` 2.6x faster.
  - The preview shows a recursive-CTE graph reachability microbenchmark going from 4.90 s to 0.12 s.

**SQL**
- **Triggers** (`BEFORE`/`AFTER`, row- and statement-level, transition tables)
- `$variable` shorthand
- Data-modifying CTEs
- `NEAREST` joins (top-k similarity)
- Nested schemas
- `json_set`/`json_insert`/`json_replace`/`json_remove`
- `FETCH FIRST`
- `CREATE SECURE VIEW`
- `SHOW <setting>`
- `setval`
- `COPY ... PARTITION_PATH` / `ORDER_BY`
- GeoJSON writing

**Parser and extensibility**
- **PEG parser** is now the default. Extensions can add grammar, and error messages point at exact locations. The old parser is available through the `legacy_parser` extension.
- **C API v2**: versioned, with a YAML spec and a stable ABI goal. It is *not yet marked stable*.
- **Trusted extension repositories**: `CREATE EXTENSION REPOSITORY ... WITH PREFIX ...` with pinned RSA keys, gated by `allow_extension_repositories`.
- Extensions can define custom window functions.

**CLI and clients**
- **Agent mode** (`-agent`/`-no-agent`): auto-detects Claude Code, Codex and Cursor, then uses compact Markdown tables, JSON errors and up-front cost estimates. It cut model-read output by 59% on TPC-H.
- `.manual`, `.history` and `shell_history()`.
- Pretty-printed `.schema`.
- Calculator mode.
- **Python 2.0.0:**
  - Needs Python 3.11+.
  - Moved to nanobind.
  - Results stream.
  - New Appender.
  - Breaking API changes (for example, Arrow fetches moved to `con.sql(...).to_arrow_table()`, and bare strings in relation expressions are values, not column names).

**Lakehouse**
- DuckLake 1.1
- Iceberg REST catalog options
- Delta `CREATE TABLE`, time travel and VARIANT

## 5. Migration gotchas (from the 2.0 migration guide)
1. **Storage version 69.** Files created by 2.0 can't be opened by older versions (1.5.6 reads 64-68). 2.0 still reads and writes old files without upgrading them. To keep files compatible, use `-storage-version v1.5.6`, `storage_compatibility_version`, or `ATTACH ... (STORAGE_VERSION ...)`. **Mixed-version fleets are the main risk.**
2. **Parser changes that can silently change results:**
   - `~` / `!~` now do *partial* regex matching.
   - `2 * 3!` now returns 12 instead of 720.
   - `X'..'` and `B'..'` now produce BLOB and BITSTRING.
3. **Any error inside an explicit transaction aborts it**, including parser and binder errors.
4. **Integer `//` and `%` by zero now throw** instead of returning NULL.
5. **`MERGE` that hits the same target row twice now errors.**
6. **An untyped `NULL` column now has type "NULL"**, not INTEGER.
7. **Single-arrow lambdas (`x -> x+1`) are off by default.** Use `lambda x: x+1`. Both the old syntax and the flag that re-enables it are removed in 2.1.
8. **S3 credentials in URL parameters are disabled.**
9. **Some profiling and HTTP-logging settings were removed.**
10. **Python requires 3.11+.**
11. **Build and platform changes:** C++17 required; amalgamation deprecated; macOS x86_64 being phased out and dropped in 2.1.
12. **Advice:** shadow-test first. Mission-critical users should wait for the November patch release.

## 6. Bottom line
DuckDB 2.0 doesn't turn DuckDB into Postgres. It adds a **thin, fast, HTTP-based server mode** on top of the same single-node engine. That solves the long-standing "only one process can write" problem and lets browsers, CLIs, JVM tools and DuckLake share one live DuckDB.

- **Good fit:** internal shared analytics databases, ingestion plus dashboards, a team DuckLake catalog, and remote querying of Postgres or MySQL with pushdown.
- **Not yet a fit:** internet-facing multi-tenant production that needs per-user security, HA or horizontal write scaling. You'd build that yourself around Quack, or use MotherDuck or Postgres.
- **Maturity:** early. Quack was beta until four months before GA, the third-party drivers are alpha, and the first patch release is due in mid-November.

## 7. Discrepancies noted across sources
- **Codename:** "Cyanoptera" (preview and branch name) vs "Cinnamon Teal" (release). Same bird.
- **Commit count:** ">10,000 since 1.5" (InfoQ, August preview) vs "close to 20,000" (October release post). The two numbers are from different points in time.
- **Quack auto-load:** the May 12 post said auto install and load was "planned"; the May 20 1.5.3 post says Quack auto-installs and auto-loads.
- **CLI flags:** `-serve`/`-connect` are in the release post and the 2.0 CLI arguments page, but not on the Quack reference page.
- **quack-jdbc README:** says Quack would stabilize with v2.0 "in September 2026". GA was actually October 21.
- **Search-engine summaries:** several hedged that 2.0 "may not have shipped". The official blog, the docs and the PyPI/GitHub Python release all confirm GA on 2026-10-21.

## Sources
- [Announcing DuckDB 2.0.0 LTS "Cinnamon Teal"](https://duckdb.org/2026/10/21/announcing-duckdb-200)
- [A Preview of DuckDB v2.0](https://duckdb.org/2026/08/17/duckdb-20-highlights)
- [Quack: The DuckDB Client-Server Protocol](https://duckdb.org/2026/05/12/quack-remote-protocol)
- [DuckDB 1.5.3: Not an Ordinary Patch Release](https://duckdb.org/2026/05/20/announcing-duckdb-153)
- [Try DuckDB v2.0-dev](https://duckdb.org/2026/09/02/try-duckdb-20-alpha)
- [Gems of DuckDB 2.0.0](https://duckdb.org/2026/10/26/gems-of-duckdb-200)
- [Asynchronous I/O in DuckDB](https://www.duckdb.org/2026/07/31/asynchronous-io)
- [DuckDB v2.0: Your Database Deserves a Better Parser](https://duckdb.org/2026/08/20/duckdb-20-peg-parser)
- [Quack overview (2.0 docs)](https://duckdb.org/docs/2.0/quack/overview)
- [Quack reference (2.0 docs)](https://duckdb.org/docs/2.0/quack/reference)
- [Quack security (2.0 docs)](https://duckdb.org/docs/2.0/quack/security)
- [Quack reverse proxy setup](https://duckdb.org/docs/2.0/quack/setup/reverse_proxy)
- [CONNECT statement](https://duckdb.org/docs/2.0/sql/statements/connect)
- [CLI arguments (2.0)](https://duckdb.org/docs/2.0/clients/cli/arguments)
- [Concurrency (2.0)](https://duckdb.org/docs/2.0/connect/concurrency)
- [Migration guide (2.0)](https://duckdb.org/docs/2.0/operations_manual/migration)
- [duckdb-python v2.0.0 release](https://github.com/duckdb/duckdb-python/releases/tag/v2.0.0)
- [MotherDuck: If It Quacks Like a Duck](https://motherduck.com/blog/duckdb-client-server/)
- [Definite: Using DuckDB Quack as the DuckLake catalog](https://www.definite.app/blog/duckdb-quack-ducklake-catalog)
- [InfoQ: DuckDB Quack protocol (May 2026)](https://infoq.com/news/2026/05/duckdb-quack-protocol)
- [InfoQ: DuckDB v2 preview (Aug 2026)](https://infoq.com/news/2026/08/duckdb-v2-distributed)
- [The Stack: server mode, triggers and SQL](https://www.thestack.technology/server-mode-triggers-and-sql-duckdb-teases-v2-0/)
- [AlternativeTo: DuckDB 2.0 turns into a full-fledged server](https://alternativeto.net/news/2026/10/duckdb-s-major-2-0-release-turns-the-database-management-system-into-a-full-fledged-server/)
- [I Programmer: DuckDB Adds Quack Protocol](https://www.i-programmer.info/news/84-database/18914-duckdb-adds-quack-protocol.html)
- [RuntimeWire: DuckDB previews v2.0 server mode](https://runtimewire.com/article/duckdb-v2-server-mode-embedded-analytics)
- [GizmoData quack-jdbc](https://github.com/gizmodata/quack-jdbc)
- [DuckLake discussion #1315 (multi-tenant Quack catalog)](https://github.com/duckdb/ducklake/discussions/1315)
