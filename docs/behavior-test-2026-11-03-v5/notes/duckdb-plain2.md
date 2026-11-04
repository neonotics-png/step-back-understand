# DuckDB 2.0 and server mode (Quack)

Researched 2026-11-03 from DuckDB's official announcement and docs, plus press coverage. Sources are listed at the end.

## The short version

- **What shipped:** DuckDB 2.0.0 LTS, codename "Cinnamon Teal", was released on **October 21, 2026**. It is supported until at least October 2027.
- **Headline feature:** server mode. DuckDB was built to run inside an application, with no server. 2.0 lets one DuckDB process serve its databases over the network to other DuckDB processes.
- **How it works:** server mode runs on **Quack**, DuckDB's own network protocol, which ships as the `quack` extension. Quack first appeared as a beta in 1.5.3 (May 2026). 2.0 ships a polished version with performance and stability fixes.
- **The new SQL:** a `CONNECT` statement points your whole session at a remote database. It works with Quack servers, PostgreSQL and MySQL.
- **The main upgrade risk:** the new default storage format. **DuckDB versions before 2.0 cannot open database files that 2.0 creates.**

## Server mode, in plain terms

Before 2.0, a DuckDB database file belonged to the single process that had it open. To share it with other programs, you needed MotherDuck, an export step, or your own API layer.

With Quack, one DuckDB process becomes the server and other DuckDB processes connect to it as clients:

**Server:**
```sql
CALL quack_serve('quack:localhost', token = 'my_token');
```

**Client:**
```sql
ATTACH 'quack:server.example.com' AS qk (TOKEN 'my_token');
CONNECT qk;                    -- the session now runs against the server
SELECT count(*) FROM events;   -- runs on the server; results stream back
DISCONNECT;                    -- back to your local session
```

**From the command line:**
- `duckdb -serve` starts a Quack server.
- `duckdb -connect` connects to one.

### How it works

- **Transport:** plain HTTP or HTTPS. Requests are POSTs to `/quack`, and the default port is **9494**. Standard proxies, load balancers and firewalls can handle the traffic.
- **Client-driven:** the client starts every exchange, and the server never pushes. After the handshake, a query is one request and one response. Large results stream back in chunks through follow-up `FETCH` requests, which can run in parallel.
- **Data format:** DuckDB's internal serialization, the same code path its write-ahead log uses. Complex types (nested, `VARIANT` and so on) arrive intact.
- **Transactions:** `BEGIN ... COMMIT` on an attached Quack database is forwarded to the server.
- **Other remote databases:** `CONNECT` also works with PostgreSQL and MySQL. A new "remote pushdown" optimizer sends the SQL to those servers to run there, instead of pulling whole tables over the network:
  ```sql
  CONNECT 'postgres://localhost/mydb';
  SELECT count(*) FROM orders;
  DISCONNECT;
  ```

### Security defaults

- **Listening:** the server listens on **localhost only** by default. Binding to `0.0.0.0` needs `allow_other_hostname = true`, which the announcement says is "only meant for testing."
- **Production setup:** DuckDB recommends keeping the server on localhost and putting a **reverse proxy that handles TLS** in front of it.
- **Encryption:** local servers use plain HTTP. Non-local servers use HTTPS with a self-signed certificate unless you supply your own. `quack_generate_keys` creates a certificate, and clients can pin one with `SSL_FINGERPRINT`.
- **Authentication:** token-based, with a minimum of 4 characters. The token can be passed with `TOKEN` on `ATTACH`, stored in a `quack` secret, or generated at random.
- **Custom hooks:** the `quack_authentication_function` and `quack_authorization_function` settings let you point authentication and per-query authorization at your own SQL functions. By default every authenticated query is allowed.

### Useful functions and settings

- **Server management:** `quack_server_list()`, `quack_active_connections()` and `quack_stop()` show and stop running servers.
- **Queries:** `quack_query(uri, sql)` runs a single remote query without attaching. `quack_cancel()` cancels a query running on the server.
- **Limits:** `quack_server_max_connections` defaults to 1024. Reconnects with server-side result caching are off by default (`quack_enable_reconnects`).
- **Debugging:** `CALL enable_logging('Quack')` logs every protocol message on both the client and the server.

### What the docs don't say yet

- **Concurrent writes:** the docs don't explain how writes from several clients interact. My reading: all clients go through the one server process, so DuckDB's existing MVCC and multi-connection transaction isolation should handle them. If two clients change the same rows, expect normal DuckDB conflict errors.
- **Workload fit:** the docs make no claim that this suits high-concurrency transactional (OLTP) traffic. DuckDB is still an analytics engine, so treat Quack as a way to **share one analytical database with several clients**, not as a replacement for Postgres.
- **Attach failures:** `ATTACH` binds every remote table definition. If one of them fails to bind, the whole attach fails.

## Other notable changes in 2.0

- **Asynchronous I/O:** reads of Parquet, CSV and DuckDB files keep several requests in flight. The biggest gains are on S3 and other object storage.
- **VARIANT type:** now works end to end. Storage is shredded for compression and speed, including reading and writing shredded Parquet, and there is a family of `variant_*` functions.
- **New SQL parser:** a PEG-based parser replaces the one derived from PostgreSQL. Error messages show where the problem is, and extensions can add new syntax. The old parser is still available: load the `legacy_parser` extension and run `SET enable_legacy_parser = true`.
- **Triggers:** `CREATE TRIGGER` supports BEFORE/AFTER, row and statement level, and transition tables. Some combinations are not supported yet, such as BEFORE ... FOR EACH ROW on UPDATE.
- **SQL additions:**
  - `$variable` shorthand
  - CTEs that modify data
  - `NEAREST` joins for similarity search
  - Nested schemas
  - `json_set`, `json_insert`, `json_replace` and `json_remove`
  - `FETCH FIRST` and `UNNEST` in `GROUP BY`
  - Defined `MERGE` and `UPDATE ... FROM` behavior when several rows match
- **CLI agent mode:** the CLI detects coding agents. It switches to compact Markdown tables and JSON errors, stops runaway queries early, and reports expected cost before long queries. On TPC-H, the announcement says this cut the output an agent had to read by 59%. Use `-agent` or `-no-agent` to control it.
- **Performance:**
  - The recursive CTE engine was rewritten (one benchmark in the preview went from 4.90 s to 0.12 s).
  - Aggregations can spill to disk.
  - Partial aggregates are pushed down.
  - Data pruning was improved.
  - Native timezones and collations replace the ICU library, which made those operations about 2.2-2.6x faster.
- **Extensions:**
  - C API v2, not yet marked stable. The goal is extensions that don't need rebuilding for every release.
  - Custom signed extension repositories (`CREATE EXTENSION REPOSITORY`), still a work in progress.
  - DuckLake 1.1.
  - Delta can now `CREATE TABLE` and supports time travel.

## Breaking changes and upgrade advice

- **Storage format:** 2.0 still reads and writes old files, and those files stay readable by older versions. **New files created by 2.0 cannot be opened by 1.x.**
- **Lambda syntax:** `x -> x + 1` is disabled by default. Write `lambda x: x + 1` instead. `SET lambda_syntax = 'enable_single_arrow'` turns the old syntax back on for now, but both the setting and the old syntax are removed in 2.1.
- **Build and platform changes:**
  - Building now needs a C++17 compiler.
  - Amalgamation (single-file) builds are deprecated.
  - Intel Mac (`osx_amd64`) support ends in 2.1.
- **Support windows:** 1.4 is supported until 2026-11-17, and 1.5 until March 2027.
- **DuckDB's advice:** test on a shadow copy before switching production. For mission-critical work, wait for the first patch release, planned for November 2026.

## Sources

- [Announcing DuckDB 2.0.0 LTS "Cinnamon Teal"](https://duckdb.org/2026/10/21/announcing-duckdb-200) (official, 2026-10-21)
- [A Preview of DuckDB v2.0](https://duckdb.org/2026/08/17/duckdb-20-highlights.html) (official, 2026-08-17)
- [Quack overview, 2.0 docs](https://duckdb.org/docs/2.0/quack/overview)
- [Quack reference, 2.0 docs](https://duckdb.org/docs/2.0/quack/reference)
- [RuntimeWire: DuckDB 2.0 adds a server to a database built to run without one](https://runtimewire.com/article/duckdb-2-0-client-server-embedded-analytics)
- [AlternativeTo: DuckDB's major 2.0 release turns it into a full-fledged server](https://alternativeto.net/news/2026/10/duckdb-s-major-2-0-release-turns-the-database-management-system-into-a-full-fledged-server/)
