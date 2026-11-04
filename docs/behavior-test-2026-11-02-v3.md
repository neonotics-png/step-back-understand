# Behavior test of the revised procedure, 2026-11-02

**Tested revision:** 5fd666a (SKILL.md md5 `a7b802d6…`), the one with "learn each source as a subject in its own right", the explanation-first report, and the Albert King example. Same design as the [first test](./behavior-test-2026-11-02.md): three subjects, each run once by a fresh general-purpose agent with web search and fetch, once told to read and follow `SKILL.md`, once given only the question plus "explain it through what shaped it". Raw outputs: [`behavior-test-2026-11-02-v3/`](./behavior-test-2026-11-02-v3/).

Judged on the five criteria from the review: does the answer teach something concrete about the shaping sources themselves, show what that knowledge reveals in the original, preserve the original's distinctiveness, support its specific claims, and stay readable. One reader, who also wrote the revision, so read the verdict with that in mind. One run per cell, so this is initial behavioral evidence, not a guarantee.

| Subject | With skill | Without skill |
| --- | --- | --- |
| Khruangbin (musician) | 40 sources read, 87 tool calls, 156k tokens, 10 min | 28 sources read, 56 tool calls, 108k tokens, 6.5 min |
| uv (product) | 36 sources read, 56 tool calls, 112k tokens, 8 min | 17 sources read, 28 tool calls, 76k tokens, 3.5 min |
| Test-driven development (idea) | 26 sources read, 73 tool calls, 141k tokens, 9 min | 17 sources read, 56 tool calls, 104k tokens, 6.5 min |

## 1. Does it teach the sources themselves?

**With the skill, yes, in all three.** This is the change from the first test, where no run did it.

- *Khruangbin.* A section per source: what luk thung and "wong shadow" are and where they came from; Hank Marvin's setup and style, as the layer behind Thai shadow music; Scientist's path from building amplifiers to King Tubby's assistant to Channel One, with his own words on reverb and bass; Googoosh's pre-1979 records and the ban; DJ Screw; the barn and the Deluxe Reverb. Each ends with "what it did differently".
- *uv.* Cargo read from the Cargo Book: the manifest/lock division of labour, "check `Cargo.lock` in", and the trait that matters, that Cargo's resolver "ignores the platform or `cfg` expression". pnpm from its motivation page. PubGrub from the Dart `pub` solver doc. Rye's philosophy page. python-build-standalone's own docs. Posy's README.
- *TDD.* The tape-era checkout method, with McCracken 1957, Weinberg and Project Mercury as its context; the Smalltalk workspace habit; the actual text of Beck's 1994 SUnit paper, with the observation that it never says to write the test first; "Test Infected" and the JUnit bar; the C3-era loops of Jeffries and Wake, which have no refactor step.

**Without the skill, mostly no.** The without-skill runs were a strong baseline this round, with inline links per claim and caveat sections. But their knowledge of the sources is asserted rather than studied: "Dub bass is, by design, the lead instrument" with no source; Cargo described only as uv's stated model; the SUnit paper known through a Cambridge abstract.

## 2. Does it show what that reveals in the original?

With the skill, each run has at least one connection that only exists because a source was studied:

- uv's lockfile is Cargo's "resolve as if all platforms are enabled" carried into an ecosystem that cannot install two versions of a package, which is why uv forks the PubGrub search on markers. The without-skill run explains forking well but as uv's own idea.
- Beck changed the grain: every ancestor works at a bigger unit (a program against a tape, a micro-increment, a class), and "committing whatever sins necessary" only becomes possible when the test is one assertion. The reader can date any description of the loop by whether "refactor" is the third word.
- Khruangbin's guitar as "the space a voice would sit in", with the Shadows' tune-carrying echo guitar and dub's vocal-less version as the two lineages, and a sentence telling the reader how to tell them apart.

## 3. Distinctiveness

Both versions handle this, but the skill runs bound it honestly: "Whether the guitar lines themselves are theirs cannot be settled without listening, which this pass did not do." The without-skill Khruangbin run closes with a confident one-paragraph synthesis instead.

## 4. Supported claims

Every skill run labels its inferences and gives the basis ("That it reached Speer by this route rather than by surf rock generally is my inference, so the trait-level link is interpretive"). Leads not studied are listed as such (Jamerson, the Gainsbourg session bassists, the refactoring lineage). The without-skill runs are well cited this round but still carry unsourced generalizations in the connective prose.

## 5. Readability

Much better than the first test. All three skill runs lead with prose sections per source and use one connections table, instead of a table-first report. The uv table is still long. The without-skill outputs remain slightly easier reads, and shorter.

## Second layer

All three skill runs went one layer further back where it helped, without being told which: the Shadows behind Thai shadow music; Dimsdale and von Neumann behind Weinberg behind the tape-era method; Cargo's resolver semantics behind uv's lockfile. That is the "Vaughan is best understood through Albert King" rule working.

## Cost

Roughly 35 to 45 percent more tokens, 30 to 100 percent more tool calls, and about half again the wall time. Nearly all of it is fetching the sources' own material.

## Verdict

The revised step does what it was written to do, on this sample: the agent learns the influences and the reader comes out able to recognize specific things in the original. Keep the evidence discipline, the inspection limits, and the "what you can now recognize" requirement. Open items: the connections table still has a strong pull; the cost is real and should be stated in the README.

## Blind grading (added later the same day)

The verdict above was one reader who also wrote the revision. To get a number, nine fresh agents each graded one subject's pair without knowing which answer used the skill: answers were copied to neutral filenames as A and B, order randomised per pair, and the words "skill" and "step-back" removed. Each grader scored both answers 0 to 10 on the five criteria and named a preference. Rubric, key and raw scores are in [`blind-grading/`](./behavior-test-2026-11-02-v3/blind-grading/). Three graders per pair, nine in all.

| Criterion | With skill | Without | Change |
| --- | --- | --- | --- |
| Teaches the sources | 8.7 | 6.6 | +32% |
| Reveals the subject | 8.4 | 7.6 | +12% |
| Distinctiveness | 8.1 | 6.4 | +26% |
| Support for claims | 9.2 | 6.7 | +38% |
| Readability | 5.9 | 9.0 | -35% |
| All five | 8.1 | 7.2 | +11% |
| First four (what the skill is for) | 8.6 | 6.8 | +27% |

Per subject, all five criteria: uv +26% (3 of 3 graders preferred the skill answer), Khruangbin +8% (3 of 3), TDD +2% (2 of 3). Overall preference: 8 of 9 for the skill answer. The one dissent, on TDD, said the plain answer taught the same sources "at least as vividly inside a narrative a reader would actually finish".

Reading: on these three answer pairs, model graders scored the skill answers about a quarter higher on understanding and evidence and about a third lower on readability. These are subjective model-grader scores on three pairs, not a measured improvement in human understanding, and nine graders are not nine independent test pairs. The readability loss is the next thing to fix; the explanation-first wording did not stop any skill run from producing a long connections table. Graders are the same model family as the writers, and nine grades over three pairs is a small sample.
