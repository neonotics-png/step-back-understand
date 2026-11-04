# Step back: test-driven development as Kent Beck formulated it

Sources checked 2026-11-02. Everything below comes from pages fetched and read today. Where a source could not be opened, the supporting detail says so, and claims that rest only on someone else's report are marked that way.

**Subject and question.** Kent Beck's TDD, in the form his 2002 book *Test-Driven Development: By Example* gives it: two rules ("Write new code only if an automated test has failed" and "Eliminate duplication") and the mantra red/green/refactor. Red means write a little test that doesn't work. Green means make it work quickly, "committing whatever sins necessary". Refactor means remove the duplication you just created (preface, pp. ix–x, as quoted in Stanislaw Pankevich's reading notes). The question is where the loop and the write-the-test-first rule came from.

## The explanation

Read through what shaped it, TDD stops looking like one invention. It is three older things bolted together, with the order of steps settled late. The evidence for each part is uneven, so each part below says what kind of evidence it rests on.

**1. Writing the test first is the oldest part, and Beck says it isn't his.** His own account, a 2012 Quora answer quoted on Wikipedia's TDD page: "The original description of TDD was in an ancient book about programming. It said you take the input tape, manually type in the output tape you expect, then program until the actual output tape matches the expected output. After I'd written the first xUnit framework in Smalltalk I remembered reading this and tried it out." That is why he calls his role "rediscovering". In batch tape processing, the expected output is a separate physical object that you prepare before the program exists. The program is finished when its output matches. That is red before green, decades before the colours existed. A 2026 Pragmatic Engineer article by Gergely Orosz adds that the book belonged to Beck's father and that Beck read it as a child. That detail is Orosz's summary, not a Beck quote I could check. **Beck never names the book in anything I read.** Blogs often point to Daniel McCracken's *Digital Computer Programming* (1957), quoting a passage about a "hand-calculated check case" prepared "before coding is begun". I could not open the book: the Internet Archive copy is lending-restricted, and the quote reaches the web only third-hand. So McCracken shows the idea existed in 1950s textbooks. It does not show that his was Beck's book. Larman and Basili's history (IEEE Computer, 2003) says NASA's Project Mercury software team in the early 1960s "applied the Extreme Programming practice of test-first development, planning and writing tests before each micro-increment." That is a parallel, not an ancestor: nothing I read links Beck to Mercury.

**2. The test harness came from Smalltalk's way of working, and test-first came after it.** Beck's October 1994 Smalltalk Report column, "Simple Smalltalk Testing: With Patterns", is the founding text of SUnit and the whole xUnit family. I read it in full, and it is not a test-first text. It recommends that "developers write their own unit tests, one per class", spending "25-50% of their time developing tests", and it never says when to write them. What it does show is where the shape of a unit test came from. "The simplest way to represent this is interactively. You open an Inspector on your fixture and you start sending it messages." The problem is that interactive checks interfere with each other and "you can't easily communicate interactive tests to others." So Beck turned each "predictable situation" into a method, each with its own fixture. An xUnit test is a Smalltalk programmer's interactive poke at an object, made into a saved object. This matches Beck's own sequence: build the framework, then remember the tape book and try writing the test first. In his Canon TDD post (2023) he dates SUnit to 1994 and his first TDD demo for Ward Cunningham to OOPSLA, October 1995. Test-first came after the framework: the order in Beck's own account, confirmed by the 1994 column's silence on when to write tests.

**3. Red and green came from a window colour, and the name came later.** Martin Fowler, who worked on Chrysler's C3 payroll project, writes: "At C3 we would color the whole window red once a test failed and green if they all passed. JUnit introduced doing this as a progress bar" (bliki "Xunit", 2006). JUnit's 1998 introduction, "Test Infected", describes the bar ("initially green but turns into red as soon as there is an unsuccessful test"). It does not use the phrase "red/green/refactor". That phrase appears in Beck's 2002 preface as "the TDD mantra". So the colours are documented as a UI convention from C3 and JUnit. Whether anyone used them as verbs before C3 is unknown.

**4. Refactoring came from the Smalltalk community, and tests are what made it safe.** Fowler: "The foundations of what we refer to these days as refactoring comes from the Smalltalk communities" (2003). Test Infected states the link in Beck's own co-signed words: "You will be able to refactor much more aggressively once you have the tests." The third step of the mantra is that link made routine: once the bar is green, you are allowed to clean up.

**5. The order of steps settled between 1994 and 2002.** Read in date order, Beck's texts show the rule forming:
- 1994: write tests, no order given.
- 1998: "code a little, test a little". The authors wrote their first test "immediately after" the code, yet they also advise "first write the test. Then run it (you never know, it might already work)."
- 2001: "Aim, Fire" in IEEE Software. Its abstract says "test-first coding is not testing", calls it "nearly as old as programming", and treats it as an analysis and design technique.
- 2002: the hard rule. Write new code "only if an automated test has failed."

A c2 wiki history page (unsigned) adds that the name changed from "Test-First Programming" to "Test-Driven Development" while that book was being written.

**What remains Beck's own.** On his own testimony, not the idea of writing expected output first. What is his, as far as these sources go, is the package. Tests are cheap enough to write one at a time, because they are Smalltalk-style objects the programmer writes. The tape book's ordering is applied at the scale of minutes instead of whole programs. Refactoring is made a required third step. The whole thing is reframed as design rather than testing. Calling that combination the distinctive part is my synthesis. Beck's sources support each piece, but none of them claims the combination as the invention. His 2023 Canon TDD also softens the 2002 form. It adds a step zero (write a list of test scenarios), makes refactoring "optional", and splits design into interface decisions, made while writing the test, and implementation decisions, made while refactoring.

**What you can now recognise.** The three words of the mantra come from three different places and periods:
- **Red** is the tape-era rule: expected output before the program. Beck documents the source but never names it.
- **Green** is a C3/JUnit screen colour, documented by Fowler.
- **Refactor** is Smalltalk-community practice that tests made safe.

You can also see why "the test" in TDD is shaped like a Smalltalk inspector session saved as a method. And you can see why Beck keeps insisting TDD is not a testing technique: in the tape version, the expected output was the specification. If you meet the McCracken or Mercury stories presented as "the origin of TDD", you can now tell they are real precedents that no source ties to Beck.

## Supporting detail

Inspected directly (fetched and read today):
- Beck, "Simple Smalltalk Testing: With Patterns", full text: Ron Jeffries' HTML transcription from xprogramming.com, via a Wayback Machine snapshot. The original venue (Smalltalk Report, October 1994) is from the Cambridge University Press chapter page for *Kent Beck's Guide to Better Smalltalk*, ch. 30, whose abstract opens "Smalltalk has suffered because it lacks a testing culture."
- "Test Infected: Programmers Love Writing Tests", full text from the JUnit 4 repository docs (raw.githubusercontent.com/junit-team/junit4/main/doc/testinfected/testing.htm). The page itself has no byline. The attribution to Beck and Gamma (Java Report 3(7), July 1998) comes from bibliographic search results, not the page.
- Beck, "Canon TDD", newsletter.kentbeck.com, 2023-12-11.
- Larman & Basili, "Iterative and Incremental Development: A Brief History", IEEE Computer, 2003 (PDF at mosaicprojects.com.au; text extracted locally).
- Wikipedia, "Test-driven development": the History section and its citation of Beck's Quora answer (2012-05-11).
- Martin Fowler bliki: "Xunit" (2006-01-17), "EtymologyOfRefactoring" (2003-09-10), "C3" (2004-08-03), "TestDrivenDevelopment" (2005, updated 2023-12-11), and the Refactoring book page.
- Gergely Orosz, "How Kent Beck shapes the software engineering industry", Pragmatic Engineer (page date 2026-07-01), plus the June 2025 podcast page (no transcript on it).
- Notes on the TDD book with page numbers (stanislaw.github.io, 2016) and a second summary (vivekkpatel.github.io). John Mullins' ACCU review (2004).
- IEEE "Aim, fire" bibliographic record with abstract (search.isc.ac): IEEE Software 18(5), 2001, pp. 87–89, DOI 10.1109/52.951502. The record lists the author as "Beck, Kristin", which looks like a metadata error.
- c2 wiki mirrors (kidneybone.com): XpRoots, TestFirstDesign, TestDrivenDevelopment, TenYearsOfTestDrivenDevelopment, SmalltalkUnit.
- Hiranabe's Japanese relay translation of SE Radio episode 167 (Beck on JUnit history, 2010), the SE Radio episode page, and Shchetinin's Habr article (2013) quoting a Beck Quora answer on the Workspace.
- Fernando Rodriguez, "Programming History", De Programmatica Ipsum (2019).

Not inspected:
- Beck's Quora answers themselves: Quora returned 403. The "rediscovery" text is Wikipedia's quotation of it. The Workspace answer is known only through Habr's Russian rendering.
- The TDD book itself: O'Reilly returned 403. Its wording comes from two sets of reader notes and a review.
- The full text of "Aim, Fire" (abstract only).
- McCracken (1957): the Internet Archive copy is lending-restricted, the full-text endpoint is unavailable, and the blog that quotes it (markusheilig.netlify.com) returned 404.
- No audio: the SE Radio and Pragmatic Engineer podcasts were not listened to.
- *Extreme Programming Explained* (1999): known only through Noel Rappin's 2026 reading of the first edition, which says it "doesn't quite stress testing first yet".

Sources studied, in their own right:
- **Tape-era batch programming** (as Beck describes it): input and output were physical tapes. A correct program was one whose output tape matched a prepared expected tape, so the expected result existed before the program.
- **Smalltalk practice c. 1987–1994**: a live image, Workspaces and Inspectors where you poke objects and read the answers. The 1994 column distrusted brittle UI tests and favoured fixtures created fresh for each test. Beck, via the SE Radio translation, says Smalltalk had a habit of making a small change and running it, but nothing automated.
- **C3 (1996–1999)**: the Chrysler payroll rewrite in Smalltalk, restarted under Beck in 1996. Fowler says it "first pulled together all the practices that became known as Extreme Programming". It is where the red/green window colouring is documented.
- **The Smalltalk refactoring community**: Opdyke and Johnson's coining of "refactoring" (Opdyke recalled a conversation with Johnson about a "Software Refactory") and the broader Smalltalk origin Fowler describes.

Connections:

| Shaping source | Trait carried forward, and how it changed | Evidence | Source |
| --- | --- | --- | --- |
| Unnamed "ancient book" on tape programming | Expected output typed before the program, program until it matches. Shrunk from whole-program tapes to one small automated check at a time | documented (Beck's own words; book unidentified) | Beck Quora 2012-05-11, quoted at en.wikipedia.org/wiki/Test-driven_development |
| Father's book, read in childhood | Same book, with the detail that it was his father's | documented second-hand (Orosz's summary, not a verified Beck quote) | newsletter.pragmaticengineer.com/p/how-kent-beck-shapes-the-software |
| McCracken, *Digital Computer Programming* (1957) | "Hand-calculated check case" before coding | unknown as an influence on Beck; the quote itself is unverified | De Programmatica Ipsum 2019 (claim only); book not readable |
| Project Mercury (early 1960s) | Tests planned and written before each micro-increment | interpretive (a parallel); the practice is documented by Larman & Basili, but no link to Beck was found | Larman & Basili 2003, p. 2 |
| Smalltalk interactive Inspector/Workspace testing | Interactive pokes at a fixture turned into a saved method per "predictable situation"; this became the xUnit test shape | documented | Beck 1994 column ("Test Case" pattern); Habr's rendering of Beck's Quora answer |
| SUnit, 1994 | The framework came before the test-first rule; Beck recalled the tape book "after" writing it | documented | Wikipedia quote of Quora; Canon TDD (SUnit 1994, demo Oct 1995) |
| Ward Cunningham / Tektronix | First person shown TDD (OOPSLA 1995). XP drew inspiration from 1980s Tektronix work with Cunningham. A specific TDD trait from Cunningham is not established | documented for the demo and the general inspiration; unknown for any specific trait | Canon TDD; Larman & Basili 2003 |
| C3 project | Red/green whole-window colouring, which became JUnit's progress bar and then the mantra's colours | documented (Fowler was a participant) | martinfowler.com/bliki/Xunit.html |
| JUnit / "Test Infected" (1998) | "Code a little, test a little"; "first write the test"; the green/red bar; "refactor much more aggressively once you have the tests" | composition (Beck's co-written text) | junit4 doc/testinfected |
| Smalltalk refactoring community | Refactoring as the third step, made safe by a green bar | documented for the general Smalltalk origin; the specific route into the mantra is interpretive | martinfowler.com/bliki/EtymologyOfRefactoring.html; Test Infected |
| Beck's anxiety about programming | Writing the test first made his "anxiety about programming vanish" | documented second-hand (Orosz); consistent with "less stressed" in Test Infected | Pragmatic Engineer 2026; Test Infected |

Secondary connections and context:
- Ward Cunningham is reported to have said "test-first is not a testing technique" (Keith Braithwaite, recalled second-hand on the c2 TestFirstDesign page). Unverified, but it matches the "Aim, Fire" abstract.
- An unsigned c2 comment likens the loop to Deming's PDSA cycle. That is interpretive only, and no Beck source draws the link.
- Fowler (2005/2023) calls skipping the refactor step "the most common way that I hear to screw up TDD".

Unresolved branches:
- Which book Beck read. He doesn't name it in any source read here. McCracken is a popular guess with no link to Beck.
- SUnit's date: 1994 (Canon TDD and the Cambridge venue) against 1989, which Wikipedia's SUnit article gives according to search snippets (not fetched). I used 1994.
- Whether red/green as words existed before C3's window colouring.
- How test-first was described in *Extreme Programming Explained* (1999). Only one reader's account was available.
- The unsigned c2 claim that Gerry Weinberg wrote tests first on punched cards. Weinberg's own account (via Larman & Basili) covers incremental development on Mercury, not test-first by him personally.
- Who chose the phrase "red/green/refactor". It is first seen here in Beck's 2002 preface. Whether he coined it or picked it up is not established.

What changes for the current task: none was stated. If you teach or critique TDD, Beck's 2023 Canon TDD is the version he defends ("If you plan on critiquing TDD & you're not critiquing the following workflow, then you're critiquing a strawman").
