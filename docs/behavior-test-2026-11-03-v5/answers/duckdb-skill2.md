# DuckDB 2.0 / Quack quiz answers

## 1. Server and client roles; ATTACH vs CONNECT
Both roles are played by DuckDB. The server is a DuckDB process that owns the database file and runs `CALL quack_serve(...)`. The client is another DuckDB process (CLI, Python, Wasm and so on) that runs `ATTACH 'quack:host' AS qk (TOKEN ...)`, or creates a `quack` secret first.

**ATTACH as a catalog.** Attaching makes the remote database one more catalog in your local session. You name its tables explicitly (`qk.tbl`) and can mix them with local tables. DDL, writes, filters and transactions are forwarded, and the client binds all the remote table definitions at attach time. I'm not sure exactly how much of a mixed query runs remotely versus locally in 2.0. Before `CONNECT` existed, shipping a whole query meant using `remote.query(...)` or the stateless `quack_query(...)`.

**CONNECT.** `CONNECT qk` switches the whole session to the remote, so plain unqualified SQL runs on the server and results stream back until you `DISCONNECT`. The docs recommend this approach.

## 2. How requests and results travel
Quack runs over plain HTTP: POSTs to a `/quack` endpoint on default port 9494. HTTPS is meant to come from a reverse proxy. The client starts every exchange, and the server never pushes.

The payload uses DuckDB's own internal serialization (MIME type `application/duckdb`), the same code it uses for the write-ahead log, not Arrow. After the connection handshake, which carries the token, a query needs one round trip, and the reply includes the first part of the result. Large results are pulled with follow-up FETCH requests, and those can run in parallel across threads.

## 3. Access protection
**What you get by default:**
- The server makes a random token at startup, or uses one you supply, and clients must present it.
- The server binds only to localhost and refuses other hostnames unless you set `allow_other_hostname`.

**What you don't get:**
- The server doesn't do TLS itself. Local binding means plain HTTP, and the docs mention a self-signed certificate if you bind externally.
- Authorization is allow-all: anyone holding the token can run any query, including DDL and deletes.
- There are no users or roles.
- The team says not to expose Quack directly to the internet.

**The two hooks.** The authentication hook decides whether a connection's credential is valid. You can replace it, for example to check an LDAP directory or a text file. The authorization hook runs per query and can inspect the query alongside the authentication string to allow or reject it. Both hooks can be written as plain SQL macros.

## 4. Why HTTP and a custom format, not Arrow Flight SQL or the Postgres wire protocol
**HTTP.** It is the de facto layer above TCP, and the whole network stack is tuned for it. Load balancers, auth proxies, firewalls, intrusion detection and TLS termination (nginx, Let's Encrypt) already handle it. It also lets DuckDB-Wasm in a browser speak Quack natively.

**Own serialization, not Arrow Flight SQL.**
- DuckDB's internal structures differ from Arrow, and the team avoids depending on a format someone else controls.
- Owning the format keeps types such as nested values, decimals and intervals exact, and lets them add types or protocol messages without waiting on outsiders.
- Flight SQL needs at least two round trips per query (CommandStatementQuery, then DoGet), which hurts small writes, especially over high-latency links.

**Not the Postgres wire protocol.** The team's post doesn't discuss it directly, so this is partly inference. Their benchmark showed standard Postgres clients are row-oriented and don't parallelize reads (60M rows took 158 s versus 4.9 s with Quack). Adopting it would also tie DuckDB's type system to Postgres's.

## 5. Why a server process instead of shared file writes
DuckDB keeps important live state in memory: the MVCC transaction state, caches and similar. Letting several independent processes write one file would mean synchronizing that memory state across processes, which the design doesn't support. That is why only one process may open a file for writing.

A server process sidesteps the problem: one process owns the file and its state, and everyone else talks to it. The team's argument is that DuckDB was "a transactional, multi-connection database with full MVCC and transaction isolation since day one." Many threads in one process could already write concurrently, so funneling outside writers into that one process reuses the concurrency engine they already had. That is why the feature could ship as an extension rather than a rewrite.

## 6. Before Quack, and how Quack differs from DuckLake and MotherDuck
**The old model.** Within one process, you could have many reader and writer threads under MVCC with optimistic concurrency. Across processes, either one process wrote, or several processes opened the file read-only. Multi-process writes needed application-level design.

**Workarounds people used:**
- Their own RPC or web services in front of DuckDB.
- Arrow Flight SQL front ends such as GizmoSQL.
- MotherDuck's hosted service with its own protocol.
- Moving to Postgres, or running DuckDB inside it with pg_duckdb.
- DuckLake with a Postgres catalog for many instances reading and writing shared data.

**How Quack differs:**
- **Quack** is an open, self-hosted protocol that turns any DuckDB process into a server for its own database. You operate, secure and scale it yourself.
- **DuckLake** is a lakehouse table format: data in Parquet on object storage, metadata in a catalog database such as Postgres. Many independent DuckDB instances coordinate through that catalog, with no single DuckDB server. The 2.0 docs still name DuckLake 1.1 with a Postgres catalog as the production route for multi-instance read-write. Quack can now also serve as a DuckLake catalog.
- **MotherDuck** is a managed commercial cloud service, with its own protocol and hybrid execution. The DuckDB team credited lessons from MotherDuck's protocol.

## 7. Reception
I found limited direct evidence, so this is partly unverified.

**Enthusiasm:**
- People built standalone Quack clients within weeks of the preview. The team wrote that "the world said no, no, no, and built their own clients."
- Columnar released an ADBC driver for Quack in public preview.
- The demand had been long-standing: users asked "very persistently," and many had already built their own workarounds.
- The benchmarks looked strong against Postgres and Arrow Flight.

**Skepticism:**
- One Hacker News commenter argued HTTP is bad for large transfers and streaming.
- Commentary framed it as "DuckDB spent 6 years telling you it doesn't need a server, then added one," and worried it could complicate the embedded story.
- Real limitations feed the doubts: it was beta through 1.5.x, security is basic, there is a single server and no replication, and hot-table insert scaling hit a ceiling around 8 threads in the team's own tests.

I didn't find a substantial Hacker News thread about the 2.0 release itself.

## 8. Setting up the sensor-ingestion case
**Setup.**
- Make the ingestion service's process (or one dedicated DuckDB process on the VM) the only owner of the database file. Run `quack_serve` with a strong token you supply.
- Have the ingestion writer insert either in-process or through Quack.
- The dashboard on the same VM attaches to `quack:localhost` with the token, plain HTTP, and reads. No process should open the file directly alongside the server.
- The analysts attach (or `CONNECT`) to `quack:their.domain` over HTTPS.

**Two analysts fixing the same reading.** The server uses optimistic concurrency, so both transactions run without waiting on each other. The first to commit wins, and the second gets a write-write conflict error and has to retry; it will not wait on a lock. Updates to different readings, and ingestion appends, don't conflict.

**Changes from the defaults before exposing it:**
- Keep Quack bound to localhost.
- Put a TLS-terminating reverse proxy such as nginx with a Let's Encrypt certificate in front, and open only port 443 in the firewall. Avoid `allow_other_hostname` with its self-signed certificate, which is meant for testing.
- Store the token in a `quack` secret on each analyst's laptop, and use a proper token rather than sharing the random startup token around.
- Replace the allow-all authorization hook, for example so analysts can only SELECT and UPDATE the readings table, and not DROP or DELETE.
- Ideally give each analyst their own credential through a custom authentication hook, or add an auth layer, VPN or IP allowlist at the proxy.
- Watch the logs.
- Remember the server is a single point of failure, and back up the file.
