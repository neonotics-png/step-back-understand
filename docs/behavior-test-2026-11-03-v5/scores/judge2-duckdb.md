# Judge 2: DuckDB 2.0 and server mode

**Ranking: Y > Z > X > W**

(Word counts: W 3,633, X 3,215, Y 2,138, Z 1,309.)

## 1. Y (best)
Y teaches the subject best. It explains why the file lock exists (in-memory state can't be synced across processes) and why Quack could ship as one extension (an MVCC engine from HyPer that already supported many connections). It also gives the reasoning behind each protocol choice, with the benchmark numbers placed as evidence. The "what to expect" section separates documented behavior from inference, and the "where it sits" section (SQLite, MotherDuck, DuckLake) gives real context. Weakness: it calls 2.0's Quack "a production version", which W contradicts with the FAQ's "not yet production-ready" and Z hedges on. That reads as an overclaim. A few of the rest-of-2.0 bullets are list-dense.

## 2. Z
Z gives the most understanding per minute. It says outright that Quack "is not a new concurrency engine, it is a new front door", and it has a clear "CONNECT is bigger than Quack" section, including the useful point that psql can't connect because DuckDB doesn't speak the Postgres wire protocol. It is honest about what is inferred and doesn't claim Quack is stable. Weaknesses: it says the server "doesn't do TLS itself". W, X and Y all say 2.0 defaults to HTTPS with a self-signed certificate for non-local binds, so that looks like stale beta information. The "check tokens against LDAP" macro example is doubtful, since the other answers say macros can't do DML or keep state. It is thinner on the rest of 2.0 and on migration.

## 3. X
X is a thorough reference with better framing than W: a bottom-line section, a MotherDuck/Postgres positioning section, a discrepancies-across-sources section and a clear limitations list. For an operator it is excellent. As an explainer it is mostly enumeration (option tables, function lists), so the "why" is brief compared with Y and Z. Possible errors: it says Quack graduated "to stable" / 1.0, which conflicts with W's FAQ claim. "Quack was beta until four months before GA" doesn't fit its own timeline (1.0 at the 2026-09-02 alpha, about 7 weeks before GA). Its CSV size (81 GB) also differs from W's (76 GB).

## 4. W (worst, but still accurate and useful)
W is the most exhaustive: every option, setting, proxy gotcha, breaking change and Python API change, plus deployment checklists. It is also the most careful about maturity, flagging the FAQ's "not yet production-ready" and the HN critique of the vendor benchmarks. But it is a reference manual, not an explanation. The "why" gets one short section, and the rest costs a lot of reading time for someone trying to understand the subject. Small internal tension: it cites HTTP/2 from the FAQ, while its own proxy recipe relies on HTTP/1.1 keep-alive.

## Cross-answer conflicts to note
- **Is Quack production-ready in 2.0?** W says no (FAQ, beta). X says stable/1.0. Y says "production version". Z says the pages it read don't clearly call it stable. This is the most important disagreement and isn't resolved.
- **TLS:** Z says no built-in TLS. The others say HTTPS with a self-signed certificate is the non-local default in 2.0, and Y says TLS was added after the beta.
- **All performance numbers** are DuckDB's own beta benchmarks (May 2026). Only W and X point out that no independent benchmarks exist.
