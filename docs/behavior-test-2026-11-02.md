# Behavior test, 2026-11-02

Three subjects, each run twice by a fresh general-purpose agent with web search and fetch: once told to read and follow `SKILL.md` (revision 54b953c, the one before the "learn the sources" rewrite), once given only the question plus "explain it through what shaped it". Raw outputs are in [`behavior-test-2026-11-02/`](./behavior-test-2026-11-02/). Judged by one reader on whether the run studied the shaping sources and explained something concrete about the subject, not on whether it filled the template.

| Subject | With skill | Without skill |
| --- | --- | --- |
| Khruangbin (musician) | 23 sources read, 54 tool calls, 104k tokens | 14 sources read, 32 tool calls, 77k tokens |
| uv (product) | 23 sources read, 41 tool calls, 88k tokens | 20 sources read, 34 tool calls, 80k tokens |
| Test-driven development (idea) | 25 sources read, 75 tool calls, 133k tokens | 16 sources read, 58 tool calls, 106k tokens |

## What the skill changed

1. **It went to the sources' own material.** With the skill, the musician run read the Thai cassette blog the band learned from and the blog's Dao Bandon posts, not just the band's interviews about it. The product run read uv's `Cargo.toml`, the `uv-git` crate header crediting Cargo, a 2023 commit explaining the reflink cache, and the Dart `pub` solver design doc behind PubGrub, which took it a layer further back to conflict-driven clause learning. The idea run read Beck's own book front matter from the publisher PDF, Cunningham's 1992 WyCash report, and the Larman and Basili paper. Without the skill, all three runs stayed mostly on secondary accounts of the subject.

2. **Every claim carried an evidence class and a source, and the gaps were disclosed.** The with-skill runs said which pages were paywalled or blocked and cited nothing from them. The without-skill runs listed sources at the end but presented search-snippet material and Wikipedia details inline as fact (the church name and year in the Khruangbin piece, several uv claims from a GitHub issue thread). The musician run without the skill also added "surf and spaghetti-western" with no source; the with-skill run flagged the same idea as interpretive at best.

3. **Both found the same factual correction** on the idea subject: SUnit is 1994, not Wikipedia's 1989. The skill did not make the agent smarter; it made it show its work.

4. **The without-skill outputs are the better reads.** Narrative sections, a one-paragraph answer, a timeline table for uv. The with-skill outputs are dense tables. All three with-skill runs chose the template even though that revision had already made it optional, so the template's pull is strong; the rewrite's "choose the form that explains best" line may need to be stronger.

5. **Cost:** the skill runs used roughly a quarter to a third more tokens and tool calls, almost all of it extra fetching.

## What the skill did not do, and the rewrite targets

Neither version profiled a shaping source as a subject in its own right. Every connection row reads "source, what the subject says it took, quote from the subject". The source's own signature traits are rarely studied: what dub actually is beyond a Wikipedia sentence, who played bass on the Gainsbourg record, what Albert King-style playing would be in Khruangbin's case. The one exception is the musician run's row on the blog itself. That is the gap the 25a7401 rewrite addresses with "profile each source, then map the traits forward". It is untested.

## Verdict

Keep the evidence discipline and the "say what you could not inspect" rule: they are the measurable difference. The rewrite's "learn each source as a subject in its own right" is the right next thing to test. Status stays behavior-tested once, on the previous revision.
