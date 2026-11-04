# DuckDB / Quack quiz answers (from memory, no lookups)

**1. Server vs client roles.**
- **Roles:** Both ends are ordinary DuckDB processes with the `quack` extension loaded. The server is the process that owns the database and has called `quack_serve(...)` (or was started with `duckdb -serve`). It listens on `quack:localhost:9494` by default. Clients are any other DuckDB instances (CLI, Python, Wasm and so on) that point at it with a `quack:` URI and a token.
- **Attach as a catalog:** `ATTACH 'quack:host' AS qk` mounts the remote as a catalog alongside the client's local databases. You use qualified names like `qk.events`, and remote DDL, writes and transactions are forwarded. The client binds all the remote table definitions locally.
- **`CONNECT`:** `CONNECT qk` (new SQL statement in 2.0) routes the whole session to that remote. Unqualified queries then execute on the server and results stream back, until you run `DISCONNECT`.
- **One-off queries:** there is also stateless `quack_query(uri, sql)` and the `qk.query('...')` macro.

**2. How data moves.**
- **Transport:** plain HTTP, which the FAQ says is HTTP/2. Requests are POSTs to `/quack`. Local addresses use HTTP and non-local ones default to HTTPS.
- **Who initiates:** always the client, in a request/response pattern. The server never pushes.
- **Encoding:** payloads use DuckDB's own serialization, MIME type `application/duckdb`. It is built on the same serialization primitives as the write-ahead log, so DuckDB's internal data blocks and types travel losslessly with no conversion.
- **Round trips:** after the connect/auth handshake, a query needs one round trip. Small results come back inline in the `PREPARE` response. Large results are pulled with follow-up `FETCH` requests, which can run in parallel on several threads.

**3. Default protection and the auth hooks.**
- **What protects it:** the server binds only to localhost unless `allow_other_hostname = true`. Every connection must present a token, which is randomly generated if you don't set one. Non-local connections default to HTTPS with a self-signed certificate.
- **What it does not protect against:** the default authorization is a no-op, so anyone holding the token can run any SQL the server can run. There are no users or roles, and the team says not to expose it directly to the internet.
- **Authentication hook:** `quack_authentication_function` (default `quack_check_token`) is called once per connection with `(session_id, client_token, server_token)` and returns a boolean.
- **Authorization hook:** `quack_authorization_function` (default `quack_nop_authorization`) is called per query with `(connection_id, query)`. It returns a boolean, or a replacement SQL string to run instead.
- **Customizing them:** operators set both with `SET GLOBAL` to a SQL macro, for example a token-table lookup or a read-only check. Logic that needs state or logging goes in an extension UDF written in C++, Rust, C or Go. Python UDFs don't work as callbacks.

**4. Why HTTP plus its own format.**
- **Why HTTP:** existing infrastructure (load balancers, proxies, firewalls, TLS, auth, intrusion detection) works with it unchanged. It also lets DuckDB-Wasm in a browser speak Quack natively.
- **Why not Arrow Flight SQL:**
  - The team didn't want DuckDB's internals and protocol evolution tied to an externally controlled format.
  - Flight SQL needs at least two round trips per query (`CommandStatementQuery` plus `DoGet`), which hurts small writes.
  - Their own block format avoids conversion. In their benchmark Quack was about 3.5x faster than Flight on bulk transfer.
- **Why not the PostgreSQL wire protocol:** it is row-based and its clients don't parallelize reads, so bulk transfers are dramatically slower. Their benchmark had 60M rows taking about 158 s over Postgres versus about 5 s over Quack.

**5. Why a server process instead of shared-file writers.**
- **The problem with shared files:** DuckDB keeps much of its database state in main memory (catalog, buffers, transaction/MVCC information). Several independent processes writing one file would each have to keep that in-memory state synchronized, which is hard and slow.
- **The server answer:** one process holds all the mutable state and every other process becomes a client of it.
- **Why it fits the existing engine:** within one process DuckDB has always been a fully transactional, multi-connection database with MVCC and isolation. Quack doesn't need a new concurrency design. It exposes the engine's existing in-process concurrency to remote connections.

**6. Before Quack, and how it compares.**
- **Before Quack:** DuckDB was in-process. Only one process could open a database file for writing; several processes could open it read-only, and the writer lock excluded others.
- **Workarounds people used:**
  - custom RPC, REST or gRPC wrappers
  - Arrow Flight SQL servers such as GizmoSQL and the `airport` extension
  - pg_duckdb (DuckDB inside Postgres)
  - MotherDuck's hosted service
  - DuckLake, for multiple writers via a catalog database plus Parquet
  - hand-rolled WAL and snapshot tricks
- **Versus DuckLake:** DuckLake is an open lakehouse format. It stores Parquet on object storage at petabyte scale, needs a catalog database, and other engines can use it. Quack is DuckDB-only, uses the native storage file, and scales to a few TB. The two combine: a Quack server can act as the DuckLake catalog.
- **Versus MotherDuck:** MotherDuck is a managed commercial cloud service with its own non-HTTP protocol, user management, backups and dual execution. Quack is a self-hosted, MIT-licensed protocol that has none of those managed features.

**7. Reception.**
- **Enthusiasm:** practitioners on HN and Reddit were largely enthusiastic. Typical comments were "exactly what I needed", "such a big unlock", and that it solves "how do I horizontally scale this". People cited concrete uses:
  - ingestion running alongside the UI or dashboards
  - shared team or homelab analytics boxes
  - the DuckLake catalog (DuckLake developers predict it becomes the main catalog option)
  - replacing home-made wrappers
- **Ecosystem:** third parties shipped clients and drivers within weeks, including Columnar's ADBC driver and community JDBC drivers and a TypeScript SDK. A MotherDuck developer asked whether the "no multi-writer" objection could now be retired.
- **Skepticism:**
  - doubts about HTTP for bulk streaming
  - vendor-run benchmarks near the link limit, and the misleading "76 GB CSV" framing
  - the unexplained meaning of "concurrent writers"
  - confusion about DuckDB's scope ("is it SQLite or a server?")
  - reminders that it is beta, has no HA or replication, has write throughput capped around 5.5k tx/s, and is still an OLAP engine rather than a Postgres replacement.

**8. Sensor-ingestion scenario.**

*Setup*
- **Make the ingestion service the Quack server.** It already owns the file, and only one process can write it. Have it run `quack_serve` on localhost after opening the database (or start it with `duckdb sensors.duckdb -serve`), with a strong token.
- **Dashboard:** it attaches/`CONNECT`s to `quack:localhost` over plain local HTTP instead of opening the file itself.
- **Analysts:** they `ATTACH 'quack:their.domain:443' AS s (TOKEN ...)` and `CONNECT s`, or use `ATTACH` with qualified names. Their `UPDATE`s run inside the server's transactions alongside ingestion.

*Two analysts correcting the same reading at once*
- DuckDB's MVCC uses optimistic concurrency, so the two transactions don't block each other.
- The second one to update the same row should get a transaction/write-write conflict error and must retry. Since 2.0, any error inside an explicit transaction aborts it until `ROLLBACK`.
- I'm confident about this from general DuckDB knowledge, but I didn't see it stated specifically for Quack in my sources.

*Changes needed before internet exposure*
- **Front it with a reverse proxy:** keep Quack bound to localhost (don't use `allow_other_hostname`, which is meant for testing). Put nginx or Caddy in front to terminate TLS with a real certificate such as Let's Encrypt. In the proxy:
  - disable response buffering (`proxy_buffering off` / `flush_interval -1`)
  - raise the body-size limit (about 256 MB) and the read/send timeouts
  - keep upstream keep-alive
- **Client port:** analysts must specify `:443` explicitly in the URI, because the default port stays 9494.
- **Auth:**
  - Replace the single shared token with per-analyst tokens via a custom authentication function.
  - Add an authorization function. The default allows any SQL, including file reads and extension loads. You might allow only SELECT plus UPDATE on the readings table, for example; a regex prefix check is bypassable, so real enforcement needs statement-type inspection or an extension UDF.
- **Version and stability:** run `FORCE INSTALL quack` so all ends match. Treat the setup as beta: the protocol may change, and there is no HA or replication.
