# Kent Beck's TDD, understood through what shaped it

Sources checked 2026-11-03. I read about the books; I did not read *Test-Driven Development: By Example* (2002) or *Extreme Programming Explained* (1999) themselves. Wording from the book comes from a reader's page-numbered notes, and the full text of Beck's 1994 SUnit column and his 2001 "Aim, Fire" column were not available. The supporting detail at the end lists exactly what was and wasn't read.

**Subject.** Test-driven development as Kent Beck formulated it. Its preface states two rules: "Write new code only if an automated test has failed" and "Eliminate duplication". The loop is "Red/green/refactor – the TDD mantra": write a little test that fails, make it pass "quickly, committing whatever sins necessary", then "eliminate all of the duplication created in merely getting the test to work" (TDDbE preface pp. ix–x, via reader notes on stanislaw.github.io). In 2023 Beck restated it as "Canon TDD": write a test list, turn exactly one item into a runnable test, make it and all previous tests pass, optionally refactor, then repeat.

**Open question.** Why is the loop shaped this way? Why does the test come *first*, why are "make it pass" and "make it clean" two separate steps, and why are the steps so small?

---

## The explanation

Five inputs explain most of the shape. Each one accounts for a different part of the loop.

### 1. The "ancient book": write down the answer before you write the program

Beck's own origin story (Quora, 2012-05-11, quoted on Wikipedia's TDD page) goes like this. "The original description of TDD was in an ancient book about programming. It said you take the input tape, manually type in the output tape you expect, then program until the actual output tape matches the expected output." He says he remembered this only "after I'd written the first xUnit framework in Smalltalk", and tried it then. Older programmers he describes TDD to often say "Of course. How else could you program?", so he calls his role "rediscovering" TDD. *Evidence: documented, from Beck himself. He does not name the book. Gergely Orosz says it was one of his father's tape-to-tape-era books, but that is Orosz's narration, not a quote from Beck.*

**What it is in its own right.** This was batch-era practice. A program turned an input tape (time cards, say) into an output tape (paychecks). The programmer typed the correct output tape by hand before writing any code, and the program was done when its actual output matched that tape.

**What it explains.** The "write the test first" rule is, at bottom, "write down the expected answer first". The test isn't there to check code that already exists. It is a *specification you can run*. Beck says this outright in his 2001 IEEE Software column "Aim, Fire": test-first coding "is not testing". It is "an analysis technique" (you decide what you are and aren't building and what answers you expect) and "also a design technique". He adds that it is "nearly as old as programming" (abstract only; full text not read). Canon TDD says the same about the test list, calling it "analysis, but behavioral analysis".

**How Beck changed it.** The tape held one expected output for the *whole program*, made once. Beck shrinks it to one expected value for *one small behaviour*, and he makes them one at a time. Canon TDD names writing all the tests first as a mistake: "convert all the items on the Test List into concrete tests, then make them pass one at a time." The tape idea is kept. Its single big target becomes a sequence of tiny ones.

**A second, independent precedent.** Larman and Basili's history of iterative development (IEEE Computer, June 2003) reports that NASA's Project Mercury software team in the early 1960s "applied the Extreme Programming practice of test-first development, planning and writing tests before each micro-increment", in half-day iterations. Gerald Weinberg recalls a 1957 IBM project whose technique was "as far as I can tell, indistinguishable from XP". *No source connects this to Beck.* It confirms his claim that the idea is old. It is not his lineage.

### 2. SUnit and the xUnit bar: the tool came before the practice

**What it is in its own right.** Beck described a minimal Smalltalk testing framework in "Simple Smalltalk Testing: With Patterns" (Smalltalk Report, October 1994; reprinted in *Kent Beck's Guide to Better Smalltalk*, 1997). According to the abstract, he argued that Smalltalk lacked a testing culture, and his framework let a test check a block of code with `should:`. Tests were written in the language itself and run inside the IDE. Fowler (bliki "Xunit") says it was built for fast edit-and-test cycles, and that Beck wanted each team to build its own copy. In 1997 Beck and Erich Gamma ported it to Java as JUnit on a flight to OOPSLA, "pair programmed, and done test first" (Fowler). Their 1998 article "Test Infected" describes the JUnit runner: "The bar is initially green but turns into red as soon as there is an unsuccessful test". It also gives the instruction "When you need to add new functionality to the system, write the tests first".

**What it explains.**
- *Order.* By Beck's own account, the framework existed **before** he tried test-first. The loop depends on tests being cheap enough to write and run dozens of times an hour. Typing a whole output tape by hand doesn't allow tiny steps. An in-language test framework does. *Documented (Quora quote above).*
- *Vocabulary.* "Red" and "green" are the colours of the xUnit progress bar. In Beck's book the bar is the unit of progress: reviews describe its "Red Bar Patterns" and "Green Bar Patterns" and its emphasis on keeping the bar green. The C2 wiki's TDD page spells out the mapping ("watch the test fail… the 'Red Bar'… watch all of the tests pass… the 'Green Bar'"). *Composition: the bar is part of the toolchain the practice runs on. I found no statement from Beck that the mantra was **named** after the bar, so that specific claim stays interpretive.*

**How Beck changed it.** Most testing tools of the time were for checking finished work. Beck turned the runner into a step-by-step signal for *what to do next*. Red means write just enough code. Green means you may now clean up.

### 3. Extreme Programming and the C2 wiki community: test-first becomes a rule

**What it is in its own right.** XP took shape on Chrysler's C3 payroll project (from 1996) and was published in *Extreme Programming Explained* (1999). It was argued out in public on Ward Cunningham's wiki. Fowler's 2023 bliki says TDD "was developed by Kent Beck in the late 1990's as part of Extreme Programming."

The wiki pages show the rule's earlier wording and the arguments for it:
- The CodeUnitTestFirst page opens: "Rule: NeverWriteaLineOfCodeWithoutaFailingTest. The interface & output of an object are more important than its implementation, so write the former first."
- On the same page Ron Jeffries gives the loop as pick the smallest increment, test, fix the immediate problem, retest, and adds: "don't try to implement two things at a time… Just do one."
- Michael Feathers gives the emotional argument: traditionally "the test tells you that you failed. No one wants to hear that. Let's turn it around. Write the test first… Soon, the test will tell you that you've succeeded!"
- The TestDrivenDevelopment page frames the loop as "add a test, get it to fail, and write code to pass the test" alternating with "remove duplication". It calls this the "inner loop [that] pumps the outer loops of ExtremeProgramming" (continuous integration, frequent releases).
- Beck's phrase "Clean code that works", the stated goal on the book's first page, is credited there to Ron Jeffries.

**What it explains.** By the time of the 2002 book, the test-first rule wasn't Beck's private habit. It was a team rule that a community had argued over. The extreme wording ("only if an automated test has failed", "never a line") comes from XP's style of turning a good practice all the way up. "Just do one" became Canon TDD's "turn exactly one item into a test". *Documented that XP is the setting (Fowler; Wikipedia). The exact wiki wording is documented as community practice. It is **interpretive** that these particular wiki authors shaped Beck's book wording rather than the other way round, since they were his collaborators and the influence probably ran both ways.*

**How Beck changed it.** The wiki versions are mostly two-step loops: test, then code. Beck's book adds an explicit third step and makes "eliminate duplication" one of only *two* rules. That move comes from the next source.

### 4. Refactoring, from the Smalltalk community: the third step

**What it is in its own right.** Fowler says "the foundations of what we refer to these days as refactoring comes from the Smalltalk communities". Bill Opdyke, working with Ralph Johnson at Illinois, wrote the first thesis on it (1992). John Brant and Don Roberts built the first automated refactoring tool for Smalltalk. Fowler's "Etymology of Refactoring" (2003) names Cunningham and Beck as part of that circle. In XP the practice became "RefactorMercilessly": when two methods look the same, combine them, and do it continuously instead of doing big design up front, which is safe "since we have extensive UnitTests".

**What it explains.**
- *Why there is a refactor step at all.* Beck's loop contains no up-front design phase. Design happens in small refactorings after each green. Canon TDD splits it this way: writing a test involves design decisions that are "primarily interface decisions", and refactoring is when you make "implementation design decisions".
- *Why green and refactor are kept apart.* The green step allows you to commit "whatever sins necessary". That is only safe because a refactor step is coming and the tests protect it. The two steps depend on each other: tests make refactoring safe, and refactoring makes sinful green code acceptable. Canon TDD lists "mixing refactoring into making the test pass" as a mistake and sums it up as "Make it run, *then* make it right." The C2 wiki says "make it work, make it right, make it fast" is attributed to Beck in this form but is older Unix wisdom, citing a 1983 Johnson and Kernighan Byte article. *That earlier origin is a wiki claim; I did not check the Byte article.*
- *Why the rule is "eliminate duplication".* In the book, duplication is what drives generalization. Its Triangulation pattern says to "abstract only when you have two or more examples" (p. 153). Fake It returns a constant and gradually replaces constants with variables (p. 13). The duplication between the test's expected value and the faked code is the signal to generalize.

*Evidence: refactoring's Smalltalk and XP roots and Beck's membership of that circle are documented (Fowler). The step-by-step link from "RefactorMercilessly" to the third step of TDDbE is **interpretive**: it is the obvious reading and Beck's Canon TDD confirms the interface/implementation split, but I found no statement from Beck saying "I added refactoring to test-first because of X."*

**How Beck changed it.** Refactoring had been something you did now and then, when needed. Beck schedules it into *every* cycle, keeps it small ("refactoring further than necessary for this session" and "abstracting too soon" are both Canon TDD mistakes), and only lets it happen on green.

### 5. Fear: the loop as a way to manage anxiety

**What it is in its own right.** This input is personal rather than technical. Orosz reports (secondary) that when Beck followed the tape pattern, "he found his anxiety about programming vanished". In the book Beck writes: "Tests are the Programmer's Stone, transmuting fear into boredom" and "With automated tests you have a chance to choose your level of fear" (p. 124, via reader notes).

**What it explains.** It explains the *step size*. The book doesn't fix how big a step is. Its gears go from Fake It (tiny steps) through Triangulate to Obvious Implementation ("Type in the real implementation", p. 13). You change gear according to how confident you feel. When you're unsure, you take smaller steps. Feathers's line about tests telling you "you've succeeded!" fits the same reading. *Documented in Beck's book text (via notes). The link from his anxiety to the step-size gears is **interpretive**: my reading, which the "choose your level of fear" line supports.*

---

## What this shows about Beck's TDD

- **The test-first rule is a specification rule in testing clothes.** It comes from "type the expected output tape first" and keeps that meaning. Beck himself says it "is not testing". The test is written first because it states what you want, and the code is written to match it.
- **The loop's three steps come from three different places.** Red comes from the expected-answer idea, made cheap by xUnit. Green comes from XP's "simplest thing" style, with sins allowed. Refactor comes from the Smalltalk refactoring tradition, made safe by the tests. Beck's main contribution is fitting them together so each step makes the next one safe, and stating the result as just two rules.
- **What looks like Beck's own work** (as far as these sources show): (a) reducing the whole thing to two rules where duplication-removal has equal standing with test-first; (b) a named set of gears (Fake It / Triangulate / Obvious Implementation) for controlling step size; (c) in 2023, bringing the test list back to the front (people "missed this step in the book") and assigning interface decisions to test-writing and implementation decisions to refactoring. Neither the tape book, the Mercury account, nor the wiki pages contain this combination. That is an absence in the sources I read, not a proof of originality.
- **What is not original, by Beck's own account:** writing the expected result before the program. He calls himself its rediscoverer.

## What you can now recognize

- When someone says "TDD is about testing", you can see why Beck disagrees. The test was the expected output tape, an analysis and design document that happens to run. Coverage is a side effect. Canon TDD even calls tests written "without assertions just to get code coverage" a mistake.
- When you're tempted to write all the tests up front, you can recognize that as the *original* tape-era form that Beck deliberately broke into one-at-a-time steps.
- When you're tempted to tidy code while still red, or while getting to green, you can see that you're merging two steps that come from different traditions. Green is "make it run" and refactor is "make it right". Keeping them apart is what makes the "sins" safe.
- "Red" and "green" refer to an actual coloured bar in xUnit tools, and they mean *what you're allowed to do next*, not just pass or fail.
- When the book says "fake it" and returns a hard-coded constant, you can see why that isn't cheating. The duplication between the expected value and the constant is what the refactor step (rule two) is designed to remove.
- How small your steps should be is a choice Beck ties explicitly to how afraid or confident you are, not a fixed rule.

---

## Supporting detail

**Inspected directly (web pages read, 2026-11-03)**
- Kent Beck, "Canon TDD", 2023-12-11, https://newsletter.kentbeck.com/p/canon-tdd (full post).
- Wikipedia, "Test-driven development", https://en.wikipedia.org/wiki/Test-driven_development (History section, including Beck's 2012 Quora quote; Quora itself returned 403).
- stanislaw.github.io, "Notes on Test-Driven Development by Example", https://stanislaw.github.io/2016-01-25-notes-on-test-driven-development-by-example-by-kent-beck.html (page-numbered quotations from the book).
- Martin Fowler, "Test Driven Development" (bliki, 2005, updated 2023-12-11), https://martinfowler.com/bliki/TestDrivenDevelopment.html
- Martin Fowler, "Xunit" (bliki), https://martinfowler.com/bliki/Xunit.html
- Martin Fowler, "Etymology Of Refactoring", 2003-09-10, https://martinfowler.com/bliki/EtymologyOfRefactoring.html
- Beck and Gamma, "Test Infected: Programmers Love Writing Tests" (Java Report, July 1998), copy in the JUnit docs: https://android.googlesource.com/toolchain/jack/+/ub-jack-douarn-b4/junit4/doc/testinfected/testing.htm (the copy has no author line; authorship is from bibliographic records).
- C2 wiki pages CodeUnitTestFirst, TestDrivenDevelopment, RefactorMercilessly, MakeItWorkMakeItRightMakeItFast (via https://c2.com/wiki/remodel/pages/<Name> JSON).
- Larman and Basili, "Iterative and Incremental Development: A Brief History", IEEE Computer 36(6), June 2003, PDF mirror https://mosaicprojects.com.au/PDF-Gen/History_of_Iterative_and_Incremental_Development.pdf (text extracted and read around the Mercury passage).
- Gergely Orosz, "How Kent Beck shapes the software…", https://newsletter.pragmaticengineer.com/p/how-kent-beck-shapes-the-software (narration only; no first-person Beck quote on TDD's origin).

**Not inspected**
- The book *Test-Driven Development: By Example* itself (O'Reilly pages returned 403). Its wording is taken from the page-numbered notes above. One secondary source reports a 2002 draft phrasing, "Don't write a line of new code unless you first have a failing automated test", which I did not verify.
- *Extreme Programming Explained* (1999). I couldn't verify any quote from it.
- Full text of "Simple Smalltalk Testing: With Patterns" (1994). Only bibliographic records and search-result abstracts (Cambridge: https://www.cambridge.org/core/product/identifier/CBO9780511574979A037/type/BOOK_PART). I could not check whether it says anything about writing tests first.
- Full text of "Aim, Fire" (IEEE Software 18(5):87–89, 2001, DOI 10.1109/52.951502). Abstract only, via https://manuscript.isc.ac/Inventory/49/1533551.htm
- The "ancient book" itself (it is not named anywhere I found).

**Connections**

| Shaping source | Trait carried forward, and how it changed | Evidence | Source |
| --- | --- | --- | --- |
| Tape-to-tape "ancient book" | Write the expected output before the program, program until it matches. Changed from one whole-program tape to one small assertion at a time | documented | Beck, Quora 2012-05-11, quoted on Wikipedia TDD page |
| Same, as Beck reframed it | Test-first "is not testing"; it is analysis and design; "nearly as old as programming" | documented (abstract only) | "Aim, Fire", IEEE Software 2001 |
| SUnit (1994) | In-language, cheap, automated tests. The framework came *before* Beck tried test-first, and made tiny steps affordable | documented (order); composition (tool) | Beck Quora quote; Cambridge record; Fowler "Xunit" |
| JUnit / xUnit runner bar | Red bar / green bar becomes the loop's vocabulary and the signal for the next step | composition; the claim that the mantra was *named* after the bar is interpretive | "Test Infected" 1998; C2 TestDrivenDevelopment; book reviews (Mark Needham, https://www.markhneedham.com/blog/2008/10/07/test-driven-development-by-example-book-review/) |
| XP / C3 / C2 wiki | "Never write a line of code without a failing test"; "just do one"; TDD as XP's inner loop. Beck adds duplication-removal as an equal rule | documented (XP setting); interpretive (direction of influence on wording) | Fowler TDD bliki; C2 CodeUnitTestFirst, TestDrivenDevelopment |
| Ron Jeffries | "Clean code that works" as the stated goal | documented | TDDbE p. ix via stanislaw.github.io notes |
| Michael Feathers (C2) | Turning the test from a report of failure into a report of success | interpretive (a parallel statement in Beck's circle; no source says it shaped Beck) | C2 CodeUnitTestFirst |
| Smalltalk refactoring tradition (Opdyke/Johnson, Brant/Roberts; XP "RefactorMercilessly") | Continuous small design improvement protected by tests, becoming a mandatory, small third step on green only | documented (Beck in that circle; refactoring as an XP pillar); interpretive (specific lineage of the third step) | Fowler "Etymology of Refactoring"; C2 RefactorMercilessly; Canon TDD |
| "Make it work, make it right, make it fast" | Green and refactor kept apart ("Make it run, *then* make it right") | documented (Beck uses the phrase); unknown (older Unix origin is only a C2 claim) | Canon TDD; C2 MakeItWorkMakeItRightMakeItFast |
| Beck's programming anxiety | Tests "transmuting fear into boredom"; "choose your level of fear"; step-size gears | documented (book text via notes; Orosz secondary); interpretive (link to the gears) | stanislaw.github.io notes p. 124; Orosz |
| Project Mercury / IBM 1957 (Weinberg) | Test-first before each micro-increment, decades earlier | unknown as an influence on Beck (independent precedent only) | Larman and Basili 2003 |

**Unresolved branches**
- Which "ancient book" Beck read. Unnamed in his own words; Orosz's "father's book" is unattributed narration.
- Whether the 1994 SUnit column already recommends writing tests first, or whether test-first only appears from about 1996 onward in C3/XP. Beck's Quora answer only says he tried test-first *after* writing the framework.
- Whether the red/green/refactor wording first appeared in the 2002 book or earlier on the wiki or in XP talks.
- Whether the duplication-as-design-driver idea has its own source (for example Beck's Smalltalk patterns work, or the "Once and Only Once" rule on the wiki). Not researched.
- The pre-Beck history of "make it work, make it right, make it fast" (Byte 1983, Lampson 1983) was not checked first-hand.

**What changes for the current task**
- There is no coding task here. If you practise TDD from now on, the clearest guidance is Canon TDD, which is Beck's own most recent wording. Start with a behavioural test list. Turn one item at a time into a test. Allow ugly code to get to green. Refactor only on green and only as far as this session needs.
