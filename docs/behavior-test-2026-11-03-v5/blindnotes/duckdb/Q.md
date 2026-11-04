# DuckDB 2.0 and its new server mode (Quack)

Researched 2026-11-03 from the official DuckDB blog and docs plus news coverage. Sources are at the end.

## The short version

- **DuckDB 2.0.0 LTS ("Cinnamon Teal") shipped on October 21, 2026.** It is an LTS release, supported until at least October 2027. The DuckDB team calls it their biggest feature release so far, with close to 20,000 commits since 1.5.
- **The big change is server mode.** DuckDB has always been an *embedded* database, a library that runs inside your app with no server, like SQLite for analytics. With 2.0, any DuckDB process can also act as a **server** that other DuckDB processes connect to over the network. The protocol is called **Quack**.
- **Why it matters:** before 2.0, only one process could have a DuckDB database file open for writing. Quack is now the officially supported way for many processes or machines to read and write the same database at once.
- **Context:** in August 2026 AWS agreed to acquire DuckLabs, the company behind DuckDB (about 30 staff, staying in Amsterdam). DuckDB, DuckLake and Quack stay MIT-licensed under the non-profit DuckDB Foundation.

## Server mode (Quack) explained

### What problem it solves

DuckDB keeps much of a database's state in memory inside the process using it. That works for one process. When several processes try to change the same file, that in-memory state can't be kept in sync, so DuckDB allowed only one writer process. People built workarounds: custom RPC services, Arrow Flight SQL wrappers, or putting PostgreSQL in front. Quack is DuckDB's own answer. One DuckDB process owns the database and serves it, and everyone else connects as a client.

### How it works

- **Over HTTP or HTTPS.** Ordinary load balancers, firewalls and reverse proxies work with it, and DuckDB-Wasm in a browser can connect directly.
- **The client always asks and the server answers.** A query normally takes one round trip. Large results stream back in chunks that can be fetched in parallel.
- **DuckDB's own binary format** (`application/duckdb`, the same serialization DuckDB uses for its write-ahead log). Complex types survive the trip without loss.
- **Default port 9494.** Addresses look like `quack:localhost`, `quack:myhost:9000` or `quack:[::1]:1234`.
- **The `quack` extension implements it.** It was first released as a preview in DuckDB 1.5.3 (May 2026). 2.0 ships a more polished version that the docs say has substantial performance and stability improvements.

### Using it

Start a server from any DuckDB session:

```sql
CALL quack_serve('quack:localhost', token = 'my_token');
-- stop it later:
CALL quack_stop('quack:localhost');
```

Connect from another DuckDB:

```sql
ATTACH 'quack:server.example.com' AS qk (TOKEN 'my_token');
CONNECT qk;                 -- send everything in this session to the server
SELECT count(*) FROM events; -- runs on the server, results stream back
DISCONNECT;
```

From the command line, use `duckdb -serve` to start a server and `duckdb -connect` to connect to one.

There are three ways for a client to use a remote database:
1. **`ATTACH` + `CONNECT`** (recommended): the whole session runs on the server.
2. **`ATTACH` alone:** the remote appears as another catalog, and you query it with qualified names (`remote_db.table`). Writes, DDL and `BEGIN`/`COMMIT` are forwarded to the server.
3. **`quack_query(uri, sql)`:** one-off queries with no session state.

For tokens, the docs recommend a `quack` secret over typing the token into `ATTACH`:

```sql
CREATE SECRET (TYPE quack, TOKEN '...', SCOPE 'quack:localhost');
ATTACH 'quack:localhost' AS remote_db (TYPE quack);
```

### `CONNECT` also works with PostgreSQL and MySQL

`CONNECT` is a general new statement, not only for Quack. You can point a session at PostgreSQL or MySQL, and a new "remote pushdown" optimizer sends the SQL to that server instead of pulling whole tables over the network:

```sql
CONNECT 'postgres://localhost/mydb';
SELECT count(*) FROM orders;  -- runs on PostgreSQL
DISCONNECT;
```

### Security defaults

- **The server listens only on localhost by default.** Exposing it on other addresses needs `allow_other_hostname = true`, for example `quack_serve('quack:0.0.0.0:9494', allow_other_hostname = true)`.
- **A token is always required.** If you don't set one, the server generates a random token at startup.
- **TLS:** connections to localhost use plain HTTP. Connections to other hosts default to HTTPS with a self-signed certificate unless you supply `ssl_cert_file` and `ssl_key_file`.
- **Default authorization allows everything.** Anyone with the token gets the full SQL surface, including reads and writes. You can plug in your own authentication and authorization functions (`quack_authentication_function`, `quack_authorization_function`), even as SQL macros. The docs warn that their example read-only filter is only illustrative: a query that starts with `WITH` can still contain an `INSERT`.
- **The official advice for production:** keep the server on localhost, put nginx or Caddy in front to handle TLS, and don't expose Quack directly to the internet.

### Performance (DuckDB's own benchmarks, May 2026)

Run on AWS m8g.2xlarge instances in the same availability zone:

| Test | Quack | Arrow Flight | PostgreSQL |
|---|---|---|---|
| Transfer 60M TPC-H lineitem rows | 4.94 s | 17.40 s | 158.37 s |
| Single-row INSERT transactions/sec, 8 threads | ~5,434 | ~half of PG | ~4,320 |

### Limitations and caveats

- **Write conflicts still happen.** Concurrent transactions that touch the same data can conflict, and the usual fix is to retry the transaction.
- **Insert scaling:** concurrent inserts into one table stop scaling at about 8 threads because of current DuckDB internals. PostgreSQL scales better there.
- **Attaching can fail as a whole:** `ATTACH` loads all of the remote's table definitions, so one bad definition (for example some lazy Iceberg catalog cases) can make the whole attach fail.
- **Security is minimal by default.** Real access control needs custom callbacks or an extension. Python UDFs aren't visible to those callbacks.
- **Still maturing:** the 1.5.x docs said the protocol, function names and defaults could still change. Planned work includes serving DuckLake catalogs over Quack, read replicas (a replication protocol), and higher transaction throughput.

### When to use it vs alternatives

- **Quack:** you want a central DuckDB that several apps, services or browsers read and write, for example telemetry ingestion while dashboards query the same data.
- **DuckLake 1.1** (also released with 2.0): you want many DuckDB instances to read and write a lakehouse with data in object storage and the catalog in PostgreSQL.
- **Plain embedded DuckDB:** still the default for single-process analytics, notebooks and scripts. Nothing changes there.

## Other notable 2.0 changes

- **Async I/O:** Parquet, CSV and native-format reads now keep many requests in flight at once, which helps most on S3 and other remote storage.
- **VARIANT type:** fully integrated, with "shredding" that stores semi-structured data in columnar form, plus `variant_*` functions and shredded Parquet support.
- **Triggers:** `CREATE TRIGGER` supports BEFORE/AFTER, row and statement level, and transition tables. Some combinations are not supported yet, such as BEFORE ... FOR EACH ROW on UPDATE.
- **CLI agent mode:** the CLI detects coding agents (e.g. Claude Code, Codex) and switches to compact Markdown tables, JSON errors, truncation flags, runaway-query stops and cost estimates. On TPC-H tests this cut agent-facing output by 59%. Control it with `-agent` / `-no-agent`.
- **New PEG parser by default:** better error locations, and extensions can add SQL syntax. The old parser is available through the `legacy_parser` extension.
- **SQL additions:** `$variable` shorthand, data-modifying CTEs, `NEAREST` joins, nested schemas, `json_set`/`json_insert`/`json_replace`/`json_remove`, `FETCH FIRST`, and `UNNEST` in `GROUP BY`.
- **Extensions:** C API v2 (not yet marked stable), and organizations can host signed extension repositories of their own (work in progress).
- **Built-in timezones and collations:** ICU is no longer needed. Timezone data is about 45 kB, and timezone/collation work runs about 2.2 to 2.6 times faster than before.

## Upgrading: breaking changes

- **Storage format:** versions before 2.0 can't read database files created by 2.0. 2.0 still reads and writes older files.
- **Lambda syntax:** `x -> x + 1` is off by default. Use `lambda x: x + 1`. You can re-enable the old form temporarily with `SET lambda_syntax = 'enable_single_arrow'`. Both the setting and the old syntax are removed in 2.1.
- **Builds:** C++17 is required, Clang is the default compiler on Linux and macOS, and amalgamation builds are deprecated. Intel macOS (`osx_amd64`) will be dropped in 2.1.
- **Support windows:** 1.4 is supported until November 17, 2026. 1.5 support is extended to March 2027.
- **The team's advice:** test on a shadow system first. For mission-critical workloads, wait for the first patch release, planned for November 2026.

## Sources

- [Announcing DuckDB 2.0.0 LTS "Cinnamon Teal" (duckdb.org, 2026-10-21)](https://duckdb.org/2026/10/21/announcing-duckdb-200)
- [A Preview of DuckDB v2.0 (duckdb.org, 2026-08-17)](https://duckdb.org/2026/08/17/duckdb-20-highlights.html)
- [Quack: The DuckDB Client-Server Protocol (duckdb.org, 2026-05-12)](https://www.duckdb.org/2026/05/12/quack-remote-protocol)
- [Quack overview docs, 2.0](https://duckdb.org/docs/2.0/quack/overview)
- [Quack security docs, 2.0](https://duckdb.org/docs/2.0/quack/security)
- [Concurrency docs, 2.0](https://duckdb.org/docs/2.0/connect/concurrency)
- [DuckLabs to Join AWS, Projects to Remain Open Source (duckdb.org, 2026-08-26)](https://duckdb.org/2026/08/26/ducklabs-to-join-aws)
- [AlternativeTo: DuckDB's 2.0 release turns it into a full-fledged server](https://alternativeto.net/news/2026/10/duckdb-s-major-2-0-release-turns-the-database-management-system-into-a-full-fledged-server/)
- [RuntimeWire: DuckDB 2.0 adds a server to a database built to run without one](https://runtimewire.com/article/duckdb-2-0-client-server-embedded-analytics)
- [GeekWire: Amazon acquires DuckLabs](https://geekwire.com/2026/amazon-acquires-ducklabs-adding-the-team-behind-duckdb-amid-broader-shakeup-in-cloud-data/)
