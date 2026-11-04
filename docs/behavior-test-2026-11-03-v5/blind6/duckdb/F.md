# DuckDB 2.0 / Quack quiz answers (from memory, no lookups)

1. **Server and client roles.** Both roles are ordinary DuckDB processes running the `quack` extension. The server is the DuckDB instance that owns the database and calls `quack_serve(...)` (or `duckdb file.duckdb -serve`). It listens on localhost:9494 by default. The client is any other DuckDB (CLI, Python, Wasm, or a third-party JDBC/ADBC driver) that runs `ATTACH 'quack:host' AS qk (TOKEN '...')`. Attaching alone mounts the remote as a catalog: you qualify names like `qk.events`, filters and DDL/DML are pushed to the server, transactions are forwarded, and `qk.query('...')` sends a whole query. `CONNECT qk` routes the entire session to the remote, so unqualified SQL runs on the server and the results stream back until `DISCONNECT`. `CONNECT` also accepts Postgres and MySQL connection strings, which use remote pushdown.

2. **How data moves.** Quack runs over plain HTTP(S) as request/response, and the client always starts each exchange; the server never pushes. Payloads use DuckDB's own internal binary serialization (the one the WAL uses) under a new `application/duckdb` MIME type. That means vectors and complex types such as nested types, DECIMAL and VARIANT go over the wire nearly as-is, with no Arrow conversion. After the connection/auth handshake, a query needs one round trip, and small results come back inline (24,576 rows by default). Large results stream through follow-up FETCH requests, which can run in parallel across threads in batches of about 32 MiB.

3. **Default protection and the auth hooks.**
   - **Defaults:** every connection needs a shared token, which is randomly generated if you don't set one. The server binds only to localhost unless you pass `allow_other_hostname`. Non-local servers default to HTTPS with a self-signed certificate.
   - **What the defaults don't cover:**
     - There are no users or roles.
     - Anyone holding the token gets full read and write access to everything the server session can see.
     - The default authorization allows every query.
     - Local traffic is unencrypted.
     - Self-signed certificates need fingerprint pinning.
     - There is no rate limiting or hardening for direct internet exposure.
   - **The two hooks:**
     - Authentication (`quack_authentication_function`, default `quack_check_token`) runs once per connection with (session id, client token, server token) and returns a BOOLEAN.
     - Authorization (`quack_authorization_function`, default `quack_nop_authorization`) runs before every query with (connection id, query text). It returns a BOOLEAN, or a VARCHAR of SQL to run instead.
     - Operators replace either one with `SET GLOBAL` pointing at a SQL macro or an extension UDF. Examples include a token table for several users or a read-only allowlist. Python connection-scoped UDFs aren't visible to the callbacks, and prefix-based "SELECT only" checks are not robust.

4. **Why HTTP plus its own format.**
   - **HTTP:** everyone already knows how to run it. It passes through load balancers, firewalls, reverse proxies, TLS termination, auth proxies and intrusion detection without a custom transport. It also lets DuckDB-Wasm in a browser speak Quack natively.
   - **Not Arrow Flight SQL:** the team wants to control its own serialization so it can add data types and protocol messages without depending on an external format. Flight also needs at least two round trips per query, which hurts small writes. In their benchmarks Flight was about 3.5x slower on bulk transfer and much slower on single-row inserts.
   - **Not the Postgres wire protocol:** in their tests it was far slower, with 60M rows taking about 158 s versus about 5 s for Quack. It also usually can't parallelize bulk reads, and it would force type translation to Postgres types.

5. **Why a server instead of shared-file writes.** DuckDB keeps a lot of live state in memory: the catalog, buffer manager, MVCC version info and transaction state. Coordinating that across independent processes writing one file would be very hard (the post calls it hard to synchronize). The engine already has full MVCC plus optimistic concurrency for many connections and threads inside one process. So the simplest fit is to keep a single process owning the file and let remote clients become extra connections into it over the network. Multi-writer then comes "for free" from the existing transaction machinery, and the team argues that machinery competes with Postgres on quite a few workloads.

6. **Before Quack, and how Quack compares.**
   - **Old model:** across processes, a database file could be opened by one read-write process or by many read-only processes; multi-process writes were not supported.
   - **Workarounds:**
     - custom RPC/HTTP services wrapping DuckDB
     - Arrow Flight SQL wrappers
     - MotherDuck's hosted service and protocol
     - putting the data in Postgres, or using pg_duckdb
     - DuckLake with a Postgres catalog for multi-writer lakehouse access
   - **Quack** is a self-hosted, single-node, open-source server for a live DuckDB database with shared-token auth.
   - **DuckLake** is a lakehouse table format: Parquet in object storage plus a SQL catalog database. It gives multi-engine, multi-writer access to files. A Quack server can now act as the DuckLake catalog.
   - **MotherDuck** is a managed cloud service. It adds what Quack lacks: per-user permissions and sharing, SSO, storage separated from compute, read scaling, time travel and zero-copy clones, serverless lifecycle and an SLA. It positions itself as complementary and plans to support Quack endpoints.

7. **How Quack was received.**
   - **Enthusiasm:**
     - Hacker News reaction was largely positive.
     - Ryan Glover called it "rad" and said it solved the "how do I horizontally scale this" problem.
     - A Redditor called reaching DuckDB remotely "like a normal DB" a big unlock.
     - MotherDuck's Mehdi Ouazza asked whether the "no multi-writer" objection can now be retired.
     - The ecosystem moved fast: standalone clients appeared within weeks (GizmoData quack-jdbc, an ADBC driver), DuckLake integration followed, and companies like Definite used Quack as a catalog. MotherDuck framed it as complementary.
     - The enthusiasm came from long-standing demand for shared and multi-writer access and from strong benchmark numbers.
   - **Skepticism:**
     - Some see a tension with DuckDB's "no server needed" simplicity, e.g. the LinkedIn post "spent 6 years telling you it doesn't need a server" and RuntimeWire on simplicity pressure.
     - Others pointed out that DuckDB is still an OLAP engine, not an OLTP replacement for Postgres.
     - Practical limits: basic shared-token auth, a single node with serialized writes, no HA or replication, beta/alpha maturity, a write-scaling ceiling around 8 threads, and operations left to the user.

8. **Sensor-ingestion setup.**
   - **Setup:**
     - Make one process the sole owner of the database file and run `quack_serve` there, with a strong token (or a token-table authentication callback with one token per analyst) and the default localhost binding. The ingestion service itself can be the server, or a dedicated `duckdb sensors.duckdb -serve` process with ingestion as a local Quack client.
     - The dashboard on the same VM attaches to `quack:localhost`.
     - The analysts attach (or `CONNECT`) through a TLS reverse proxy at `quack:host:443`, and their `UPDATE`s run on the server inside its transactions.
     - Optionally use an authorization callback to limit what analysts can run, and enable reconnects for flaky laptop connections.
   - **Concurrent edits:** ingestion appends never conflict with each other. Analyst updates use DuckDB's MVCC with optimistic concurrency, so if two analysts update the same row at the same time, the first commit wins and the second transaction fails with a conflict error and must be retried. It is not silently merged or blocked.
   - **Changes from the defaults before internet exposure:**
     - Don't just flip `allow_other_hostname` (it's meant for testing). Keep Quack on localhost behind nginx or Caddy with a real certificate, such as Let's Encrypt.
     - Proxy settings: raise the body size limit to 256M, raise proxy timeouts (e.g. 600 s), use HTTP/1.1 keep-alive upstream, and turn off buffering (`flush_interval -1` in Caddy).
     - Firewall so only 443 is reachable.
     - Use strong, per-user tokens rather than the one shared token, plus authorization rules if needed.
     - Plan for backups and process supervision, since there is no HA.
     - Pin the 2.0 storage version or upgrade every client together.
