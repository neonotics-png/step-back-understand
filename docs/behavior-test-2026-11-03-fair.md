# step-back-understand fair test, 2026-11-03

Skill revision e94d773 (SKILL.md md5 faf6d848). Six subjects, none in the skill's examples. Plain request "Help me really understand X: <aspect>" vs the same request + "Use step-back-understand" with SKILL.md loaded. Both arms use web. One run per arm.

Measurement: (1) independent quiz writer per subject (8 expert questions + key, written before seeing any answer, told to cover what an expert considers most important, no slant); (2) fresh reader per answer, sees only the blinded answer (X/Y, random order), answers the quiz with no web; (3) blind scorer marks both sheets 0/1/2 per question against the key without seeing the explainers; (4) separate blind holistic judge reads both explainers as the asker and picks one.

| Subject | Quiz with | Quiz without | Holistic |
| --- | --- | --- | --- |
| Tailwind CSS | 13/16 | 15/16 | without, clear |
| TDD | 13/16 | 14/16 | without, strong |
| uv | 13/16 | 15/16 | without, clear |
| The Wire | 12/16 | 16/16 | without, clear |
| Rust borrow checker | 15/16 | 15/16 | without, strong |
| Jacob Collier | 10/16 | 13/16 | without, clear |
| **Total** | **76/96 (79%)** | **88/96 (92%)** | **without 6 of 6** |

Quiz: without won 5, tied 1. Holistic: without preferred 6/6.

Every holistic judge gave the same reason: the skill answer is a genealogy of the subject's influences with evidence labels and hedges; the plain answer explains how the subject itself works. Where the skill answer won individual quiz questions, they were design-rationale questions (e.g. Tailwind q1, Wathan's dependency-direction argument).

Caveats: one run per arm; model readers carry prior knowledge (Rust with-skill reader marked 5 questions "not covered" but answered correctly anyway, which inflates both arms on well-known topics); same model family throughout; quiz writers are models.
