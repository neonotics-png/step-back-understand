# Step Back Understand

**To understand a thing, learn the things that made it what it is.**

You can read about John Mayer and know John Mayer. Learn Stevie Ray Vaughan, Jimi Hendrix, B.B. King and Albert King as players in their own right, and you hear Mayer differently. This agent skill does that for anything: a person, a work, an idea, a product, a system, a company. It finds what shaped the subject, studies each of those sources as a subject in its own right, and returns with what you can now recognize in the original.

## The loop

1. Inspect the subject itself.
2. Identify what shaped it.
3. Learn each source as a subject in its own right, not merely its name.
4. Trace further back only where it improves understanding.
5. Return to the original subject and explain what the connections reveal.

Depth is flexible: follow a branch while it materially improves understanding, keep the original subject visible, and stop when the explanation is sufficient or evidence runs out. Deeper understanding can be the result; it does not have to justify an immediate action. Connections require evidence: resemblance alone does not prove influence, and every connection in the output is labelled `documented`, `composition`, `interpretive` or `unknown`, with a source.

## Install

Clone this repository:

```sh
git clone https://github.com/neonotics-png/step-back-understand.git
```

Copy the [`step-back-understand`](./step-back-understand) directory into your agent's skill directory. The portable entry point is [`SKILL.md`](./step-back-understand/SKILL.md).

| Runtime | Path |
| --- | --- |
| Claude Code (personal) | `~/.claude/skills/step-back-understand/SKILL.md` |
| Claude Code (one project) | `<project>/.claude/skills/step-back-understand/SKILL.md` |
| Codex | `~/.codex/skills/step-back-understand/SKILL.md` |
| Shared repo | `skills/step-back-understand/SKILL.md` |

Other runtimes differ; check where they discover `SKILL.md` files.

## Invoke

The skill runs when invoked by name, or when you explicitly ask to understand a subject through what shaped it. In Claude Code, `/step-back-understand` works once installed. In any runtime:

```text
Use step-back-understand to explore what shaped [subject].
```

For example:

```text
Use step-back-understand on John Mayer's guitar playing. Where do the
bends and the chord-melody rhythm style come from, and what is his own?
```

The result leads with an explanation that teaches the most revealing sources and what they show about the subject, each connection taught once, with inference qualified in the prose and not only in a table. Supporting detail follows: what was inspected, every connection with its evidence class and source, and what is unresolved. Research depth and presented length are separate; the explanation stays selective however deep the investigation went. Two abbreviated examples, one in prose and one in the template, are at the end of [`SKILL.md`](./step-back-understand/SKILL.md).

## Which step back?

- **Step Back Understand:** study what shaped a subject, even when nothing has gone wrong.
- **[Step Back Protocol](https://github.com/neonotics-png/step-back-protocol):** use `step-back-evaluate` to reframe stalled work and repeated failed approaches.

This skill stands alone; the recovery skill is not a dependency.

## Status

v3 procedure: the influences are the agent's research, and the answer explains the subject itself (how it works, why it is that way, its context). Best on subjects the agent does not already know well.

Tested 2026-11-03 on three subjects past the model's training cutoff or niche (a database release, a pop album, a folk album), two runs per arm. An independent quiz writer, never told the skill exists, wrote eight questions per subject; each agent answered from what it had learned, with no tools, and a blind scorer marked the sheets. Arms: with the skill, plain, and plain told to "research thoroughly" (matched effort).

- What the agent learned: skill 91/96, thorough plain 93/96, plain 72/96. The skill's gain over a plain request comes from more research, not from its method.
- What the person reads: blind judges, asked which answer they would rather receive, ranked the two skill answers first and second in all three subjects, ahead of the thorough-plain answers, at about half the length.

An earlier revision that made the influences the answer lost to a plain answer on six well-known subjects ([fair test](./docs/behavior-test-2026-11-03-fair.md)); this revision is the fix ([v5 test](./docs/behavior-test-2026-11-03-v5.md)). All graders are models of the same family as the writers; small samples; initial evidence, not a guarantee. Cost: roughly twice the research of a plain answer. Publishing or cloning this repository does not activate it in an agent runtime.

## License

[MIT](./LICENSE)
