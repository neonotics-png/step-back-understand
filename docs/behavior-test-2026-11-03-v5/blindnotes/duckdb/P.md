# DuckDB 2.0 and its server mode

DuckDB 2.0.0 LTS ("Cinnamon Teal") shipped on October 21, 2026, with support promised until at least October 2027. Where 1.0 was a stability release, the team calls 2.0 "above all a feature release": close to 20,000 commits since 1.5. The headline feature is that DuckDB, a database built to run inside your program, can now also run as a server that other DuckDB processes connect to.

## What server mode is

Server mode means one DuckDB process serves its databases over a protocol called **Quack**, and other DuckDB processes connect to it as clients:

```sql
-- on the server
CALL quack_serve(token = 'my_token');

-- on a client
ATTACH 'quack:server.example.com' AS qk (TOKEN 'my_token');
CONNECT qk;          -- the rest of this session runs on the server
SELECT ...;          -- executes remotely, results stream back
DISCONNECT;          -- back to the local database
```

The CLI also gets `-serve` and `-connect` shortcuts. Quack itself is not new in 2.0. It first appeared on May 12, 2026, and shipped as a beta core extension in 1.5.3, with "production readiness" promised for 2.0. What 2.0 adds is a polished Quack ("substantial performance and stability improvements") and the `CONNECT` statement.

## Why a server, and why it was easy to add

This is the part that makes the feature make sense. DuckDB already handled many concurrent writers: inside one process it has had full MVCC and transaction isolation "since day one", and many threads can write at once. Two writers only clash when they touch the same row, and then the second one gets a conflict error. The limit was across processes. Only one process could open a database file for writing, because DuckDB keeps live state in memory and that state can't be synchronized between processes. In practice this broke a common pattern: one job ingesting telemetry while dashboards query the same file.

That limit is inherited from the in-process model DuckDB shares with SQLite: no server, no protocol, just a library and a file. People got around it by writing their own RPC services, putting Arrow Flight SQL in front of DuckDB (GizmoSQL), using MotherDuck's hosted protocol, or giving up and using Postgres or pg_duckdb. The team says users asked "very persistently" and that "we have finally caved."

So Quack is not a new concurrency engine. It is a new front door. One process owns the file and its in-memory state, every client goes through that process, and the MVCC DuckDB always had now covers writers in other processes and on other machines. That is why the feature could be added as an extension instead of a rewrite.

## How the protocol is built, and why

- **It runs over plain HTTP** (default port 9494). Clients send every request, and the server never pushes. Once connected, a query takes one request/response, and large results come back in chunks that can be fetched in parallel threads. The stated reason is operational: "Everyone and their little brother knows how to deal with HTTP in load balancing, authentication, firewalls, intrusion detection etc." A side benefit is that DuckDB-Wasm in a browser can speak Quack natively.
- **The payload is DuckDB's own serialization** (`application/duckdb`), the same code path as its write-ahead log, not Arrow. That keeps nested types, decimals and intervals exact, and the team can add types without waiting on an outside format. They also rejected Arrow Flight SQL because it needs at least two round trips per query, which hurts small writes.
- **Performance, by the team's own benchmark** (May 2026, on 1.5.3, AWS, same availability zone). Moving 60M rows of TPC-H lineitem took 4.9 s with Quack, 17.4 s with Arrow Flight and 158 s with Postgres. Single-row insert transactions reached about 5,400/s at 8 threads, against about 4,300/s for Postgres. Beyond 8 threads, DuckDB's limit on concurrent inserts into one table becomes the bottleneck and Postgres scales better. These are vendor numbers, and 2.0 says it improved performance without publishing new figures.
- **Security is minimal by design.** The server makes a random token at startup, binds only to localhost unless you set `allow_other_hostname`, and doesn't do TLS itself. The advice is to keep it on localhost and put a TLS-terminating reverse proxy such as nginx in front. Authentication and authorization are hooks you can replace, even with SQL macros (for example, check tokens against LDAP). By default, any authenticated query is allowed.

## CONNECT is bigger than Quack

`CONNECT` points your whole session at any remote database that supports it, not just another DuckDB. `CONNECT 'postgres://localhost/mydb'` works too. A new remote pushdown optimizer sends the SQL to Postgres or MySQL instead of pulling tables over the wire. This builds on DuckDB's existing `ATTACH` for Postgres, MySQL and SQLite. Earlier, remote work went through a `remote.query($$...$$)` workaround, which the authors rejected: "no, this cannot be it." Note the direction: DuckDB connects out to Postgres. It does not speak the Postgres wire protocol, so `psql` and other Postgres tools can't connect to a Quack server. For non-DuckDB clients there are third-party Quack clients, including Columnar's ADBC driver.

## What to expect (some of this is inferred, not confirmed)

- **One server is one point of failure.** All state lives in one process, and there is no replication yet. Read replicas are only a "maybe" on the roadmap.
- **Hot-row writers fail; they don't wait.** Because the server uses the same optimistic concurrency control, clients updating the same rows will get conflict errors to retry, not lock waits. The docs say this for in-process writers. I haven't seen it stated for Quack, but it follows from the design.
- **For serious multi-instance production, the docs still point elsewhere.** The 2.0 concurrency docs list DuckLake 1.1 with a Postgres catalog as the production route for many instances reading and writing. Quack is where DuckDB is heading ("the year of DuckDB as a server"), but it is 5 months old. The pages I read don't clearly call it stable.
- **It is DuckDB-to-DuckDB first.** The protocol is DuckDB's own internal format, so clients in other languages depend on third-party drivers that have to keep up with format changes.

## The rest of 2.0, briefly

- **New SQL parser.** A PEG parser replaces the one inherited from PostgreSQL. It gives precise error locations and lets extensions add grammar. `enable_legacy_parser` is the escape hatch.
- **Async I/O** for Parquet, CSV and native files. It helps most on S3 and other object storage.
- **VARIANT is now a full type.** It is "shredded" into columns for speed and compression, including in Parquet.
- **Triggers** (`CREATE TRIGGER`, BEFORE/AFTER, row/statement, transition tables), with some limits.
- **Smaller SQL additions:** `$var` variables, data-modifying CTEs, `NEAREST` joins, nested schemas, JSON mutation functions, and a much faster recursive-CTE engine (about 40x on the team's reachability benchmark).
- **CLI agent mode** for coding agents: compact tables, JSON errors, early stops for runaway queries.
- **C API v2** for extensions (not yet marked stable) and signed extension repositories.

## Before upgrading

- **The storage format is one-way.** Older DuckDB versions cannot read files created by 2.0, though 2.0 still reads and writes old files.
- **Lambda syntax changed.** Single-arrow lambdas (`x -> x + 1`) are off by default; use `lambda x: x + 1`.
- **ICU is gone.** Timezone, calendar and collation code is now native.
- **macOS x86_64** is being phased out, and 2.1 drops it.
- **Patch timing.** For anything mission-critical, the team suggests waiting for the first patch release, due in November 2026.

## Sources

- [Announcing DuckDB 2.0.0 LTS "Cinnamon Teal"](https://duckdb.org/2026/10/21/announcing-duckdb-200) (Oct 21, 2026)
- [A Preview of DuckDB v2.0](https://duckdb.org/2026/08/17/duckdb-20-highlights.html) (Aug 17, 2026)
- [Quack: The DuckDB Client-Server Protocol](https://www.duckdb.org/2026/05/12/quack-remote-protocol) (May 12, 2026)
- [DuckDB 1.5.3: Not an Ordinary Patch Release](https://duckdb.org/2026/05/20/announcing-duckdb-153.html)
- [Quack overview, 2.0 docs](https://duckdb.org/docs/2.0/quack/overview.html)
- [Concurrency, 2.0 docs](https://duckdb.org/docs/2.0/connect/concurrency)
- [Columnar: ADBC driver for Quack](https://columnar.tech/blog/quack-adbc)
