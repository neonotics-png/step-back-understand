# DuckDB 2.0 / Quack quiz answers

These answers come only from what I read during the research: the official 2.0 announcement, the August preview, the Quack overview and reference docs, and RuntimeWire and AlternativeTo coverage. Where my sources didn't cover something, I say so.

## 1. Server, client, ATTACH vs CONNECT

The server is one DuckDB process that has loaded the `quack` extension and called `quack_serve(...)` (or was started with `duckdb -serve`). It owns the database and runs the queries. The clients are other DuckDB processes that `ATTACH 'quack:host' AS qk (TOKEN '...')`; their SQL is serialized, sent to the server over HTTP, run there, and the results stream back. With a plain `ATTACH`, the server shows up as one more catalog in your local session, so you name it in queries (`qk.events`) and can mix it with local tables. `CONNECT qk` switches the whole session to the remote database, so unqualified SQL runs on the server until you run `DISCONNECT`. `CONNECT` also works for PostgreSQL and MySQL, where remote pushdown sends the query to that server.

## 2. Transport and round trips

Quack runs over plain HTTP or HTTPS, as POST requests to a `/quack` endpoint on port 9494 by default, so normal proxies, load balancers and firewalls can handle it. The client starts every exchange; the server never pushes. Data is encoded with DuckDB's own internal serialization, the same code path its write-ahead log uses, so complex types come through intact. Once the handshake is done, a query takes one request and one response; large results come back in chunks through follow-up `FETCH` requests, which can run in parallel.

## 3. Default protections, and the auth hooks

By default the server listens on localhost only; binding elsewhere needs `allow_other_hostname = true`, which the docs say is only for testing. Clients must also present a token (at least 4 characters), and non-local servers use HTTPS, though with a self-signed certificate unless you supply one. What the defaults don't protect against:
- **Traffic on localhost is plain HTTP.**
- **One token for everyone:** the token is a single shared secret, with no per-user identity.
- **No authorization:** once a client is authenticated, every query is allowed, because the default authorization hook (`quack_nop_authorization`) always says yes.

There are two separate hooks:
- **Authentication** (`quack_authentication_function`, default `quack_check_token`) answers "is this client allowed in?" It takes `(sid, client_token, server_token)` and returns a boolean.
- **Authorization** (`quack_authorization_function`) answers "may this session run this query?" It takes `(sid, query)` and returns a boolean or a message.

Operators customize either one by writing their own scalar function with that signature and pointing the setting at it with `SET GLOBAL`. Note that restoring the defaults needs `RESET GLOBAL`.

## 4. Why HTTP plus their own format, not Arrow Flight SQL or the Postgres wire protocol

My sources didn't explain this choice, so I can't give the team's stated reasons. What the docs do suggest:
- **HTTP** works with standard proxies, load balancers, firewalls and TLS-terminating reverse proxies.
- **DuckDB's own serialization** keeps every DuckDB type intact, including nested types and VARIANT, and reuses code the engine already has.

Beyond that, any comparison with Flight SQL or the Postgres protocol would be my guess, not something I read.

## 5. Why a server process instead of many processes writing one file

The 2.0 preview argues that DuckDB has had full MVCC and transaction isolation across multiple connections since day one, but only inside one process. A server lets that existing machinery serve many clients for multi-tenant, long-running deployments, instead of building cross-process file locking and coordination. I didn't find a fuller rationale, for example a detailed argument against multi-process file writing, so I won't make one up.

## 6. Before Quack, and how Quack compares with DuckLake and MotherDuck

**Before Quack:** a database file was owned by the one process that had it open for writing. Other processes could at most open it read-only, and as far as I know, not while a writer had it open. My sources don't describe specific workarounds. The usual ones I'd expect are:
- exporting to Parquet
- wrapping DuckDB in your own API service
- using MotherDuck

That list is my inference, not something I read.

**Quack:** a self-hosted, DuckDB-to-DuckDB client-server protocol that comes with DuckDB.

**DuckLake:** a lakehouse table format with a catalog (version 1.1 ships with 2.0). It solves shared storage differently, and my sources don't compare it with Quack in detail.

**MotherDuck:** a managed cloud service. None of my sources compared it with Quack.

## 7. How Quack was received

I have little evidence on this.
- **Enthusiasm:** the preview says developers built standalone Quack clients within weeks of the beta.
- **Coverage:** RuntimeWire and AlternativeTo framed 2.0 as turning DuckDB into a full server.
- **Skepticism:** the only caution I saw was about upgrade risk from the new storage format (pre-2.0 versions can't read files 2.0 creates). The docs' warnings to use a reverse proxy and not expose the server directly also point to rough edges.

I didn't read any substantial practitioner commentary.

## 8. Sensor-ingestion scenario

**Setup:** make one process on the VM the only owner of the database file, ideally the ingestion service itself, which also runs `quack_serve` with a strong token and can keep appending through its own connections. If ingestion runs as a separate process, it should connect as a Quack client rather than open the file directly. The dashboard connects as a local client (`quack:localhost`).

**Exposing it to the internet:** keep the server bound to localhost and put a TLS-terminating reverse proxy in front for the analysts. Don't set `allow_other_hostname`, which is meant for testing only. Before exposing it:
- Use a long, random token, or per-analyst checks through a custom authentication function.
- Replace the allow-everything default with a custom authorization function, for example to limit what the dashboard can run.
- Use a real certificate on the proxy.
- Restrict network access to the proxy.

**Two analysts correcting the same reading:** my sources don't spell this out. My understanding of DuckDB's optimistic MVCC is that both transactions run on the server, the first to commit wins, and the other gets a transaction conflict error and must retry. Ingestion appends and edits to other rows shouldn't conflict. This part is inference, not something I read.
