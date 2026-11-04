# DuckDB 2.0 and its server mode

*Researched 2026-11-03. DuckDB 2.0.0 LTS "Cinnamon Teal" shipped on 2026-10-21.*

## The short version

DuckDB 2.0 is the release where the "SQLite for analytics" database stops being only a library. Since 2019 DuckDB has run inside your process, like SQLite. 2.0 adds a production version of **Quack**, a client/server protocol. Any DuckDB process can now serve its databases over HTTP, and other DuckDBs can attach to it and send it queries. The team calls the coming year "the year of DuckDB as a server."

The server mode is narrower than "DuckDB is now Postgres." One DuckDB process still owns the database file, and Quack lets other processes reach it over the network. That fixes the main thing people hit, which is only one process at a time being able to write to a DuckDB file. It does not make DuckDB distributed, replicated or multi-user in the way a hosted service like MotherDuck is.

The rest of 2.0 is a large feature release: a new extensible SQL parser, asynchronous I/O, a full `VARIANT` type, triggers, an "agent mode" for the CLI, and a new on-disk format that older versions cannot read.

## Why a server mode, and why now

**The problem it solves is the file lock.** An in-process database keeps much of its state (caches, transaction bookkeeping) in the memory of the process that opened it. Two processes changing the same file would have to keep that memory in sync, so DuckDB avoids it: one process opens a file read-write and holds a lock, or any number of processes open it read-only. Inside that one process, many threads can write at the same time. Across processes, nobody else can. A web app with several workers, or one job loading telemetry while a dashboard reads it, ran straight into this.

**People had been building servers around it for years.** The Quack announcement lists the workarounds: custom RPC services, wrappers built on Arrow Flight SQL, MotherDuck's own protocol, and putting DuckDB inside PostgreSQL with pg_duckdb. The team says users "asked us – very persistently – for a client/server mode, and we have finally caved." Quack is the official version of what everyone was already building.

**DuckDB already had most of a server inside it.** Its transaction system was built for many connections from the start. It uses multi-version concurrency control (MVCC) adapted from the HyPer research database, plus optimistic conflict detection. Before 2.0, all those connections had to live in one process. Quack's main job is to let connections from other processes and machines reach the existing machinery. That is why a full server mode could arrive as one extension rather than a rewrite.

## How Quack works

**One DuckDB serves and other DuckDBs connect.**

```sql
-- server
CALL quack_serve('quack:localhost', token = 'super_secret');

-- client
ATTACH 'quack:localhost' AS qk (TOKEN 'super_secret');  -- use qk.tablename, or:
CONNECT qk;          -- now every query runs on the server; results stream back
...
DISCONNECT;          -- back to the local database
```

The CLI also has `duckdb -serve` and `duckdb -connect`. The default port is 9494. Transactions (`BEGIN … COMMIT`) are forwarded to the server. The client is an ordinary DuckDB with the `quack` extension, which installs and loads itself on first use. Columnar has released an ADBC driver for Quack in preview, and other third-party clients appeared within weeks of the announcement.

**It runs over plain HTTP.** The team argues it "would be rather misguided not to build a database protocol on top of HTTP in 2026," because HTTP brings load balancers, proxies, firewall rules and auth tooling for free. It also lets DuckDB-Wasm in a browser speak the protocol directly. Every exchange is a request from the client; the server never starts one.

**The data on the wire is DuckDB's own in-memory format.** Results go out as DuckDB's internal vector blocks, using the serializer DuckDB already uses for its write-ahead log (MIME type `application/duckdb`). Nothing is converted on either side. The team says it rejected two obvious alternatives on purpose:

- **PostgreSQL's wire protocol** sends rows one at a time, and standard clients don't read in parallel. In the team's benchmark, moving 60 million rows took 158 s over Postgres, against 4.9 s over Quack.
- **Arrow Flight SQL** is columnar like Quack, but every query needs at least two round trips (one call to plan, one to fetch). The format is also controlled outside DuckDB, and the team says "we cannot allow ourselves to be restricted by formats that are controlled externally." The same 60 million rows took 17.4 s over Flight.

A query on an open connection finishes in one round trip, and large results can be fetched in parallel across threads. In the team's small-write benchmark (single-row `INSERT` transactions on 8-vCPU machines), Quack reached about 5,400 transactions per second at 8 threads, against about 4,300 for Postgres. The team also says plainly that Postgres scales better past 8 threads, because DuckDB limits concurrent inserts into one table. These figures come from the DuckDB team's own May 2026 beta benchmarks.

**`CONNECT` is not only for Quack.** The same statement works against PostgreSQL and MySQL. A new remote pushdown optimizer sends your SQL to those servers instead of pulling whole tables into DuckDB. It replaces the older `postgres_query(...)`-style escape hatches.

## What to expect when you use it

The Quack docs don't spell out how concurrent writers behave. Because a Quack server is a normal DuckDB process handing client sessions to its usual transaction manager, the engine's documented behavior should carry over:

- **Appends from many clients don't conflict,** even into the same table. The ingest-plus-dashboard case is a good fit.
- **Two clients updating the same row at the same time:** the second gets a "Transaction conflict" error and must retry. There are no lock waits as in Postgres; the conflict is detected and the transaction fails. Treat this as an expectation until you test it; the Quack pages don't state it.
- **It is analytics-first, not an OLTP replacement.** The concurrency design favors fast reads. A few thousand small writes per second is the published ceiling. Community discussion of 2.0 reached the same verdict.
- **The file is still locked.** While the server has the database open for writing, other processes can't open the file directly, even read-only. Everything goes through the server.
- **It is still one machine.** There is no replication, no read replicas and no failover. MotherDuck's own write-up says that with Quack "you're still basically tied to a single compute instance." A replication layer for read replicas is on the roadmap, not in 2.0.

## Security: secure enough on localhost, plain elsewhere

- **The server listens only on localhost by default.** Binding to another host requires `allow_other_hostname = true`.
- **Authentication is one shared token.** If you don't supply a token, the server generates a random one at startup. There are no built-in users.
- **Auth and permissions are SQL hooks.** The authentication check (`quack_authentication_function`) and the per-query authorization check (`quack_authorization_function`) can be swapped for your own SQL macros. The docs show a token table where adding a user is an `INSERT`, and a read-only filter. The docs warn that this filter is "illustrative, not a robust read-only filter," because `WITH … INSERT` still writes. This is typical DuckDB: instead of adding a permission system, it hands you SQL to build one.
- **TLS changed between beta and 2.0.** The May beta did not use TLS at all. In 2.0, non-local servers default to HTTPS with a self-signed certificate, and you can supply your own. The official advice is still to keep Quack on localhost behind an nginx or Caddy reverse proxy that handles TLS, and never to expose it directly to the internet.

## The rest of 2.0, briefly

- **New SQL parser.** DuckDB's parser was derived from PostgreSQL's, which is why DuckDB's SQL has always felt Postgres-like. That kind of generated parser (YACC/Bison) breaks with grammar conflicts when someone adds syntax. 2.0 switches the default to a PEG parser, which tries alternatives in order, so extensions can add their own SQL syntax. Error messages now point to the exact location of the problem. Python made the same switch in 3.9. The parser is meant to be compatible apart from a few listed changes, and `SET enable_legacy_parser = true` brings back the old one.
- **Asynchronous I/O** for Parquet, CSV and DuckDB's own files, with many requests in flight. Reading from `s3://` and other network storage benefits most.
- **`VARIANT`** is integrated end to end, with "shredded" storage for semi-structured data, including in Parquet. The team plans to back `JSON` with it later.
- **Triggers** (`BEFORE`/`AFTER`, per row or per statement, with transition tables), a first step toward materialized views. Some combinations, such as `BEFORE … FOR EACH ROW`, are not supported yet.
- **SQL additions:** `NEAREST` joins for similarity search, data-modifying CTEs, nested schemas, `$variable` shorthand, JSON mutation functions, and defined `MERGE`/`UPDATE … FROM` behavior when several rows match.
- **Agent mode for the CLI.** When a coding agent such as Claude Code or Codex drives the CLI, it switches to compact Markdown tables, JSON errors, cost estimates and early stops on runaway queries. In the team's TPC-H tests the model had 59% less output to read. Turn it on or off with `-agent`/`-no-agent`.
- **ICU dependency removed.** Time zones, calendars and collations are now native, with time zone data of about 45 kB, and are 2.2–2.6× faster.
- **Extension C API v2 and signed private extension repositories.** The August preview promised a stable ABI so extensions wouldn't need rebuilding for each release. The final announcement says C API v2 is "not yet marked stable," and repository syntax "may change." Some coverage repeats the preview's stability claim; the release post is more cautious.

## Before you upgrade

- **The storage format only upgrades one way.** "DuckDB versions before 2.0 cannot read database files created by DuckDB 2.0." 2.0 can still read and write old files.
- **The single-arrow lambda `x -> x + 1` is off by default.** Use `lambda x: x + 1`. A flag re-enables the old form, but both are removed in 2.1.
- **Build and platform changes:** C++17 is required, the Python package now needs Python 3.11 or later, amalgamation builds are deprecated, and macOS Intel support ends in 2.1.
- **Support windows:** 2.0 is supported until at least October 2027, 1.5 until March 2027, and 1.4 only until 2026-11-17.
- **The team suggests mission-critical users wait for the first patch release,** planned for November. As of 2026-11-03 I found no 2.0.1.

## Where it sits

- **SQLite:** it stayed in-process for good. DuckDB started in the same camp and has now added a server on top, while keeping in-process use unchanged ("DuckDB as an in-process database will continue working just as it did before").
- **MotherDuck,** the commercial company built on DuckDB, publicly welcomes self-hosted Quack and plans to accept Quack connections itself. It positions its service as the layer Quack doesn't provide: real users and SSO, separate storage and compute, read scaling, scaling to zero, time travel, and an SLA. Quack covers the self-hosted single-server case; MotherDuck covers the managed, multi-user case.
- **DuckLake 1.1** shipped with 2.0 and is the other route to multiple writers: many DuckDB instances share Parquet files through a central catalog. A Quack server can now act as that catalog. The rough split: Quack when one server can hold the database, DuckLake when the data lives in object storage and compute has to scale out.
- **Reception** of Quack since its May beta has been mostly positive. Users call it "a big unlock" for running DuckDB as a shared service. Skeptics note that you still run one process yourself and handle deployment and operations yourself.

## What is DuckDB's own here

Most of Quack follows from pieces DuckDB already had: the transaction engine, the WAL serializer and the extension system. The distinctive choices are a database protocol that is HTTP-native, columnar and one round trip per query, with no external format standard. The client is the full engine, so a "client" can also run queries locally and join remote tables with local ones. Access control is left to SQL macros instead of a built-in user system.

## Sources

- [Announcing DuckDB 2.0.0 LTS "Cinnamon Teal"](https://duckdb.org/2026/10/21/announcing-duckdb-200) (DuckDB blog, 2026-10-21)
- [A Preview of DuckDB v2.0](https://duckdb.org/2026/08/17/duckdb-20-highlights.html) (DuckDB blog, 2026-08-17)
- [Quack: The DuckDB Client-Server Protocol](https://duckdb.org/2026/05/12/quack-remote-protocol.html) (DuckDB blog, 2026-05-12)
- [DuckDB 1.5.3: Not an Ordinary Patch Release](https://duckdb.org/2026/05/20/announcing-duckdb-153)
- Quack docs: [overview](https://duckdb.org/docs/2.0/quack/overview.html), [security](https://duckdb.org/docs/2.0/quack/security.html); [concurrency](https://duckdb.org/docs/2.0/connect/concurrency.html); [quack product page](https://duckdb.org/quack/); [duckdb-quack repo](https://github.com/duckdb/duckdb-quack)
- [Analytics-Optimized Concurrent Transactions](https://duckdb.org/2024/10/30/analytics-optimized-concurrent-transactions.html) (DuckDB's MVCC and its HyPer roots)
- [If It Quacks Like a Duck: the DuckDB Client-Server Protocol](https://motherduck.com/blog/duckdb-client-server/) (MotherDuck)
- [Announcing an ADBC driver for Quack](https://columnar.tech/blog/quack-adbc) (Columnar)
- InfoQ: [Quack protocol](https://infoq.com/news/2026/05/duckdb-quack-protocol), [DuckDB v2](https://infoq.com/news/2026/08/duckdb-v2-distributed); [Runtimewire](https://runtimewire.com/article/duckdb-v2-server-mode-embedded-analytics)
- [duckdb-python v2.0.0 release](https://github.com/duckdb/duckdb-python/releases/tag/v2.0.0)
