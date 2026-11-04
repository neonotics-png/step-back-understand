# DuckDB / Quack quiz: blind scores (six sheets)

| Sheet | Q1 | Q2 | Q3 | Q4 | Q5 | Q6 | Q7 | Q8 | Total |
|---|---|---|---|---|---|---|---|---|---|
| A | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 16 |
| B | 2 | 2 | 2 | 0 | 1 | 1 | 1 | 2 | 11 |
| C | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 16 |
| D | 2 | 2 | 2 | 2 | 2 | 2 | 1 | 2 | 15 |
| E | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 16 |
| F | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 2 | 16 |

## A
- Q1 (2): Both ends are DuckDB with quack; ATTACH = catalog/qualified names, CONNECT = session routing until DISCONNECT.
- Q2 (2): HTTP, client-initiated, application/duckdb WAL serialization, one round trip plus parallel FETCH. (The HTTP/2 aside is immaterial.)
- Q3 (2): Localhost bind, auto token, no-op authorization so any SQL runs; auth per connection vs authz per query; SQL macro/UDF customization.
- Q4 (2): HTTP tooling plus Wasm; Flight: external control and 2 round trips; PG: row-based, no parallel reads.
- Q5 (2): In-memory state sync problem, and reuse of the existing in-process MVCC machinery.
- Q6 (2): Single writer / many read-only model; many real workarounds; DuckLake (scale, format, catalog) and MotherDuck (managed features) contrasted.
- Q7 (2): Positive with reasons (HN quotes, ADBC driver, MotherDuck) and skepticism with reasons (HTTP bulk, benchmarks, "concurrent writers", scope).
- Q8 (2): Single server owns the file; second same-row update conflicts and retries; reverse proxy/TLS, per-analyst tokens, real authorization.

## B
- Q1 (2): Server and clients are DuckDB processes; ATTACH vs CONNECT/DISCONNECT distinguished correctly.
- Q2 (2): All four elements: HTTP, client-initiated, native WAL serialization, one round trip plus parallel FETCH.
- Q3 (2): Localhost, token, permissive nop authorization; authn vs authz hooks with SET GLOBAL customization.
- Q4 (0): Only an HTTP reason; explicitly gives no reasons against Flight SQL or the PG protocol (below the two-of-three bar for 1).
- Q5 (1): Ties the server to the existing MVCC machinery but gives no in-memory state synchronization reason.
- Q6 (1): Prior model right, but workarounds are guessed and the DuckLake/MotherDuck comparisons are only thin labels ("no source compared").
- Q7 (1): Little real reception; lists early clients and a storage-format caveat without real practitioner reasons for either side.
- Q8 (2): Single owner with clients; correct conflict-and-retry prediction; localhost plus TLS proxy, per-analyst authn, non-permissive authz.

## C
- Q1 (2): Both DuckDB; ATTACH as qualified catalog vs CONNECT session routing until DISCONNECT.
- Q2 (2): HTTP, client-driven, application/duckdb WAL serialization (not Arrow), one round trip plus parallel FETCH.
- Q3 (2): Token and localhost defaults, allow-all authorization; authn vs per-query authz, customizable via SQL macros.
- Q4 (2): HTTP tooling plus Wasm; Flight control and 2 round trips; PG row-oriented, no parallel reads.
- Q5 (2): In-memory state sync problem plus reuse of the day-one MVCC/multi-connection engine.
- Q6 (2): Correct prior model; RPC, GizmoSQL, MotherDuck, pg_duckdb, DuckLake workarounds; substantive DuckLake and MotherDuck contrasts.
- Q7 (2): Enthusiasm with reasons (clients, ADBC, long-standing demand) and skepticism with reasons (HTTP for transfers, scope, limits).
- Q8 (2): Single owner; first commit wins, second conflicts and retries, appends fine; proxy/TLS, authz replacement, per-analyst credentials.

## D
- Q1 (2): Server and clients are DuckDB processes; ATTACH/qualified vs CONNECT/DISCONNECT correct.
- Q2 (2): HTTP, client-initiated, native WAL serialization, single round trip plus parallel FETCH.
- Q3 (2): Localhost, auto token, permissive nop authz; authn at connect vs authz per query; macro or extension customization.
- Q4 (2): HTTP tooling plus Wasm; Flight external control and 2 round trips; PG no parallel reads and slow bulk.
- Q5 (2): In-memory state sync problem plus reuse of the existing MVCC/optimistic concurrency.
- Q6 (2): Correct prior model; real workarounds; DuckLake contrast solid; MotherDuck as managed vs self-hosted is a substantive (if thin) contrast.
- Q7 (1): Admits little practitioner evidence; enthusiasm is speculative ("plausibly") and skepticism only about beta stability; no real reasoned reception.
- Q8 (2): Single server; conflict error and retry, not merged; localhost plus TLS proxy, per-analyst tokens, real authorization rules.

## E
- Q1 (2): Both ordinary DuckDB; ATTACH catalog with qualified names vs CONNECT session routing until DISCONNECT.
- Q2 (2): HTTP, client-initiated, native WAL vector serialization, one round trip plus parallel fetches.
- Q3 (2): Localhost, auto token, no users and allow-all authz; authn at connect vs authz per query; macro/UDF customization.
- Q4 (2): HTTP tooling plus Wasm; PG row-oriented, no parallel reads; Flight 2 round trips and external control.
- Q5 (2): In-memory state sync problem; remote clients become connections to the existing MVCC transaction manager.
- Q6 (2): Correct prior model and workarounds; DuckLake (shared Parquet via catalog) and MotherDuck (managed features) contrasts.
- Q7 (2): Positive with reasons (single-writer pain solved, ADBC, MotherDuck) plus skepticism with reasons (single node, security, OLTP ceiling).
- Q8 (2): Single server; second commit fails with a conflict and retries, appends fine; TLS proxy, per-analyst tokens, statement-type authz.

## F
- Q1 (2): Both DuckDB with quack; ATTACH catalog/qualified vs CONNECT session routing until DISCONNECT.
- Q2 (2): HTTP(S), client-initiated, application/duckdb WAL serialization with no Arrow conversion, one round trip plus parallel FETCH.
- Q3 (2): Token, localhost, self-signed defaults; full access for token holders; authn per connection vs authz per query with customization.
- Q4 (2): HTTP tooling plus Wasm; Flight control and 2 round trips; PG slow bulk and no parallel reads.
- Q5 (2): Hard-to-sync in-memory state plus reuse of the existing MVCC machinery.
- Q6 (2): Correct prior model and workarounds; substantive DuckLake and MotherDuck contrasts (including plans to support Quack).
- Q7 (2): Positive with concrete reasons and ecosystem examples; skepticism with reasons (simplicity tension, OLAP not OLTP, limits).
- Q8 (2): Single owner; first commit wins, second conflicts and retries, appends fine; proxy/TLS, firewall, per-user tokens, authz.
