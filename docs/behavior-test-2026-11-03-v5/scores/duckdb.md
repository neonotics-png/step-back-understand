# DuckDB 2.0 / Quack quiz scores

| Sheet | Q1 | Q2 | Q3 | Q4 | Q5 | Q6 | Q7 | Q8 | Total |
|---|---|---|---|---|---|---|---|---|---|
| A | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 16 |
| B | 2 | 2 | 2 | 1 | 1 | 1 | 1 | 2 | 12 |
| C | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 16 |
| D | 2 | 2 | 2 | 2 | 2 | 2 | 1 | 2 | 15 |

## A
- Q1 (2): Both sides are DuckDB; ATTACH = catalog with qualified names, CONNECT = session routing until DISCONNECT.
- Q2 (2): HTTP, client-initiated, native `application/duckdb` WAL serialization (not Arrow), one round trip plus parallel FETCH.
- Q3 (2): Localhost bind, required/auto token, allow-all authorization, authn vs per-query authz hooks, customizable as SQL macros.
- Q4 (2): HTTP tooling + Wasm; Flight rejected for format control and two round trips; Postgres row-oriented with no parallel reads.
- Q5 (2): In-memory state sync across processes is the obstacle; the server reuses existing in-process MVCC.
- Q6 (2): Correct single-writer/multi-reader model, several real workarounds, substantive DuckLake (catalog + object storage, no server) and MotherDuck (managed service, own protocol) contrasts.
- Q7 (2): Positive with reasons (long-standing demand, third-party clients, ADBC driver) and skepticism with reasoning (HTTP for bulk transfer, scope worries).
- Q8 (2): One server owns the file; second same-row update fails with a conflict and must retry, appends do not conflict; localhost + TLS proxy, real tokens, replace authorization.

## B
- Q1 (2): Server and clients are DuckDB processes; ATTACH vs CONNECT/DISCONNECT distinguished correctly.
- Q2 (2): All four elements: HTTP, client-initiated, native WAL serialization, single round trip with parallel FETCH.
- Q3 (2): Defaults, shared token, permissive `quack_nop_authorization`, hooks distinguished with how to customize.
- Q4 (1): Correct HTTP reason and a generic own-format rationale, but gives no reasons against Flight SQL or the Postgres protocol. Partial credit only.
- Q5 (1): Links the server to existing MVCC machinery but does not give the in-memory state synchronization reason.
- Q6 (1): Prior model correct but workarounds are guessed (Parquet export is not one), and the DuckLake/MotherDuck contrasts are thin ("solves it differently", "managed cloud service").
- Q7 (1): Only one concrete positive (clients within weeks); the "skepticism" is about storage-format upgrades, not Quack, and is not reasoned.
- Q8 (2): Correct architecture, correct conflict prediction (marked as inference), and several hardening steps.

## C
- Q1 (2): Both ordinary DuckDB; ATTACH catalog with qualified names vs CONNECT session routing until DISCONNECT.
- Q2 (2): HTTP, client-initiated request/response, native serialization with no conversion, one round trip with parallel fetches.
- Q3 (2): Localhost, random token, no users, allow-all authorization; authn at connect vs authz per query, customized via SQL macro or UDF.
- Q4 (2): HTTP tooling + Wasm; Postgres row-oriented/no parallel reads; Flight has two round trips and an externally controlled format.
- Q5 (2): In-memory state sync problem plus reuse of the single-process MVCC transaction manager.
- Q6 (2): Correct prior model, real workarounds, DuckLake (shared Parquet + catalog, scale-out) and MotherDuck (managed: users/SSO, scaling, etc.) contrasts.
- Q7 (2): Positive (HN, MotherDuck support, ADBC driver) with reasons, plus reasoned caveats (single node, self-secured, write throughput limits).
- Q8 (2): Single server owning the file, second-committer conflict and retry, appends do not conflict, TLS proxy on localhost, per-analyst tokens, statement-type authorization.

## D
- Q1 (2): Both DuckDB; ATTACH (qualified names, forwarded writes/transactions) vs CONNECT/DISCONNECT, plus `quack_query`.
- Q2 (2): HTTP, client-initiated, native WAL serialization, single round trip with parallel FETCH.
- Q3 (2): Localhost, auto token, permissive `quack_nop_authorization`; authn at connect vs authz per query with customization.
- Q4 (2): HTTP tooling + Wasm; Flight rejected for format control and two round trips; Postgres clients cannot parallelize reads.
- Q5 (2): In-memory state sync is the obstacle; the server funnels writes into the existing MVCC engine.
- Q6 (2): Correct prior model and workarounds; substantive DuckLake contrast; MotherDuck contrast (managed commercial vs self-hosted open) is enough despite coming from general knowledge.
- Q7 (1): Positive side is speculative ("plausibly"), the skepticism is limited to beta stability, and it brings in an unsupported AWS/DuckLabs acquisition claim. Not a sourced account of the reception.
- Q8 (2): Single server, conflict and retry (not merged), localhost + TLS proxy, per-analyst tokens, real authorization rules.
