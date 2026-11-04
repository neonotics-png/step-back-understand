# Third behavior test: delivery and calibration rules, 2026-11-02

**Tested revision:** c33d74d (SKILL.md md5 `faf6d848…`), which adds the Delivery section (lead with the explanation, teach each connection once, supporting detail after) and the "uncertainty survives synthesis" rule. Three fresh skill runs on the same three subjects. The no-skill baselines are reused from the [second test](./behavior-test-2026-11-02-v3.md), since nothing changed on that side. Raw outputs are in [`behavior-test-2026-11-02-v4/`](./behavior-test-2026-11-02-v4/).

Grading: nine fresh model graders, three per pair, A/B order re-randomised from the last round, method name stripped. Same five criteria as last round plus a sixth, **calibration**: are the opening, synthesis and takeaway no more certain than the evidence the answer itself presents? Rubric, key and raw scores are in [`blind-grading/`](./behavior-test-2026-11-02-v4/blind-grading/).

These are subjective model-grader scores on three answer pairs. Nine graders are not nine independent test pairs, and none of this measures human understanding.

## Scores

| Criterion | With skill | Without | Change | Last round's change |
| --- | --- | --- | --- | --- |
| Teaches the sources | 7.9 | 7.2 | +9% | +32% |
| Reveals the subject | 8.3 | 7.2 | +15% | +12% |
| Distinctiveness | 7.8 | 6.2 | +25% | +26% |
| Support | 9.1 | 6.4 | +41% | +38% |
| Readability | 6.4 | 8.8 | -27% | -35% |
| Calibration (new) | 8.6 | 5.2 | +64% | not graded |

Preference: 7 of 9 for the skill answer. Khruangbin 3 of 3, uv 3 of 3, TDD 1 of 3.

The without-skill scores moved a little between rounds although the answers are identical. That is grader noise, and it is about the size of some of the differences above.

## What the revision fixed

**Calibration.** This was the author's concrete complaint, and it is the clearest result in the test. The uv answer now says, in the explanation itself, "The sources do not say that uv's lockfile is Cargo's lockfile". It goes on to find that uv's own resolver docs credit the forking technique to Poetry. It concludes that "the lockfile's purpose reads as Cargo's, while the mechanism is Poetry's and the format is uv's own. That is a comparison on my part, not a statement from Astral about lineage." The Khruangbin answer drops the last round's unsupported idea that Speer's tone copies Thai guitarists. It qualifies each inference in the sentence that makes it ("That last link is interpretive; no member states it in those terms"). Every grader, unprompted, cited the baselines' flat inferences as a weakness, for example "is going to play like a sampled break" and the gear as "a recipe for" a cassette tone.

**Structure.** All three answers lead with an explanation section and put the supporting detail after it under its own heading. Total length fell by 18 to 31 percent.

| Subject | Last round | This round | Explanation section |
| --- | --- | --- | --- |
| Khruangbin | 4,875 words | 3,358 | 2,106 |
| uv | 4,095 | 3,368 | 2,243 |
| TDD | 3,579 | 2,550 | 1,145 |

## What it did not fix, and what it broke

**Readability is better but still well behind.** The gap narrowed from 35 to 27 percent. That is an improvement, not a resolution. The supporting detail still carries a full connections table in all three answers, and graders still described the skill answers as "dense" and "table-heavy".

**TDD over-corrected.** That run cut its explanation to 1,145 words. Two of three graders preferred the plain answer because it "explains each influence much more fully", and "Teaches the sources" fell for the skill answer. Selectivity removed teaching, not only repetition. The Delivery rule says the explanation teaches the five connections that matter. It does not say the teaching has to stay at full depth inside that selection, and this run read it as permission to be brief.

## Verdict on the author's acceptance test

- **Uncertainty survives synthesis:** met on this sample.
- **Easier to follow:** partly met. The direction is right, but the gap is still large.
- **Still teaches the influences:** met for Khruangbin and uv. On TDD it regressed against the baseline.

The branch is not ready by the author's standard. Next revision: say that selectivity applies to how many connections are taught and how often, not to how deeply each chosen source is taught. Limit the supporting-detail table to connections that are not already taught in the explanation, or move it to a separate file. Then rerun and re-grade.
