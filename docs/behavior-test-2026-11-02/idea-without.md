# Test-driven development as Kent Beck formulated it, and what shaped it

## 1. What Beck actually formulated

Beck's canonical statement is the preface of *Test-Driven Development: By Example* (Addison-Wesley, 2002; many catalogues say 2003). It sets a goal, two rules, and a three-beat loop:

- **Goal.** "Clean code that works," which Beck credits as "Ron Jeffries' pithy phrase." That phrase came out of the Chrysler C3 team (see section 2.4).
- **Two rules.** "Write new code only if you first have a failing automated test. Eliminate duplication." Beck adds: "Two simple rules, but they generate complex individual and group behavior."
- **The loop.** "Red: write a little test that doesn't work, perhaps doesn't even compile at first. Green: make the test work quickly, committing whatever sins necessary in the process. Refactor: eliminate all the duplication created in just getting the test to work." Then: "Red/green/refactor. The TDD's mantra."
- **Why.** "Test-driven development is a way of managing fear during programming," with fear meant "in the legitimate, this-is-a-hard-problem-and-I-can't-see-the-end-from-the-beginning sense." The preface has a section titled "Courage" that asks: "Why would a programmer work in tiny little steps when their mind is capable of great soaring swoops of design?"

Twenty-one years later, in "Canon TDD" (Beck's newsletter, 11 Dec 2023), he restated it as a procedure, partly to correct what he sees as common misreadings:

1. Write a **test list** of expected behavioral cases for the change (including existing behavior that must keep working). Keep implementation design out of the list.
2. Write **one** real automated test (setup, invocation, assertions). The design decisions here are about the interface.
3. **Make it pass** by changing the system; add any newly discovered cases to the list.
4. **Optionally refactor** now that the test passes (implementation design decisions happen here).
5. Repeat from step 2 until the list is empty.

The stated purposes: existing behavior keeps working, new behavior works as expected, the system is ready for the next change, and the programmer and colleagues feel confident of all three. The mistakes he calls out are instructive about what the loop is *for*: deleting assertions to fake a pass; pasting computed output into the expected value; mixing refactoring into getting the test to pass ("wearing two hats"; "make it run, then make it right"); tests with no assertions written only for coverage; turning the whole list into tests up front; and over-abstracting ("Duplication is a hint, not a command").

So the thing to explain is not "testing" but a specific discipline: *decide the expected answer first, in executable form; make it true by the smallest means; then clean up.* Each of those three moves has a traceable origin.

## 2. What shaped it, in roughly the order it happened

### 2.1 A book on his father's shelf: the "tape-to-tape" rule (the write-the-test-first idea)

Beck has said for years that he did not invent TDD, he *rediscovered* it. His 2012 Quora answer (the original is behind a login; it is quoted in Wikipedia's TDD article and in many secondary pages) says, in substance: the original description of TDD was in an ancient programming book, which said you take the input tape, manually type in the output tape you expect, then program until the actual output tape matches the expected output. "After I'd written the first xUnit framework in Smalltalk I remembered reading this and tried it out. That was the origin of TDD for me." Hence: "Therefore I refer to my role as 'rediscovering' TDD."

The Pragmatic Engineer's 2025 profile adds the personal frame: Beck's father was an electrical engineer; as a kid Beck read one of his father's programming books from the tape-to-tape era, did not understand the advice, and forgot it. Years later, while building SUnit, the memory came back and he mapped it onto the framework. Following the old rule meant writing the test before the code. He says he laughed at it first ("such a stupid idea": why write a test guaranteed to fail?), tried it anyway, and found that his anxiety about programming disappeared. That reaction (the anxiety going away) is the emotional seed of the "managing fear" framing in the 2002 preface.

Which book? Beck has never named it. Tester Michael Bolton proposed Daniel D. McCracken's *Digital Computer Programming* (Wiley, 1957), the first general "driver's manual" for programming, which uses an imaginary machine (TYDAC) and devotes a section to "checkout." The passage Bolton quotes: "The first attack on the checkout problem may be made *before coding* is begun... In order to fully ascertain the accuracy of the answers, it is necessary to have a hand-calculated check case..." That is the same idea in 1957 vocabulary: produce the expected answer by hand before the machine runs. Treat the identification as plausible, not confirmed by Beck.

Why did 1950s programmers work that way? Because machine time was scarce and batch runs were slow, so you had to know what the right answer looked like before you submitted the job. The practice did not survive the arrival of interactive, cheap computing, which is why it read as novel when Beck brought it back in a world of instant compile-run.

### 2.2 Smalltalk, Tektronix and Ward Cunningham: the habits underneath the loop

Beck studied at the University of Oregon (1979-87), where he first met Christopher Alexander's pattern work. At Tektronix in the late 1980s he and Ward Cunningham were among the earliest Smalltalk application programmers; they built HotDraw together, kept a well-thumbed thesaurus because Cunningham cared intensely about naming, presented pattern languages for object-oriented programs at OOPSLA '87, and introduced CRC cards at OOPSLA '89. Beck moved to Apple in 1987 for Smalltalk (the project "went nowhere" because Apple's customers wanted C and Pascal), then ran his own consultancy, First Class Software, consulting to Smalltalk shops and writing *Smalltalk Best Practice Patterns* (1997).

Three things from this period are load-bearing in TDD:

- **Tiny steps with instant feedback.** Smalltalk's image-based environment lets you change a method and run it immediately. The red/green cadence of seconds-to-minutes assumes that kind of environment; it is a Smalltalk working style transplanted into Java and later everything else.
- **Naming, patterns and "refactor" as a first-class move.** *Smalltalk Best Practice Patterns* is a catalogue of small, named code-level decisions (how to name, how to split methods, how to remove duplication). Beck was also a contributor to Fowler's *Refactoring* (1999, with Brant, Opdyke and Roberts; Opdyke's 1992 thesis coined the term). The third beat of the loop, "eliminate duplication," is this body of practice given a fixed slot in the cycle.
- **Pairing and a shared vocabulary.** The pairing style Beck and Cunningham developed became an XP practice, and Cunningham's WikiWikiWeb (1995) became the place where test-first was argued out in public (section 2.4).

### 2.3 SUnit, 1994: the mechanism

The usual claim that SUnit dates from 1989 (Wikipedia) is wrong. Beck's own account in "Canon TDD" is that he wrote SUnit in 1994 for a consulting client, "so my TDD work cannot predate that," and that he demoed TDD for Ward Cunningham at OOPSLA in Austin in October 1995, which bounds the rediscovery to 1994-95. The article describing it, "Simple Smalltalk Testing: With Patterns," appeared in *The Smalltalk Report* in October 1994 and was reprinted as chapter 30 of *Kent Beck's Guide to Better Smalltalk* (1997).

What the article argues matters as much as the code:

- "Smalltalk has suffered because it lacked a testing culture." The column offers "a simple testing strategy and a framework to support it," explicitly "not intended to be complete solutions."
- UI-driving script tests are rejected as too brittle (cosmetic changes cause floods of false failures). Instead tests are written in Smalltalk by the developers themselves, one unit test class per class; he suggests developers spend 25-50% of their time on tests. An independent tester handles integration tests.
- The framework is a tiny pattern language: **Fixture** (a known starting set of objects, built fresh for every test so tests cannot interfere), **Test Case** (stimulate the fixture), **Check** (`should:` / `shouldnt:` assertions), **Test Suite**. Classes: TestCase, TestSuite, TestResult; failures are distinguished from errors; tearDown always runs.

Note what SUnit was for in 1994: making it cheap for developers to write and run their own tests. It said nothing about writing them *first*. The order rule came when Beck connected the old tape book to the new tool. That sequence (tool first, then the discipline) is why Beck is careful to say TDD "cannot predate" SUnit.

### 2.4 Chrysler C3 and XP, 1996-2000: test-first becomes a team rule

Beck was brought into Chrysler's Comprehensive Compensation (C3) payroll project (Smalltalk + GemStone) in 1996, initially for performance tuning. He became project leader in March 1996, threw out a codebase that did not work, brought in Ron Jeffries, and restarted with pairing and his testing ideas. He named the result "extreme programming" on purpose, expecting the establishment to dislike it. *Extreme Programming Explained* (October 1999) made testing one of the twelve practices: programmers continually write unit tests that must pass for development to continue, and customers write tests showing features are done. Chrysler cancelled C3 in February 2000 after the Daimler merger.

Two artefacts of this period shaped the vocabulary of the loop:

- **Red and green.** Martin Fowler records that the red/green bar in JUnit "had started as a red or green window at C3": the team coloured the whole window red once a test failed and green when all passed. The colours were a team-room signal before they were a mantra.
- **The rule, stated bluntly on Ward's wiki.** The c2 page *CodeUnitTestFirst* carries the rule "NeverWriteALineOfCodeWithoutAFailingTest," and Ron Jeffries' four-step cycle: find out what you have to do; write a UnitTest for the smallest increment of capability; run it; fix the immediate problem and rerun, one thing at a time. The discussion on that page (Beck, Michael Feathers, Pieter Nagel, others) already contains the standard justifications: a failing-first test proves the test can fail; test-first forces clean interfaces; tests written afterwards hinder refactoring. One contributor notes that the previous generation of programmers wrote unit test plans before coding, and Keith Braithwaite replies that the industry doesn't learn from its successes, which is exactly Beck's "rediscovery" point.

Also from C3: Jeffries' phrase "clean code that works," which Beck adopted as the stated goal of TDD.

### 2.5 JUnit and "Test Infected", 1997-98: the loop goes mainstream

JUnit "was born on a flight from Zurich to the 1997 OOPSLA in Atlanta," Beck pairing with Erich Gamma, built test-first (Fowler was an early alpha user). Beck and Gamma's article "Test Infected: Programmers Love Writing Tests" (*Java Report*, July 1998) carried the practice to the Java world with the arguments that still get repeated:

- The problem: "Testing is not closely integrated with development," so progress cannot be measured; the excuse is "I'm in too much of a hurry," which feeds a vicious cycle of pressure, fewer tests, less stability, more pressure.
- The rule: "When you need to add new functionality to the system, write the tests first." And the sequence in miniature: write "a test that won't run, then write the code that will make it run."
- The signal: "The bar is initially green but turns into red as soon as there is an unsuccessful test."
- The attitude: "Once you've been test infected, your attitude toward development is likely to change." Beck credits Gamma with coining "test infected."

JUnit mattered because it was the first time the SUnit shape (fixture/setUp, test methods, assertions, suite, red/green runner) was available in a language with a mass audience. Fowler calls it "the seedwork" that every xUnit port copied; Michael Feathers' CppUnit was probably the first.

### 2.6 "Aim, Fire" (2001) and the 2002 book: test-first reframed as design

By 2001 Beck was explicit that the practice was not about testing. In "Aim, Fire" (*IEEE Software* 18(5), 2001, pp. 87-89) he argues that test-first coding is not a testing technique but an analysis and design technique, that it is "nearly as old as programming," and that its core is deciding up front what we are programming, what we are not programming, and what answers we expect. That is the tape rule restated as a design discipline.

*TDD By Example* (2002) then did three things: it renamed "test-first programming" to "test-driven development" (stressing that the tests drive the design, not merely precede the code); it taught the loop through two long worked examples, a multi-currency money example Beck says he "got from Ward Cunningham years ago," and an xUnit framework built test-first in Python; and it added the fear/courage framing from his own experience of anxiety. Fowler's bliki entry (2005, revised in 2023 "inspired by Kent's canonical post") summarises the result as: write a test for the next bit of functionality; write the functional code until it passes; refactor both new and old code; preceded by making a list of test cases.

### 2.7 The deeper lineage Beck did not know about

Craig Larman and Victor Basili's "Iterative and Incremental Development: A Brief History" (*IEEE Computer*, June 2003) found that NASA's Project Mercury software, late 1950s to early 1960s, used short timeboxed iterations with a technical review of all changes, and that "the Extreme Programming practice of test-first development was applied": "tests were planned and written in advance of each micro-increment." Gerald Weinberg, who was there, says "we were doing incremental development as early as 1957" and calls a later Motorola project "indistinguishable from XP." Beck himself writes in "Canon TDD" that he could not find published references to TDD from the mid-1990s and delayed the book long enough to worry about being scooped. The Mercury evidence surfaced after the book and simply confirms his instinct that he had rediscovered a batch-era practice.

## 3. Putting the loop back together

Read each beat against its source and the shape stops looking arbitrary:

| Beat | What it says | Where it comes from |
|---|---|---|
| Test list first | enumerate expected behaviours before coding | "Aim" in "Aim, Fire"; the 1957 "attack the checkout problem before coding"; Canon TDD step 1 |
| **Red** | write one failing executable expectation | tape-to-tape rule (expected output typed before the program); SUnit/JUnit make it cheap; C3 red window; c2 "never write a line of code without a failing test" |
| **Green** | smallest change that passes, "committing whatever sins necessary" | Smalltalk's tiny-step, instant-feedback style; fear management (one small step you can see the end of); Jeffries' "fix the immediate problem, one thing at a time" |
| **Refactor** | eliminate the duplication you just created | *Smalltalk Best Practice Patterns*; Opdyke/Fowler/Beck *Refactoring*; Cunningham's naming obsession; Canon's "duplication is a hint, not a command" |
| Keep looping | until the list is empty | Jeffries' cycle on c2; XP's "tests must pass for development to continue" |

The two rules of the 2002 preface map onto the two halves of Beck's background: "write new code only if you first have a failing automated test" is the batch-era discipline carried by SUnit/JUnit; "eliminate duplication" is the Smalltalk pattern/refactoring culture. The fear framing is what Beck added himself, from the experience of trying the "stupid idea" and feeling his anxiety go away.

## 4. Things people get wrong

- **SUnit is 1994, not 1989.** Wikipedia's SUnit and Beck pages give 1989; Cambridge's listing of the original column says *Smalltalk Report*, October 1994, and Beck's 2023 account says he wrote it in 1994. The 1989 date is apparently a confusion with the Beck/Cunningham OOPSLA '89 paper.
- **"Beck invented TDD in the late 1990s as part of XP."** XP packaged and named the practice; the practice itself dates to his 1994-95 rediscovery, and the idea to the 1950s.
- **The 1957 book.** Beck's Quora answer is only available secondhand, and the McCracken identification is Bolton's inference, not Beck's statement.
- **TDD is not testing.** Beck's own line from 2001 onward is that it is analysis and design with executable expectations; the tests are a by-product.
- **Beck and TDD today.** He has moved on from evangelising it (the Pragmatic Engineer profile says he "abandoned TDD just as it peaked" in the sense of stopping promotion), took part in the 2014 "Is TDD Dead?" conversations with DHH and Fowler, but as of the 2025 interviews still works test-first, including with AI coding agents.

## Sources fetched and read

- Kent Beck, "Canon TDD," Software Design: Tidy First? newsletter, 11 Dec 2023. https://newsletter.kentbeck.com/p/canon-tdd
- Kent Beck, *Test-Driven Development: By Example*, preface (InformIT sample). https://www.informit.com/store/test-driven-development-by-example-9780321146533
- Kent Beck, "Simple Smalltalk Testing: With Patterns," *Smalltalk Report*, Oct 1994; chapter page at Cambridge University Press. https://www.cambridge.org/core/books/abs/kent-becks-guide-to-better-smalltalk/simple-smalltalk-testing/0D2F14E0129D5EF29E85D796A5B845FC
- Reproduction/summary of the same article (testingsoftware.blogspot.com, Jan 2006). https://testingsoftware.blogspot.com/2006/01
- Beck and Gamma, "Test Infected: Programmers Love Writing Tests," JUnit 3.8.2 docs copy. https://sources.debian.org/data/main/j/junit/3.8.2-9/doc/testinfected/testing.htm
- Martin Fowler, "Xunit," bliki, 17 Jan 2006. https://martinfowler.com/bliki/Xunit.html
- Martin Fowler, "TestDrivenDevelopment," bliki, 2005 rev. 2023. https://martinfowler.com/bliki/TestDrivenDevelopment.html
- Ward Cunningham's wiki, "CodeUnitTestFirst" (mirror). https://c2wiki.gothdyke.mom/CodeUnitTestFirst.html
- Ward Cunningham's wiki, "HistoryOfIterative" (excerpts of Larman and Basili 2003, mirror). https://c2wiki.gothdyke.mom/HistoryOfIterative.html
- Kent Beck author bio, Portland Pattern Repository (c2.com). https://c2.com/ppr/about/author/kent.html
- Gergely Orosz, "How Kent Beck shapes the software engineering industry," The Pragmatic Engineer, 2025 (two page views). https://newsletter.pragmaticengineer.com/p/how-kent-beck-shapes-the-software
- Markus Heilig, "Day 27 - The Origins of TDD" (Beck Quora excerpt; Bolton's McCracken 1957 quotation). https://markusheilig.netlify.app/day-27-the-origins-of-tdd
- Wikipedia, "Test-driven development." https://en.wikipedia.org/wiki/Test-driven_development
- Wikipedia, "Kent Beck." https://en.wikipedia.org/wiki/Kent_Beck
- Wikipedia, "Extreme programming." https://en.wikipedia.org/wiki/Extreme_programming

Consulted via search snippets only (not fetched in full): Beck, "Aim, Fire," *IEEE Software* 18(5), 2001 (abstract); Computer History Museum and *Physics Today* 1957 entries on McCracken's *Digital Computer Programming*; Larman and Basili 2003 PDF (downloaded but text not extractable here).
