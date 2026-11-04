# Test-driven development as Kent Beck formulated it, and what shaped it

## 1. The thing itself, in Beck's own words

Beck has written the practice down twice, twenty years apart, and the two versions bracket what "TDD as Kent Beck formulated it" means.

**2002, *Test-Driven Development: By Example*.** The preface opens with Ron Jeffries' phrase: "Clean code that works" is the goal. It then gives two rules and a three-colour mantra. The rules: write new code only if an automated test has failed; eliminate duplication. The mantra: *Red* (write a little test that doesn't work, perhaps doesn't even compile), *Green* (make it work quickly, committing whatever sins are necessary), *Refactor* (eliminate all the duplication created in merely getting the test to work). The preface also says the thing most people forget: "Test-driven development is a way of managing fear during programming."

**December 2023, "Canon TDD"** (Beck's newsletter). Written, he says, to "forestall strawmen" so critics argue with "the actual thing." Five steps:

1. **Test list** - write down the scenarios you want to cover (behaviour, not implementation).
2. **Write a test** - turn exactly one item into a real, runnable test with setup, invocation, assertions.
3. **Make it pass** - change the system so this test and all previous ones pass; add new list items as you discover them.
4. **Optionally refactor** - now make the implementation-design decisions.
5. **Repeat** until the list is empty.

Notice what Canon TDD does *not* do: it never says "red-green-refactor" as a slogan (the colours appear once, in passing), and refactoring is *optional*. It also adds step 1, the list, to answer the complaint that TDD "just launches into coding." Beck's own gloss: "I'm not telling you how to program. I'm not charging for gold stars," and "Keep testing & coding until your fear for the behavior of the code has been transmuted into boredom."

So the stable core across both formulations is: **a test that fails before the code exists, the smallest change that makes it pass, then a separate design pass with the tests as a safety net, in very short cycles.** Everything below is the story of where each of those pieces came from.

## 2. The write-the-test-first rule: a "rediscovery," not an invention

Beck is explicit that he did not invent test-first. His account (a Quora answer, quoted on Wikipedia's TDD page and retold in a 2025 Pragmatic Engineer profile):

> "The original description of TDD was in an ancient book about programming. It said you take the input tape, manually type in the output tape you expect, then program until the actual output tape matches the expected output. After I'd written the first xUnit framework in Smalltalk I remembered reading this and tried it out. That was the origin of TDD for me."

He adds that older programmers react to TDD with "Of course. How else could you program?", which is why he calls his role "rediscovering" TDD. He read the book as a child and did not understand it; the idea came back to him only when a testing framework made it cheap to try.

He said the same thing in 2001 in his IEEE Software column "Aim, Fire": test-first coding "is nearly as old as programming," it "is not really testing," and it is better understood as an analysis technique (you decide what you are building and what answers to expect) and a design technique. And in a January 2000 interview with Nicolai Josuttis: "All the techniques in XP are nearly as old as programming itself," and "I'm lucky if an original thought happens through my head once in five years."

The historical record backs the modesty. Larman and Basili's 2003 survey "Iterative and Incremental Development: A Brief History" (IEEE Computer) reports that NASA's early-1960s Project Mercury software "ran with very short (half-day) iterations that were time boxed. The development team conducted a technical review of all changes, and, interestingly, applied the Extreme Programming practice of test-first development, planning and writing tests before each micro-increment. They also practiced top-down development with stubs." Gerald Weinberg, who was there, told them the team was doing incremental development "as early as 1957, in Los Angeles," using a technique "indistinguishable from XP," and that "all of us ... thought waterfalling of a huge project was rather stupid." A contributor to the C2 wiki's CodeUnitTestFirst page adds the folk explanation: in the PL/I and COBOL era "all developers were required to write a unit test plan before they went near any code," because the compile-and-debug cycle was so slow that you had to think the expected answer through before you spent a run on it. The practice was forgotten when desktop computing made trial-and-error cheap.

That is the first shaping force: **test-first is what batch-era programmers did when a run was expensive.** Beck's contribution was to notice that it still pays when runs are nearly free, if you make the cycle very short.

Beck also credits a more personal source. Asked in 2000 where XP's ideas came from, he named his father, Doug Beck, who "wrote process control software in assembly language for 8 bit micro processors," for the idea of making changes in small steps while keeping the system running; Christopher Glaser, "a compiler writer I worked with at MasPar," for the testing strategy; Ward Cunningham for "the evolutionary design philosophy"; Jon Hopkins for the iteration schedule; and Christopher Alexander for separating business from technical decisions.

## 3. Smalltalk at Tektronix: where the short cycle comes from

The second shaping force is the environment Beck learned to program in. In the mid-1980s Beck and Ward Cunningham were at Tektronix's Tek Labs, "some of the very earliest Smalltalk application programmers" (Rebecca Wirfs-Brock, who was a principal engineer there, writing in 2023). Her description of that environment reads like a description of TDD's inner loop:

- "The Smalltalk image was always running" and "contained the entire development environment."
- A workspace was a scratchpad where you could evaluate snippets against the live system without committing them.
- The tools made it easy to "back up a step or two, revise your code, and try again."
- "I tested my code into existence, but didn't leave around any tests."

Her reading of what Beck did with this: "Many ideas about Extreme Programming (and TDD) and object-design can be traced back to these programming experiences," and she suspects Beck "tried to boil down his Smalltalk experiences into practices that would be more 'failsafe'" for programmers who did not have such a forgiving environment. The C2 wiki's XpRoots page says the same from the inside: XP's "core origins lie in the Smalltalk tradition and date back to the mid-80s when Kent and Ward worked at Tektronix"; XP is "the Smalltalk way generalized to other environments," and "has not been designed by a master plan."

Beck's own 2000 remark about Java confirms the direction of the debt: "when I program in Java I think I'm back 15 years ago... Imagine, not having decent support for refactoring! How annoying." The fast, reversible, minute-by-minute rhythm of TDD is the Smalltalk workspace rhythm, made portable by replacing "evaluate a snippet and look" with "run an automated test and look."

Two more Tektronix-era habits fed in. Beck and Cunningham started applying Christopher Alexander's *patterns* to software in 1987 (an OOPSLA '87 paper) and wrote the 1989 CRC-card paper "A Laboratory for Teaching Object-Oriented Thinking." Patterns are why Beck's testing framework paper is titled "...With Patterns" and why the TDD book is organised as a catalogue of small named moves (Fake It, Triangulate, Obvious Implementation). Cunningham's "Episodes" pattern language (1996) captured the pair's working practices from 1986 onward; XpRoots lists it as one of the four ingredients of XP.

## 4. SUnit, 1994: the tool that made the old idea cheap enough to try

The third force is a specific piece of code. In the October 1994 *Smalltalk Report*, Beck published "Simple Smalltalk Testing: With Patterns," opening with "Smalltalk has suffered because it lacked a testing culture." The article presents a philosophy, a literate-program framework (TestCase, fixtures, TestSuite, a `should:` check), a worked example on Set, and a cookbook for writing your own tests, and closes with "Get to work writing tests." That framework is SUnit, the ancestor of every xUnit. (Wikipedia dates SUnit to 1989; Cambridge University Press, which reprinted the column as chapter 30 of *Kent Beck's Guide to Better Smalltalk*, dates the original to October 1994, and Beck himself says in Canon TDD that he "wrote SUnit in 1994 for a consulting client." Take 1994.)

Beck's rediscovery story hinges on sequence: framework first, then the memory of the tape book, then the experiment of writing the test before the code. In the Pragmatic Engineer profile he says the idea seemed absurd at first, but his anxiety about programming dropped, and he was converted. In Canon TDD he dates the next milestone precisely: he "demoed TDD for Ward Cunningham at the Austin OOPSLA conference in October 1995," and says he cannot find published references to TDD from that era.

Note what the 1994 article does *not* say: it never tells you to write the test first. It is about making tests cheap and about culture. Test-first was the thing Beck did *with* the tool once it existed. This is why Beck keeps insisting TDD is not a testing technique: the framework is the testing part; the rule about ordering is a design and anxiety-management part that the framework merely enabled.

## 5. C3 and Extreme Programming, 1996-1999: test-first becomes a rule

In 1996 Beck led the restart of Chrysler's C3 payroll project (Fowler: "It was this rebooted project that first pulled together all the practices that became known as Extreme Programming"; Beck "had used similar approaches on earlier projects"). The Pragmatic Engineer profile has Beck throwing away a codebase that didn't work, restarting with a new methodology, pairing, and "using his testing ideas." The system went live in 1997, paid about ten thousand people, was stopped in 1999, and was eventually reverted to COBOL, which Fowler notes "proves that XP is no guarantee of success."

*Extreme Programming Explained* (1999) made the ordering explicit for the first time in print. Its testing chapter opens: "we will write tests before we code, minute by minute." Even so, one close reader of the first edition (Noel Rappin) observes that Beck "doesn't quite stress testing first yet": most of the chapter is spent defending the idea that programmers should write tests *at all*, isolated and automatic, and Red/Green/Refactor is not in the book. In the 2000 Josuttis interview Beck gives the reason test-first matters inside XP: "Writing the test case forces them to make design decisions about the interface for the new logic," pairs "run the test case, just in case it already runs," and "test-first-coding is also the practice that is most useful whatever project culture you have."

The C2 wiki, Cunningham's "fabulous Wiki-Wiki Web server" on which the XP crowd worked, turned this into the aphorism form that spread: CodeUnitTestFirst, with its rule NeverWriteALineOfCodeWithoutAFailingTest and Ron Jeffries' four-step loop (decide what needs doing; write a unit test for the smallest next capability; run it, and if it passes go back to the start; otherwise fix the immediate problem and run again). Jeffries: "Guaranteed flow. And you go so fast!" Michael Feathers, on the same page: "Write the test first; run it. Of course it fails." Bob Martin reports learning the fine-grained version by pairing with Beck in 1999, where Beck "wrote one line of a failing test and then the matching line of production code"; Martin's later "Three Laws of TDD" are, by his own description, a codification of that nano-cycle, not a Beck formulation.

## 6. Refactoring: where the third step comes from

The "refactor" step has its own lineage, and it is why TDD is a three-beat loop rather than the two-beat "test, then code" of the 1950s. XpRoots traces it: Bill Opdyke's 1989-92 dissertation "Refactoring Object-Oriented Frameworks," Beck's *Smalltalk Best Practice Patterns* (1996), then Martin Fowler's *Refactoring* (1999, with Beck as a contributor), which Rappin notes "assumes test-first development." XpRoots puts it bluntly: "Test-driven development is also derived from the refactoring practices." The dependency runs both ways: refactoring needs a fast regression suite to be safe, and test-first produces one as a by-product; test-first needs a licence to write sinful code to get to green, and refactoring is that licence. Beck's slogan for the split is the one Canon TDD repeats: "Make it run, *then* make it right." Bob Martin traces Red-Green-Refactor to Beck's injunction "Make it work. Make it right. Make it fast."

In the 2002 book the refactor step has one specific job, "eliminate duplication," which is Beck's second rule and his design theory in miniature: let duplication between test and code, or within the code, tell you where the abstraction should be, rather than designing it up front. Canon TDD softens this: "Duplication is a hint, not a command."

## 7. Red and green: where the colours come from

The colours are the youngest part, and they come from a user interface. In 1997-98 Beck and Erich Gamma ported SUnit to Java as JUnit (the story goes that it was written on a flight to OOPSLA). Their July 1998 *Java Report* article "Test Infected: Programmers Love Writing Tests" opens with "Every programmer knows they should write tests for their code. Few do," diagnoses a vicious cycle of pressure and skipped tests, and offers the framework as the outside influence that breaks it. It describes the rhythm as "code a little, test a little, code a little, test a little," and then states the preference that became the rule: "The style here is to write a few lines of code, then a test that should run, or even better, to write a test that won't run, then write the code that will make it run." It also describes the GUI runner: "The bar is initially green but turns into red as soon as there is an unsuccessful test," and the walkthrough narrates the state of the code by the colour of the bar. Being "test infected" means "you can't go home if your tests aren't 100%."

Once thousands of Java programmers were watching that bar, the colours became the vocabulary. Bill Wake's "The Test-First Stoplight" (January 2001) spells out a green-yellow-red-green cycle keyed to JUnit's bar ("Extreme Programmers write tests before they write the corresponding production code"; a compile failure "will be a yellow light"), and advises that a cycle of 10-15 minutes "is too long, and you need to move to smaller tests." Beck's 2002 book then fixed the three-word form, Red/Green/Refactor, that everyone now uses. So the mantra is: 1950s test-first (red before green), plus Opdyke/Fowler refactoring (the third beat), named after the 1997 JUnit progress bar.

## 8. Fear, and why Beck frames it as psychology rather than quality

The force that explains the *shape* of the loop (why so small, why run the test before the code exists, why green before clean) is the one Beck puts first in the preface: fear. His own conversion story is that his anxiety dropped. The 2002 preface: TDD "is a way of managing fear during programming," meaning the legitimate fear of a hard problem, which makes you tentative, stop communicating, and avoid feedback. Each piece of the loop is an anti-fear device: the test list turns a vague worry into a concrete next step; the failing test proves the test can fail (several C2 contributors give this as the main reason for the ordering); the tiny green step gives a guaranteed success every few minutes; the refactor step is done only with a green bar under you. Canon TDD's closing advice is the same idea twenty years on: keep going until fear "has been transmuted into boredom."

This is also why Beck resists the "testing" label. Test-first, for him, is an analysis technique (decide what the right answer is before you can be seduced by the answer you got), a design technique (the test is the first client of the interface, and it forces interface decisions before implementation ones; Canon TDD makes this explicit by splitting "how behaviour is invoked" from "how the system implements it"), and a way of working with the grain of a nervous human mind ("People are lousy computers").

## 9. What Beck has changed, and what he hasn't

Beck has kept loosening the rule while defending the core. In 2014 he called TDD "a little box in this big space of feedback" and a "sweet spot," and said he would "rather focus on teaching them the principles than policing whether they are aping one particular ritual." In 2019 (quoted by Wirfs-Brock) he introduced "the half-life of the line of code" as a factor in deciding when a test is worth writing. In 2018, at a code camp in Oslo, he and Oddmund Strømme, Lars Barlindhaug and Ole Johannessen stumbled on "test && commit || revert" (TCR): if the tests fail, the code is reverted to the last green state. It came from Strømme's observation that ordinary TDD is asymmetric (a green run commits you to nothing, a red run lets you keep hacking), and Beck initially thought it couldn't work. Canon TDD (2023) is the latest move: refactoring made optional, a test list added at the front, and the whole thing described as a programming workflow rather than a law.

What has not moved since 1995: one failing test before the code exists; the smallest change that makes it pass; all previous tests still passing; a separate moment for design; and cycles short enough that a mistake is cheap to undo. That is the Smalltalk workspace rhythm, under the discipline of a batch-era programmer who had to write the output tape before the run, with a JUnit progress bar for a metronome.

## Caveats on the record

- Beck has never named the "ancient book." It is sometimes guessed to be D. D. McCracken's *Digital Computer Programming* (1957); I could not verify any passage, so treat that as unconfirmed.
- Beck's Quora answer itself is behind a login; the quotation above is as reproduced on Wikipedia's TDD page and consistent with the Pragmatic Engineer retelling.
- The Project Mercury "test-first" claim rests on Larman and Basili's reading of the records plus Weinberg's recollection, not on a surviving Mercury document.
- SUnit: 1989 (Wikipedia) vs 1994 (Cambridge UP and Beck himself); 1994 is better supported.

## Sources read

- Kent Beck, "Canon TDD," newsletter.kentbeck.com/p/canon-tdd (Dec 2023)
- Wikipedia, "Test-driven development" (history section, quoting Beck's Quora answer)
- Wikipedia, "Kent Beck"
- Craig Larman and Victor Basili, "Iterative and Incremental Development: A Brief History," IEEE Computer, June 2003 (PDF via mosaicprojects.com.au)
- Kent Beck and Erich Gamma, "Test Infected: Programmers Love Writing Tests," Java Report, July 1998 (text in the junit4 repository)
- Erich Gamma and Kent Beck, "JUnit: A Cook's Tour" (junit4 doc)
- Cambridge University Press abstract for "Simple Smalltalk Testing" (ch. 30, *Kent Beck's Guide to Better Smalltalk*; orig. Smalltalk Report, Oct 1994)
- Nicolai Josuttis, "eXtreme Programming: An interview with Kent Beck," Overload 35, Jan 2000 (accu.org)
- C2 wiki, "XpRoots" and "CodeUnitTestFirst" (mirrors)
- Martin Fowler, "C3," martinfowler.com/bliki/C3.html
- Rebecca Wirfs-Brock, "Our Heuristics are Shaped Through Experience," 2 May 2023
- Bill Wake, "The Test-First Stoplight," xp123.com, 2 Jan 2001
- Robert C. Martin, "The Cycles of TDD," cleancoder.com, 17 Dec 2014
- Noel Rappin, "When books could code: Extreme Programming," buttondown.com/noelrap
- Pragmatic Engineer, "How Kent Beck shapes the software industry" (profile)
- Chris Oldwood, "Afterwood," Overload 183, Oct 2024 (fetched; not used)

Search-only (snippets, not fetched): Beck's IEEE Software column "Aim, Fire" (2001, abstract only); *Test-Driven Development: By Example* preface (publisher and reviewer excerpts); Beck's 2014 PragPub interview; Beck's 2018 TCR post (summaries).
