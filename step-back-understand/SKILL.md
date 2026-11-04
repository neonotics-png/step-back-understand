---
name: step-back-understand
description: "Understand a person, work, idea, product, system or anything else, including why it is the way it is and the context around it, by studying the things that shaped it and then seeing the subject through them. Strongest on subjects you do not already know well: new releases, niche people and works, unfamiliar codebases, internal systems. Use when invoked by name or when the user explicitly asks for this kind of understanding."
---

# Step Back Understand

You can read about John Mayer and know some facts about John Mayer. Learn Albert King, Stevie Ray Vaughan and Jimi Hendrix as players in their own right, and Mayer's playing starts to make sense: you know whose bends those are, why he plays the way he does, and what is his alone. That is the goal: a better understanding of the subject itself, how it works, why it is that way, and where it sits among the things around it.

The influences are how you get there. They are your research, not your answer. The person asking wanted to understand Mayer, not to read an essay about Albert King.

## When it earns its cost

- **Strong:** subjects you do not already know well (anything recent, niche, internal or new to you). Recognizing what something descends from lets you carry over everything you know about the ancestor: "this is built like Cargo" tells you how a new tool will behave before you have read its docs.
- **Strong:** questions about why: why it was designed this way, why it sounds this way, why people reacted to it as they did.
- **Weak:** a famous subject where a plain explanation is what is wanted. You already absorbed its lineage in training; a genealogy adds length, not understanding.

## 1. Look at the subject first

Read, inspect or examine the subject itself (its docs, code, the work, its own statements) before tracing anything backward. Write down what it is, how it works, and what about it you cannot yet explain. Those unexplained features are what the step back is for.

## 2. Find what shaped it

For each unexplained feature, ask what it came from:

| Subject | Where to look |
| --- | --- |
| Person | teachers, heroes, scenes, collaborators, formative jobs, tools they grew up with |
| Work | earlier works it answers or imitates, genre conventions, the maker's prior work, how it was made |
| Product or system | predecessors, the ideas and papers it inherits, its dependencies, the constraints it was built under, what it was built to replace |
| Concept or method | the problem it answered, the field it came from, the idea it reacted against |
| Organization | founders' previous companies, the gap it filled, what it copied or rejected |

Prefer sources the subject or its makers name. Pick for explanatory value, not fame.

## 3. Learn the important sources in their own right

A name on a list teaches nothing. For the two to five sources that explain the most, learn what each is actually like: its signature traits, how it works, what it did differently from what came before. Then ask which of those traits show up in the subject, and what the subject changed.

Keep private notes of what you checked and how solid each link is (stated by the maker, an observable part or dependency, or only a resemblance). Check that each source supports the specific claim, not just the topic. Never claim to have heard, watched or run something you did not.

Go a layer deeper only when a source is itself best understood through its own sources (Vaughan through Albert King). Stop when the subject's features are explained.

## 4. Turn the lineage into understanding of the subject

Write down, for yourself:

- **What it is and how it works**, now explained rather than listed.
- **Why it is this way**: which features came from where, and what the subject did differently.
- **What to expect**: predictions the lineage gives you about things you have not checked. "It descends from X, so it probably also does Y." Mark these as expectations until verified, and verify the important ones.
- **Context**: what came before and around it, what it reacts against, how it was received and why.
- **What is its own**: what the sources do not explain.

## 5. Answer about the subject

The answer explains the subject. The person should finish it understanding the subject better, not knowing more about its ancestors.

- **Lead with the subject:** what it is, how it works, why it is this way, and its context.
- **Use an influence only where it makes a specific feature click,** in a sentence or two at that point: "his bends overshoot the note and come back down the way Albert King's did, which is why they sound vocal rather than precise." No section per influence, no family tree.
- **Teach each point once.**
- **Keep uncertainty in plain words** where it matters ("the maker hasn't said this; it's a resemblance"). No evidence-class labels in the answer.
- **End with a short source list.** Your full research notes stay out of the answer unless asked for.
- **Length follows the question.** The research can be deep while the answer stays short.

## Check before sending

Send it back for another pass if any of these is true:

- A reader would learn more about the influences than about the subject.
- It is organized by influence rather than by feature of the subject.
- A claim is more certain than its evidence.
- It contains nothing a plain explanation of the subject would not have said. If the step back found nothing, say so briefly rather than padding.

## Which step back

Use this skill for understanding. Use `step-back-evaluate` (Step Back Protocol) for stalled work and repeated failed approaches; if that exposes a missing understanding of the subject, run this skill and return to the recovery.

## Example: what the answer looks like

Request: "Help me understand John Mayer's lead guitar playing."

> Mayer's lead playing is blues vocabulary played with a pop songwriter's restraint. Three things carry it.
>
> **The bends.** His signature move is a wide bend, often past a whole step, that lands on the target note and sometimes catches a second string on the way. That is Albert King's move, which reached Mayer through Stevie Ray Vaughan; Mayer has said Vaughan was the first player he heard who was "not just strumming chords." Knowing this tells you what to listen for: a King-style bend pulls past the note for tension, where a B.B. King bend is a smaller, finger-vibrato phrase.
>
> **Space.** Where Vaughan fills every bar at full volume, Mayer leaves gaps and plays under the vocal. ...
>
> **Rhythm as lead.** ... (critics compare this to Hendrix; Mayer hasn't described it that way himself, so treat it as a resemblance.)
>
> Sources: ...

The research behind this answer covered Albert King's upside-down Flying V and pull-down bends, Vaughan's adoption of them, and Mayer's own statements. The answer uses that research only where it explains Mayer.
