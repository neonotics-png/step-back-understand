# DuckDB 2.0 and Quack Client/Server Mode: Answer Key

Each question is worth 2 points (16 total). Award partial credit only as described in the rubric. Accept equivalent wording; the ideas matter, not the exact terms.

---

## 1. Server and client roles; ATTACH vs CONNECT

**Model answer.** Both sides are ordinary DuckDB processes: the `quack` extension lets any DuckDB instance serve the databases it can see (for example with `CALL quack_serve(...)`), and other DuckDB instances connect to it as clients using a `quack:` URI and a token. A client can `ATTACH` the server as a remote catalog and then qualify table names (for example `qk.my_table`), with scans, filters, writes, DDL and transactions forwarded to the server. `CONNECT qk` goes further and routes the whole session to the server, so every unqualified query executes remotely and its results stream back, until `DISCONNECT` returns the session to the local database. (There is also a stateless `quack_query(uri, query)` for one-off remote queries.)

**Rubric.**
- 2 points: States that both server and client are DuckDB instances (the server is a DuckDB process running the Quack extension, not a separate server product), AND correctly distinguishes ATTACH (remote catalog, qualified names) from CONNECT (session routing, unqualified queries run remotely, DISCONNECT returns to local).
- 1 point: Gets the roles right but blurs ATTACH and CONNECT, or explains ATTACH/CONNECT correctly but misdescribes the server as something other than a DuckDB process.
- 0 points: Describes Quack as a hosted service, a separate server binary unrelated to DuckDB, or says queries run locally against downloaded tables.

**Sources.**
- https://duckdb.org/docs/2.0/quack/overview.html
- https://duckdb.org/docs/2.0/sql/statements/connect.html
- https://duckdb.org/2026/10/21/announcing-duckdb-200
- https://duckdb.org/2026/08/17/duckdb-20-highlights.html

---

## 2. Transport, encoding and round trips

**Model answer.** Quack runs over plain HTTP (or HTTPS, typically via a reverse proxy) with a strictly client-driven request/response pattern: the server never pushes, every exchange starts with a client request. Payloads use a DuckDB-specific `application/duckdb` serialization built on DuckDB's internal serialization primitives (the same code path as the write-ahead log), so result data and complex types (nested types, decimals, intervals) cross the wire losslessly without being transcoded into an interchange format. After the initial connection handshake a query needs only one request/response pair, and large results are retrieved in chunks through follow-up FETCH requests that can run in parallel on multiple threads.

**Rubric.**
- 2 points: Covers all of: HTTP transport; client-initiated request/response; native DuckDB serialization (no conversion to Arrow/rows or similar); single round trip per query after connecting, with large results fetched in follow-up (optionally parallel) requests.
- 1 point: Gets two or three of those four elements right, or gets all of them but with one material error (for example, says the server pushes results, or that Quack uses Arrow on the wire).
- 0 points: Describes a custom TCP or PostgreSQL-style wire protocol, or cannot describe the encoding or round-trip behavior at all.

**Sources.**
- https://duckdb.org/2026/05/12/quack-remote-protocol.html
- https://duckdb.org/docs/2.0/quack/overview.html
- https://motherduck.com/blog/duckdb-client-server/
- https://alternativeto.net/news/2026/10/duckdb-s-major-2-0-release-turns-the-database-management-system-into-a-full-fledged-server/

---

## 3. Default security and the two hooks

**Model answer.** By default the server binds only to localhost (external addresses need `allow_other_hostname = true`) and requires a token on every connection, generating a random one at startup if none is supplied; local URIs skip TLS, and non-local servers default to HTTPS with a self-signed certificate. What the defaults do not give you is any user model or restriction on what an authenticated client may do: the default authorization function allows every query, so anyone holding the token can run any SQL, including writes. Authentication (a function called on each connection request, default `quack_check_token`) decides who the caller is, while authorization (called before each query, default a no-op) decides whether that query may run, returning a boolean or a replacement query. Operators can swap in their own functions, simplest as SQL macros (for example a token-table lookup), or as extension-provided scalar functions for stateful logic such as auditing or per-user ACLs.

**Rubric.**
- 2 points: Names the localhost-only default and the required (auto-generated if absent) token, states that default authorization is permissive so a token holder can do anything, AND distinguishes authentication (at connect) from authorization (per query) with a correct note on how to customize them.
- 1 point: Describes the defaults correctly but not the two hooks, or describes the hooks but misses that default authorization is wide open.
- 0 points: Claims Quack has built-in users/roles or fine-grained permissions by default, or that it is safe to expose directly to the internet as shipped.

**Sources.**
- https://duckdb.org/docs/2.0/quack/security.html
- https://duckdb.org/2026/05/12/quack-remote-protocol.html
- https://duckdb.org/docs/2.0/quack/overview.html
- https://motherduck.com/blog/duckdb-client-server/

---

## 4. Why HTTP and a custom format, not Arrow Flight SQL or the PostgreSQL protocol

**Model answer.** HTTP was chosen because its ecosystem is mature and well understood (load balancers, reverse proxies, firewalls, authentication, intrusion detection), so there is no custom transport to operate, and because it lets DuckDB-Wasm in the browser speak Quack natively. Arrow Flight SQL was rejected because the team did not want DuckDB's internals tied to an externally governed format (they want to add a type or message and "ship tomorrow"), and because Flight SQL needs at least two round trips per query (`CommandStatementQuery` then `DoGet`), which hurts small transactions on high-latency links. The PostgreSQL wire protocol was rejected because it is row-based and very slow for bulk transfer, and standard clients do not parallelize reads; the team's own benchmarks showed Quack far ahead of PostgreSQL on large transfers and ahead of both alternatives on small single-insert transactions up to eight threads.

**Rubric.**
- 2 points: Gives at least one solid reason for HTTP (tooling/infrastructure compatibility or DuckDB-Wasm) AND at least one correct reason against Flight SQL (control over format/evolution or extra round trips) AND at least one against the PostgreSQL protocol (row-based, poor bulk throughput or no parallel reads).
- 1 point: Correct reasoning for two of the three choices, or generic "performance" claims without the specific mechanisms.
- 0 points: Reasons that contradict the sources (for example, that Quack uses Flight SQL underneath, or that HTTP was chosen for speed of raw streaming), or no real reasons.

**Sources.**
- https://duckdb.org/2026/05/12/quack-remote-protocol.html
- https://infoq.com/news/2026/05/duckdb-quack-protocol
- https://duckdb.org/docs/2.0/quack/overview.html

---

## 5. Why a server process instead of multi-process file writes

**Model answer.** DuckDB keeps a lot of database state in main memory (caches and transaction state that make analytics fast), and safely letting several independent processes write the same file would require synchronizing that in-memory state across processes, which the team says there are good technical reasons it could not make work. Putting one server process in charge of the file sidesteps this: remote clients become sessions inside that single process, so the existing engine stays authoritative. The team argues this fits because DuckDB was always a transactional, multi-connection engine with full MVCC and transaction isolation, capabilities most embedded single-user users never exercised, and client/server deployment lets that machinery serve multi-tenant, long-running workloads while in-process use remains a first-class option.

**Rubric.**
- 2 points: Explains that cross-process writing would require synchronizing in-memory state (or equivalently, that DuckDB's design relies on one process owning the file and its cached state), AND connects the server approach to DuckDB's existing in-process MVCC/multi-connection transaction machinery.
- 1 point: Gives only one of these two reasons, or gives a vaguer but compatible explanation (for example "file locking prevents it" without explaining why the server approach works).
- 0 points: Claims DuckDB lacked transactions or concurrency before 2.0, or that Quack replaces the in-process mode.

**Sources.**
- https://duckdb.org/2026/05/12/quack-remote-protocol.html
- https://duckdb.org/2026/08/17/duckdb-20-highlights.html
- https://duckdb.org/docs/2.0/connect/concurrency.html
- https://duckdb.org/quack/faq.html

---

## 6. What came before and the alternatives around it

**Model answer.** Before Quack, a DuckDB database file could be opened read-write by only one process (which could itself run many concurrent writer threads), or read-only by several processes, so a pattern like "one process ingests while another serves a dashboard" did not work. People worked around this with custom RPC services wrapping a DuckDB process, Arrow Flight SQL front ends (such as the airport extension or GizmoSQL), MotherDuck's proprietary client/server protocol, moving to PostgreSQL (sometimes with DuckDB embedded via pg_duckdb), or DuckLake with a PostgreSQL catalog. Quack, by contrast, keeps DuckDB's native file format, needs only the extension on both ends, and suits data up to a few terabytes, while DuckLake puts data in object storage behind a separate catalog database and targets petabyte scale and multiple engines; the two can be combined, with a Quack server acting as a DuckLake catalog. MotherDuck is a managed service with its own protocol and features Quack lacks (user management, SSO, backups, dual execution, managed scaling and storage), and it has said it plans to support Quack as an endpoint type.

**Rubric.**
- 2 points: Correctly states the pre-Quack single-writer-process / multi-reader-read-only model, names at least two real workarounds, AND correctly contrasts Quack with both DuckLake and MotherDuck on at least one substantive dimension each.
- 1 point: Gets the prior model and workarounds right but only one of the two comparisons, or makes both comparisons but misstates the prior model.
- 0 points: Says DuckDB already supported multi-process writers, or confuses Quack with DuckLake or with MotherDuck's protocol.

**Sources.**
- https://duckdb.org/docs/2.0/connect/concurrency.html
- https://duckdb.org/2026/05/12/quack-remote-protocol.html
- https://duckdb.org/quack/faq.html
- https://motherduck.com/blog/duckdb-client-server/

---

## 7. Reception and why

**Model answer.** Reception was largely positive: developers on Hacker News and elsewhere saw it as a way to get shared, multi-user and remote access to DuckDB without moving to a heavier database (one called it the answer to "how do I horizontally scale this"), MotherDuck publicly welcomed it and announced plans to support it, and third parties shipped standalone clients within weeks, such as Columnar's ADBC driver, so non-DuckDB programs could talk to Quack servers. The enthusiasm came from removing a long-standing limitation that had forced people into workarounds, with the benefit of a simple setup and strong vendor benchmarks. Skeptics questioned DuckDB's scope ("not sure what it wants to be"), argued HTTP is a poor fit for bulk streaming and that the benchmarks were favorable to Quack, were confused about where it fits relative to DuckLake, and asked whether "concurrent writers" just meant serialized writes on the server. MotherDuck also stressed that self-hosters still own authentication, scaling, failover and support, since Quack ships with a shared-token model and no user concept.

**Rubric.**
- 2 points: Describes the generally positive reception with at least one concrete reason (removes the multi-writer/remote-access limitation, simple setup, ecosystem support such as MotherDuck or third-party clients) AND at least one substantive criticism or caveat with its reasoning.
- 1 point: Covers only the positive side or only the critical side, or lists reactions without explaining why they arose.
- 0 points: Invents reactions not supported by sources (for example, that it was widely rejected or that MotherDuck opposed it), or gives no meaningful account.

**Sources.**
- https://infoq.com/news/2026/05/duckdb-quack-protocol
- https://hackernews-master.vercel.app/item/48111765 (mirror of Hacker News item 48111765, https://news.ycombinator.com/item?id=48111765)
- https://motherduck.com/blog/duckdb-client-server/
- https://columnar.tech/blog/announcing-quack-adbc-driver/
- https://duckdb.org/2026/08/17/duckdb-20-highlights.html

---

## 8. Apply: live ingestion, dashboards, analyst corrections, internet access

**Model answer.** One DuckDB process should own the database file and run `quack_serve`, with the ingestion service, the dashboard and the analysts all connecting as Quack clients (ATTACH and then CONNECT, or qualified names); trying to open the file read-write from a second process would not work. Because every client becomes a connection inside that one server process, DuckDB's normal MVCC and optimistic concurrency rules apply: the ingestion appends never conflict, but if both analysts update the same row concurrently, the second transaction fails with a conflict error and must be retried. Before exposing it, they should keep the server bound to localhost and put a TLS-terminating reverse proxy (such as nginx or Caddy) in front rather than using `allow_other_hostname` directly, give each user their own token, and replace the permissive default authorization (for example with a function that inspects the statement type, or read-only access for anyone who should not write), because by default any token holder can run any SQL. They should also not expect Quack to spread queries across machines: it does no distributed query processing, so all work runs on that one server.

**Rubric.**
- 2 points: Single server process owning the file with all others as clients; correct prediction for concurrent same-row updates (second fails with a conflict and must retry, while appends do not conflict); AND at least two of the hardening steps (localhost plus TLS reverse proxy, per-user tokens, non-permissive authorization or read-only access).
- 1 point: Correct architecture but a wrong or missing conflict prediction, or correct conflict behavior but weak or missing hardening (for example, "just set allow_other_hostname and share the token").
- 0 points: Proposes multiple processes writing the file directly, claims both updates silently succeed or are merged, or recommends exposing the default server straight to the internet.

**Sources.**
- https://duckdb.org/docs/2.0/connect/concurrency.html
- https://duckdb.org/docs/2.0/quack/security.html
- https://duckdb.org/docs/2.0/quack/overview.html
- https://duckdb.org/2026/10/21/announcing-duckdb-200
- https://duckdb.org/2026/08/17/duckdb-20-highlights.html
- https://duckdb.org/quack/faq.html
