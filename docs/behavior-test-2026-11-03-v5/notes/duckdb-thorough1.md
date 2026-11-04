# DuckDB 2.0 and its server mode (Quack)

Research date: 2026-11-03. Based on about 40 web searches and page reads: official duckdb.org blog posts and docs, the 2.0 migration guide, GitHub release notes, InfoQ, The Stack, AlternativeTo, Definite, Columnar, and a Hacker News thread. Sources are listed at the end.

## TL;DR

- **DuckDB 2.0.0 LTS "Cinnamon Teal"** was released on **2026-10-21**. Its dev branch name and GitHub release title is "Cyanoptera", which is the cinnamon teal's species name. It is built from about 20,000 commits since 1.5, and the team calls it "DuckDB's biggest feature release so far". 1.0 was about stability, while 2.0 is about features.
- **Headline feature: server mode.** Any DuckDB process can now serve its databases over the network using **Quack**, DuckDB's own HTTP-based client/server protocol. Other DuckDB instances `ATTACH 'quack:host'` and `CONNECT` to it. Hannes Mühleisen (DuckDB Labs CEO) said: "This release kicks off the year of DuckDB as a server."
- **Quack is still beta.** It first shipped in 1.5.3 (May 2026). 2.0 brings a much-improved Quack 1.0 extension, but the official FAQ still says it is **not yet production-ready**. Expect breaking changes to the protocol, function names and defaults.
- **Other big items:**
  - async I/O, on by default (about 3x faster S3 Parquet scans and about 19x faster S3 CSV scans in their benchmark)
  - fully shredded `VARIANT` type
  - a new PEG SQL parser, now the default
  - triggers
  - C API v2
  - signed self-hosted extension repositories
  - a CLI "agent mode" for AI coding agents
  - ICU removed from core
  - a **new default storage format that older versions cannot read**
- **Support:** 2.0 is LTS until at least **Oct 2027**. 1.5 support was extended to March 2027, and 1.4 LTS ends 2026-11-17. **2.0.1 is scheduled for 2026-11-18.** For mission-critical systems, the team suggests waiting for that patch.

---

## 1. Why a server mode, and why now

DuckDB has always been in-process, like SQLite but for analytics. Its biggest limitation is that **only one process can write to a database file**, because much of its state lives in memory. Users worked around this in several ways:
- custom RPC services
- Arrow Flight SQL wrappers such as GizmoSQL and the `airport` extension
- MotherDuck's proprietary protocol
- running DuckDB inside Postgres with pg_duckdb
- DuckLake, which gives multiple writers via a catalog database plus Parquet

Inside one process, DuckDB has always had full MVCC, multiple connections and transaction isolation. Single-user embedded use just never exposed this. Quack exposes it over the network, so one server process holds the mutable state and many clients read and write concurrently. The team also says DuckDB is now fast enough to compete with PostgreSQL on "quite a few workloads", but gives no benchmark for that.

The team says the in-process model stays first-class. Server mode is an addition, not a pivot.

## 2. Timeline

| Date | Event |
|---|---|
| 2026-03-09 | DuckDB 1.5.0 "Variegata" |
| 2026-05-12 | Quack announced ("Quack: The DuckDB Client-Server Protocol"); introduced by Hannes Mühleisen at AI Council 2026 |
| 2026-05-20 | 1.5.3 ships Quack as an autoloadable core extension (beta) and as a DuckLake catalog option |
| 2026-06-24 | DuckCon #7 |
| 2026-07-31 | Async I/O blog post |
| 2026-08-17 | "A Preview of DuckDB v2.0" |
| 2026-09-02 | 2.0 alpha; `v2.0-cyanoptera` branch cut (feature freeze) |
| 2026-10-09 | Agent mode blog post |
| 2026-10-21 | **2.0.0 LTS released** |
| 2026-11-18 | 2.0.1 planned (feature freeze 11-04, merge freeze 11-11) |
| 2027 H1 | 2.1 "Islandica" (drops single-arrow lambdas and osx_amd64) |

## 3. Quack: how server mode works

### 3.1 Protocol design
- **Transport:** HTTP, using HTTP/2 according to the FAQ. Every request is a POST to `/quack`. Choosing HTTP means it works with existing load balancers, firewalls, proxies, auth and IDS tooling. It also lets **DuckDB-Wasm in a browser speak Quack natively**.
- **Client-driven request/response:** the server never pushes. Message types include a connection or auth request, `PREPARE` (execute), `FETCH` (pull more results) and `APPEND`.
- **Serialization:** a new MIME type, `application/duckdb`, built on the serialization primitives DuckDB already uses for its write-ahead log. Internal data blocks are sent as-is, so nested, decimal and interval types are lossless and no format conversion is needed.
- **One round trip per query** after the handshake. Small results come back inline in the `PREPARE` response (`quack_prepare_inline_rows` defaults to 24,576). Large results stream back through follow-up `FETCH` requests that can run in parallel threads.
- **Default port:** 9494. The URI scheme is `quack:`, for example `quack:localhost`, `quack:myhost:9000` or `quack:[::1]:1234`.
- **Why not Arrow Flight SQL?** The team didn't want DuckDB's internals tied to an externally controlled format ("cannot allow ourselves to be restricted by formats that are controlled externally"). Flight SQL also needs at least two round trips per query (`CommandStatementQuery` plus `DoGet`), which hurts small writes.
- **Why not the Postgres wire protocol?** It is row-based and slow for bulk transfer. See the benchmarks below.
- **License:** MIT. Code is in `github.com/duckdb/duckdb-quack`. It has no third-party dependencies, unlike the Flight-based options.

### 3.2 Starting a server

Starting from SQL:
```sql
INSTALL quack; LOAD quack;            -- optional, it autoloads
CALL quack_serve(token = 'my_token'); -- defaults to quack:localhost:9494
-- or bind publicly (testing only):
CALL quack_serve('quack:0.0.0.0:9494', allow_other_hostname = true);
CALL quack_stop('quack:localhost');
```

`quack_serve` returns the listen URI, the HTTP URL and the `auth_token`. Its options:

| Option | Default | Notes |
|---|---|---|
| `token` | random | Minimum 4 characters |
| `secret` | none | A named `quack` secret to take the token from |
| `create_secret_if_not_exists` | false | Persists the token as secret `__default_quack` |
| `allow_other_hostname` | false | Required to bind a non-localhost address |
| `disable_ssl` | true for local, false otherwise | |
| `ssl_cert_file`, `ssl_key_file` | self-signed generated | |

Starting from the CLI:
```bash
duckdb analytics.duckdb -serve        # serve a persistent DB on localhost:9494
duckdb -serve                         # serve an in-memory DB
duckdb -connect                       # connect to localhost:9494 with the default quack secret
duckdb -connect quack:analytics       # use a named secret
```

`-serve` creates the default secret persistently, so serving and connecting on one machine needs no setup. `-connect` cannot be combined with a local DB path. The `.serve_command` and `.connect_command` dot commands customize what these flags run, for example in `~/.duckdbrc`.

### 3.3 Connecting as a client

There are three ways to connect.

**1. `ATTACH` + `CONNECT` (recommended)** routes the whole session to the server:
```sql
ATTACH 'quack:server.example.com' AS qk (TOKEN 'my_token');
CONNECT qk;
SELECT count(*) FROM events;  -- executes on the server, results stream back
DISCONNECT;
```

**2. `ATTACH` only** mounts the remote as a catalog and uses qualified names. Remote DDL, writes and transactions are forwarded:
```sql
CREATE SECRET (TYPE quack, TOKEN 'my_token', SCOPE 'quack:localhost');
ATTACH 'quack:localhost' AS remote;
FROM remote.hello;
CREATE TABLE remote.hello2 AS FROM VALUES ('world2') v(s);
FROM remote.query('SELECT ...');   -- ship arbitrary SQL to the server
```

**3. Stateless one-off queries** with no catalog import:
```sql
FROM quack_query('quack:host', 'SELECT 42', token = '...');
```

Useful client options:
- `SSL_FINGERPRINT`, for pinning a self-signed server certificate
- `CLIENT_ID`
- `HEARTBEAT_TIMEOUT`, a lease that defaults to 60 s
- `DISABLE_SSL`

Clients use plain HTTP for localhost, 127.0.0.1 and ::1, and HTTPS for everything else. A `quack` secret can also carry `EXTRA_HTTP_HEADERS`, which is useful behind auth proxies.

Other client and operator functions:
- `quack_connection_id(catalog)` and `quack_cancel(catalog, id)` for query cancellation
- `quack_query_by_name`
- `quack_clear_cache()`
- `quack_server_list()` and `quack_active_connections()` for operators
- `whoami()` and `quack_identify(name, provider, region, meta)` for node identity

### 3.4 `CONNECT` is not only for DuckDB
`CONNECT` / `DISCONNECT` are new SQL statements in 2.0. Internally, `CONNECT` is an "ephemeral attach" (PR #23555). It also accepts Postgres and MySQL connection strings, and a new **remote pushdown optimizer** (PR #22914) sends the SQL to that server instead of pulling the tables over the wire:
```sql
CONNECT 'postgres:dbname=mydb host=localhost';
SELECT count(*) FROM orders;  -- runs on PostgreSQL
DISCONNECT;
```
Local DuckDB databases, whether in-memory or file-based, cannot be `CONNECT`ed. Running `DISCONNECT` with no active connection raises an error.

### 3.5 Security model
- **The default posture is conservative.** The server binds to localhost only, and a random token is generated if you don't set one. The default authorization allows **every** query, so a token holder has the full SQL surface. That includes reading files the server can see and loading extensions.
- **TLS:** local connections use no TLS. Non-local connections use HTTPS with a self-signed certificate unless you supply one (`quack_generate_keys(dir)` can create one). The official guidance is to **keep Quack on localhost and put a TLS-terminating reverse proxy in front**, such as nginx or Caddy with Let's Encrypt. `allow_other_hostname` is described as "only meant for testing".
- **Pluggable auth via SQL.** Both settings below are global, so set them with `SET GLOBAL`. A plain `RESET` leaves a stale global value.
  - `quack_authentication_function` (default `quack_check_token`) has signature `(session_id, client_token, server_token) -> BOOLEAN` and is called on connect.
  - `quack_authorization_function` (default `quack_nop_authorization`) has signature `(connection_id, query)`. It is called on each `PREPARE` and may return `BOOLEAN`, or a **VARCHAR that is executed instead of the client's query**, which allows query rewriting.
  - The callbacks can be SQL macros, for example a lookup in a `quack_tokens` table for multi-user tokens. The docs show a read-only regex macro but warn it is bypassable (a `WITH ... INSERT` passes it). Real read-only enforcement means attaching the DB read-only or inspecting the parsed statement type.
  - Macros can't do DML or keep state, so logging, per-user sessions and ACLs need a C++, Rust, C or Go extension UDF. **Python UDFs don't work** as callbacks, because they are not visible to the callback connection.
- **There are no built-in users, roles or GRANTs.** Multi-user access control is something you build with these callbacks.

### 3.6 Reverse proxy gotchas (from the official guide)
- Turn **off response buffering**: `proxy_buffering off;` in nginx, `flush_interval -1` in Caddy. Otherwise streamed FETCH results stall and memory grows.
- Raise the body limit to about 256 MB (`client_max_body_size 256M;`). nginx's 1 MiB default breaks large INSERT and APPEND requests partway through.
- Raise timeouts (`proxy_read_timeout 600s;`), because nginx's 60 s default kills long queries.
- Keep upstream keep-alive on (`proxy_http_version 1.1; proxy_set_header Connection "";`). Server-side connection state depends on persistent connections.
- The client must specify `:443` explicitly (`ATTACH 'quack:quack.example.com:443'`), because the default port stays 9494 even over HTTPS.

### 3.7 Tuning and limits (reference page)

| Setting | Default | Notes |
|---|---|---|
| `quack_server_max_connections` | 1024 | Extra connections are refused |
| `quack_server_keep_alive_timeout` | 300 s | |
| `quack_target_batch_bytes` | 32 MiB | |
| `quack_server_stream_buffer_bytes` | 256 MiB | Per stream, at most ¼ of `memory_limit` |
| `quack_fetch_read_ahead` | 0 | 0 means the number of async threads |
| `quack_enable_reconnects` | false | When on, the server caches the last result until the client acks it |
| `quack_result_ttl` | 3600 s | Reconnect result cache |
| `quack_cache_max_rows` | 100k | Reconnect result cache |

Logging:
- `CALL enable_logging('Quack')` logs every message, with `quack_connection_id`, `client_query_id` and `duration_ms`. Read it with `duckdb_logs_parsed('Quack')`.
- `enable_logging('HTTP')` logs the transport.
- 2.0 also reworked the metrics and observability layer (PR #22799) for long-running servers.

### 3.8 Performance (DuckDB's own benchmark, May 2026, Quack beta)
The setup was two AWS m8g.2xlarge instances (8 vCPU, 32 GB, up to 15 Gbps) in the same AZ, with about 0.28 ms ping.

Bulk transfer of TPC-H lineitem (lower is better):

| Rows | Quack | Arrow Flight | PostgreSQL |
|---|---|---|---|
| 1M | 0.24 s | 0.38 s | 2.20 s |
| 10M | 0.89 s | 2.90 s | 25.64 s |
| 60M | 4.94 s | 17.40 s | 158.37 s |

Single-row INSERT transactions per second (higher is better):

| Threads | Quack | Arrow Flight | PostgreSQL |
|---|---|---|---|
| 1 | 1,038 | 469 | 839 |
| 8 | 5,434 | 1,358 | 4,320 |

- **About 3.5x faster than Flight on bulk transfer, and about 30x faster than Postgres.**
- Write throughput **plateaus around 5,500 tx/s**. The cause is DuckDB's limit on concurrent inserts into one table, and Postgres scales better beyond that. Raising this ceiling is on the roadmap.
- 2.0 claims "higher query throughput in both directions" over the 1.5.3 version but publishes no new numbers.
- HN critics noted that these are vendor benchmarks run near the link limit (about 4.85 Gbps effective), and that the "76 GB CSV" framing overstates the data size. **I found no independent benchmarks.**

### 3.9 What Quack is not (yet)
- **Not distributed query processing.** The FAQ says "Not currently". One server process does all the compute.
- **No replication or HA.** The server is a single point of failure. A replication protocol for read replicas is on the roadmap.
- **Not a managed service.** There are no backups, user management or "dual execution". That is what MotherDuck does. MotherDuck uses its own non-HTTP protocol, although the Quack team drew on MotherDuck's experience.
- **Write scale is moderate.** "A few thousand writes per second" on 8 CPU / 32 GB, and storage scales to "a few terabytes" (native DuckDB file). For petabyte scale or open formats, use DuckLake.
- **ATTACH binds every remote table up front.** One table that fails to bind breaks the whole attach. A classic example is a lazily resolved Iceberg catalog on the server, which fails with "Type with name UNKNOWN does not exist". Fix it with `TABLE_RESOLUTION 'eager'` on the server, or use `quack_query()`.
- **Clients:** officially, "a DuckDB with the quack extension" in any language binding, including CLI, Python and Wasm. There is no standalone wire spec published yet. Third parties have already built clients, including a Columnar ADBC driver preview (`dbc install --pre quack`, which embeds DuckDB) and community JDBC, ADBC and TypeScript SDKs.

## 4. Quack + DuckLake
- DuckLake can use a **Quack-served DuckDB as its catalog database** instead of Postgres or SQLite. This shipped experimentally in 1.5.3.
- Benefits: no type-translation mismatches, no Postgres connection-limit exhaustion, server-side retries, and remote clients don't need object-store credentials.
- DuckDB/DuckLake developer pdet predicts it will become the main catalog option.
- Definite's field report:
  - 8 concurrent writers inserted 2M rows with zero errors at about 222k rows/s.
  - Most of the speedup came from keeping the catalog warm in one long-lived process.
  - Concurrent `CREATE TABLE IF NOT EXISTS` raced, so pre-create tables before starting parallel writers.
  - The server is still a single point of failure.
- Community: `quacklake`, a DuckLake catalog served via Quack on Cloudflare (alpha).

## 5. Everything else in 2.0

**Performance**
- **Async I/O, on by default.** There are separate `REGULAR` (compute) and `ASYNC` (blocking I/O, 4x threads up to 256) pools, plus a read-ahead queue bounded by `read_ahead_depth` (default -1, memory-governed). S3 TPC-H Q6 SF100 benchmark:
  - single Parquet file: 8.23 s to 2.84 s (about 2.9x), or 2.23 s tuned (about 3.7x, saturating 25 Gbit)
  - CSV: 877 s to 45 s (about 19x)
  - concurrent queries: 35.8 s to 15.6 s
  - It now also covers the native format and async Parquet writes.
- Other gains: partial aggregate pushdown, a rewritten recursive CTE engine (about 40x on one microbenchmark), aggregation spilling, bloom and zone-map pruning, and partition-aware planning.

**Types and storage**
- **VARIANT** is shredded end to end (storage, execution, Parquet read and write), with `variant_*` functions. The `JSON` type is expected to be backed by VARIANT soon after 2.0.
- **Storage format v2.0** (storage version 69) adds lazy column metadata, `DICT_FSST` by default, compact deletes, better corruption checks and ART checkpoint vacuuming.

**Extensibility**
- **New PEG parser, now the default.** Extensions can add syntax, and error locations are precise. You can revert with the `legacy_parser` extension plus `SET enable_legacy_parser = true;`.
- **C API v2** is versioned, YAML-specified and aims for a stable ABI so extensions can be built once. It is **not yet marked stable**.
- **Extension repositories:** `CREATE EXTENSION REPOSITORY ... WITH PREFIX ...` with pinned RSA keys, for self-hosted signed extensions. This is work in progress and requires `SET allow_extension_repositories = 'allowed'`.
- **Core extensions:** DuckLake 1.1; Iceberg REST catalog improvements; Delta `CREATE TABLE`, time travel and VARIANT.

**SQL**
- Triggers: `BEFORE`/`AFTER`, row and statement level, transition tables, `RETURNING`. Some combinations are unsupported, for example `BEFORE ... FOR EACH ROW`.
- `$var` variables, DML inside CTEs (including `COPY`), `NEAREST` joins (top-k similarity), nested schemas, and `json_set`, `json_insert`, `json_replace`, `json_remove`.
- `FETCH FIRST`, `UNNEST` in `GROUP BY`, `OVERLAY()`, and `dialect_compatibility_mode='spark'`.

**CLI agent mode**
- Auto-detected via `CLAUDECODE`, `CODEX_CI`, `CURSOR_AGENT`, `GEMINI_CLI` and similar variables when stdout isn't a TTY. Force it with `-agent`, `-no-agent` or `DUCKDB_AGENT_MODE`.
- Output: compact Markdown tables with types in the headers, explicit truncation (first and last 20 rows plus a hash), JSON errors on stderr, and cost estimates for queries over 1M rows.
- Result: 59% fewer output tokens read (123.6k to 50.8k) across 132 correct TPC-H runs with Claude Code. Total cost and runtime didn't measurably change.

**Other**
- **ICU removed:** timezones, calendars and collations are implemented natively. Timezone data is about 45 kB, with 2.2 to 2.6x speedups.

## 6. Breaking changes and migration checklist
1. **Storage format:** files written by 2.0 can't be opened by older versions. 2.0 still reads and writes old files. For mixed fleets, pin the format with `-storage-version v1.5.6`, `storage_compatibility_version` or `ATTACH ... (STORAGE_VERSION ...)`. To convert, `EXPORT DATABASE` with the old binary and `IMPORT DATABASE` with the new one, or use `COPY FROM DATABASE`.
2. **Lambdas:** `x -> x + 1` is disabled by default. Use `lambda x: x + 1`. `SET lambda_syntax = 'enable_single_arrow'` is a temporary bridge, removed in 2.1.
3. **Parser semantics:**
   - `~` is now partial regex match. Restore the old behavior with `SET regex_match_operator_semantics='full'`.
   - `2 * 3!` now equals 12.
   - `X'..'` and `B'..'` now produce BLOB and BITSTRING.
4. **Any error in an explicit transaction now aborts it.** The old behavior is available via `default_transaction_invalidation_policy`.
5. Integer `//` and `%` by zero now raise an error. Use `TRY()` or `error_on_division_by_zero=false`.
6. `MERGE INTO` now errors when multiple source rows match one target row.
7. An untyped `NULL` column stays type `NULL` instead of INTEGER. Cast it explicitly, or set `legacy_disable_null_type`.
8. `preserve_identifier_case` is now a VARCHAR enum.
9. **S3 URL query-string credentials are no longer parsed.** Use secrets instead (temporary escape hatch: `s3_enable_deprecated_url_parameters`).
10. Removed settings:
    - `enable_http_logging` (use `enable_logging('HTTP')`)
    - `custom_profiling_settings` (use `tracked_metrics`)
    - `profiling_mode` (now a no-op)
11. CLI log storage defaults to a non-queryable store. Use `CALL enable_logging(storage='memory')` to query logs.
12. **Python client 2.0:**
    - Requires Python 3.11 or newer, with abi3 wheels for 3.12+. No free-threaded builds.
    - `execute()` results now stream.
    - Arrow and Polars helpers were removed from `execute()` results. Use `con.sql(q).to_arrow_table()`.
    - A bare string in a relation expression is now a **value**, not a column. Use `col()` for columns.
    - `import duckdb` no longer opens the default DB.
    - Default Arrow batch size is now 131,072 rows.
    - Moved to nanobind. New Appender API.
13. Build and platform changes:
    - C++17 is required, and Clang is the default compiler.
    - Amalgamation builds are deprecated.
    - The Julia client moved to DuckDB.jl.
    - **osx_amd64 is dropped in 2.1.**
14. At launch, the CLI and Python were available first. Go, R, Java and some extensions lagged by a few days.

## 7. Practical take: when to use server mode today
**Good fits:**
- one process ingesting telemetry while dashboards or notebooks query it
- a shared high-memory analytics box for a small team or homelab
- running compute next to the data while viewing results from a laptop
- a DuckLake catalog server
- browser (Wasm) clients querying a backend DuckDB
- replacing hand-rolled Flight, gRPC or REST wrappers around DuckDB

**Not yet a fit:**
- anything needing HA, replication, real user management or audit logging without custom extension code
- high-concurrency OLTP above about 5k tx/s
- distributed or scale-out compute
- direct internet exposure

**Deployment checklist:**
- Run `FORCE INSTALL quack;` on all nodes so client and server extension versions match.
- Bind to localhost and front the server with nginx or Caddy for TLS. Disable buffering, raise body limits and timeouts, and specify `:443` on the client.
- Set a strong token in a secret, plus a custom authorization function if any non-admin users connect.
- Pre-create tables before parallel writers start.
- Watch `quack_active_connections()` and the Quack logs.
- Treat the protocol as unstable until DuckDB declares it stable. 2.0.1 lands on 2026-11-18.

**Positioning (community view):**
- Quack moves DuckDB toward a classic RDBMS footprint while keeping the embedded mode.
- It competes with Flight SQL wrappers such as GizmoSQL and airport, and partly overlaps MotherDuck's self-hosted use cases.
- Community users stress that DuckDB is still an **OLAP engine, not a Postgres replacement for transactional apps**.

## Sources
- Announcing DuckDB 2.0.0 LTS "Cinnamon Teal": https://duckdb.org/2026/10/21/announcing-duckdb-200
- A Preview of DuckDB v2.0: https://duckdb.org/2026/08/17/duckdb-20-highlights
- Try DuckDB v2.0-dev: https://duckdb.org/2026/09/02/try-duckdb-20-alpha
- Quack: The DuckDB Client-Server Protocol: https://duckdb.org/2026/05/12/quack-remote-protocol.html
- DuckDB 1.5.3 announcement: https://duckdb.org/2026/05/20/announcing-duckdb-153
- Quack landing page: https://duckdb.org/quack/ and FAQ: https://duckdb.org/quack/faq.html
- Quack docs: https://duckdb.org/docs/2.0/quack/overview.html, /reference.html, /security.html, /troubleshooting.html, /setup/reverse_proxy.html
- CONNECT statement: https://duckdb.org/docs/2.0/sql/statements/connect.html
- CLI arguments: https://duckdb.org/docs/2.0/clients/cli/arguments.html
- Migration guide: https://duckdb.org/docs/2.0/operations_manual/migration
- Release calendar: https://duckdb.org/release_calendar.html
- Async I/O post: https://duckdb.org/2026/07/31/asynchronous-io
- Agent mode post: https://duckdb.org/2026/10/09/agent-mode
- duckdb-quack repo: https://github.com/duckdb/duckdb-quack
- DuckDB v2.0.0 release notes: https://github.com/duckdb/duckdb/releases/tag/v2.0.0
- duckdb-python v2.0.0: https://github.com/duckdb/duckdb-python/releases/tag/v2.0.0
- InfoQ on Quack: https://infoq.com/news/2026/05/duckdb-quack-protocol
- InfoQ on the v2 preview: https://infoq.com/news/2026/08/duckdb-v2-distributed
- The Stack: https://www.thestack.technology/server-mode-triggers-and-sql-duckdb-teases-v2-0/
- AlternativeTo: https://alternativeto.net/news/2026/10/duckdb-s-major-2-0-release-turns-the-database-management-system-into-a-full-fledged-server/
- Definite, Quack as DuckLake catalog: https://www.definite.app/blog/duckdb-quack-ducklake-catalog
- Columnar, Quack ADBC driver: https://columnar.tech/blog/announcing-quack-adbc-driver/
- HN thread mirror: https://hackernews-master.vercel.app/item/48111765
- I-Programmer on Quack and Wasm: https://www.i-programmer.info/news/84-database/18914-duckdb-adds-quack-protocol.html
