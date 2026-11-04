# Judge: DuckDB 2.0 and server mode

**Ranking: R > P > Q > S**

## 1. R
Strengths: Best at the "why". It explains the file lock from in-process memory state, shows that the MVCC engine (with its HyPer roots) was already a server inside the process, and walks through why HTTP and DuckDB's own wire format beat the Postgres wire protocol and Arrow Flight SQL. It also places Quack against SQLite, MotherDuck and DuckLake and is honest about limits ("still one machine"). Clearly labels inferred concurrency behavior as an expectation, and flags the C API v2 stability gap between preview and release.
Weaknesses: The longest of the four, and the "rest of 2.0" section runs a bit long. Possible issues: "A Quack server can now act as that [DuckLake] catalog" conflicts with Q, which lists serving DuckLake catalogs over Quack as *planned* work. The InfoQ "duckdb-v2-distributed" source title sits oddly next to R's own "not distributed" point.

## 2. P
Strengths: Tight and very readable, with a clear story arc: "Quack is not a new concurrency engine, it is a new front door." It explains why the feature could ship as an extension, makes the point that DuckDB connects *out* to Postgres and does not speak its wire protocol (helpful context), and says outright which parts of "what to expect" are inferred.
Weaknesses: Likely wrong on TLS. P says the server "doesn't do TLS itself", but Q, R and S all say non-local servers in 2.0 default to HTTPS with a self-signed certificate (R says the no-TLS behavior was the May beta's). Less practical usage detail than Q or S. The `remote.query($$...$$)` workaround naming differs from R's `postgres_query`.

## 3. Q
Strengths: Practical and well organized. It covers three ways a client can use a remote database, secrets for tokens, security defaults with the WITH...INSERT caveat, a benchmark table, and a useful "when to use Quack vs DuckLake vs embedded" section. The upgrade notes are complete.
Weaknesses: The "why" is thinner than in R or P: it never explains that the existing MVCC is what makes this work. In the benchmark table, the "~half of PG" figure for Arrow Flight single-row inserts has no support anywhere else and looks invented. "The officially supported way for many processes or machines to read and write" overclaims a bit, since the 2.0 concurrency docs reportedly still point to DuckLake for production multi-instance use. The AWS acquisition of DuckLabs appears only in Q; it may be right, but no other answer corroborates it.

## 4. S
Strengths: The most complete reference for operating Quack: management functions, max connections, reconnects, logging, the token's minimum length, cert pinning, and the "only meant for testing" quote on `allow_other_hostname`. It is accurate and cautious ("What the docs don't say yet").
Weaknesses: It reads like a changelog or reference card more than an explanation. The "why" gets two sentences and never says *why* only one process could write (in-memory state), so the asker learns what to type but not how or why it works. It leaves out the protocol design rationale (Postgres wire vs Arrow Flight) and the performance context. Long bulleted lists make it the least engaging to read.

## Cross-answer notes
- TLS: P is the outlier, and it is probably stale (beta-era behavior).
- DuckLake catalog over Quack: R says it is already possible; Q says it is planned. Unresolved.
- Benchmarks are vendor numbers from the May 1.5.3 beta. P and R say so clearly; Q says it less clearly.
