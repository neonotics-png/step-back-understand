# step-back-understand v5: task-based test (2026-11-03)

Skill under test: SKILL-v5.md (influences are research input; answer explains the subject; lineage -> why/context/predictions; for unfamiliar subjects).

Subjects past the model's cutoff or niche: DuckDB 2.0 server mode (Oct 2026), ADELA "PRIMA" (Sep 2026), Mon Rovia "Bloodline" (2026).

Design
- Independent quiz writer per subject (did not know the skill exists): 8 questions (3 how, 2 why, 2 context, 1 apply), key with sources and 0/1/2 rubric.
- Per subject: 2 runs with skill, 2 plain. Same request ("Help me understand X"), web allowed.
- After studying, the SAME agent got the quiz and answered with no tools except Write (tests what the agent learned).
- Blind scorer per subject (sheets shuffled A-D). Blind judge per subject ranked the four study write-ups as the asker.

Quiz results (out of 32 per arm per subject)
| Subject | With skill | Plain |
| DuckDB 2.0 | 32 (16,16) | 27 (15,12) |
| ADELA PRIMA | 29 (15,14) | 23 (15,8) |
| Mon Rovia | 30 (14,16) | 23 (12,11) |
| Total | 91/96 (95%) | 73/96 (76%) |

Holistic judges: skill write-ups ranked 1st and 2nd in all 3 subjects. Reasons given: better "why" and context, clearer about what is inference; plain ones read as fact sheets/reference cards.

Compare previous round (old skill, famous subjects, reader-from-answer): plain 88/96 vs skill 76/96, judges preferred plain 6/6.

Caveats
- Effort confound: skill runs used ~17-42 tool calls, plain ~12-19. Part of the gain may be "the skill makes the agent research more". Next: effort-matched control (plain + "research thoroughly, take your time").
- 2 runs per arm, 3 subjects; same model family for every role; one quiz writer per subject; quiz includes why/context questions by design (Nate's stated goal).
- One plain ADELA run (8/16) missed four topics; without it ADELA is 29 vs 30-ish.
- Scorer flagged 3 borderline calls in Mon Rovia; none flip the result.

## Effort-matched control (added same day)
6 more plain runs told "research thoroughly, 25+ searches and page reads" (no skill idea in prompt). They used 46-73 tool calls. All 6 sheets per subject rescored together by one fresh blind scorer.

| Subject | Skill | Thorough plain | Plain |
| DuckDB | 32 | 32 | 26 |
| ADELA | 29 | 31 | 23 |
| Mon Rovia | 30 | 30 | 23 |
| Total /96 | 91 | 93 | 72 |

Blind judges, skill vs thorough write-ups (4 per subject, "which would you rather receive, weigh understanding per reading time"): skill ranked 1st and 2nd in 3/3 subjects. Thorough write-ups were 2,700-3,600 words and read as dossiers; skill write-ups 1,200-2,100 words, built around why and context.

Conclusion
- What the agent learns: the skill adds nothing over "research thoroughly". The earlier quiz gain was research effort.
- What the person gets: the skill's answer is better and about half as long at equal or better accuracy. That is the skill's real, measured value.
