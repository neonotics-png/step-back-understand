# Test-Driven Development (Kent Beck): Answer Key

## 1. Why watch the test fail first

**Model answer:** Seeing the test fail (red) checks the test itself. It shows that the test can detect the missing behaviour, that it really runs, and that it fails for the expected reason rather than because of a typo or a setup error. If it passes at once, either the behaviour already exists or the test is broken, for example by asserting nothing or not exercising the code. Either way, its later "green" tells you nothing. The failing test is also the only thing that licenses new production code, so every line of new code exists because a test required it.

**Key points for full credit:**
- The red step verifies the test: a test you never saw fail might not be able to fail, so its passing proves nothing.
- It should fail for the right reason (the behaviour is missing), not because of a compile error, a typo or a setup problem. An unexpected pass is information: the behaviour already exists or the test is wrong.
- A failing test is the trigger and justification for writing new code, which is why the code ends up covered by tests.

**Sources:** https://stanislaw.github.io/2016-01-25-notes-on-test-driven-development-by-example-by-kent-beck.html (the book's two rules and the definition of red); https://martinfowler.com/bliki/TestDrivenDevelopment.html ("we can only write some functional code in response to making a test pass")

## 2. Committing sins to get green

**Model answer:** Green's only job is to get a working state back quickly, so that you are back on solid ground with all tests passing. Shortcuts such as hard-coded values are fine because the refactor step that follows will clean them up while the tests protect you. Trying to design cleanly while the bar is red means "wearing two hats": you solve "does it work?" and "is it well structured?" at once. That makes steps bigger, keeps you red for longer, and when something breaks you can't tell which change caused it. Beck's rule is "make it run, *then* make it right."

**Key points for full credit:**
- Getting to green fast restores a known-good, all-passing state. That safe state is what makes the next steps cheap.
- Making it work and making it clean are separate activities (the "two hats"). Mixing them is a mistake Beck names in Canon TDD.
- The "sins" are paid back in the refactor step, with passing tests as the safety net. They are not left in the code.

**Sources:** https://newsletter.kentbeck.com/p/canon-tdd ; https://stanislaw.github.io/2016-01-25-notes-on-test-driven-development-by-example-by-kent-beck.html (green: "Make the test work quickly, committing whatever sins necessary")

## 3. What the refactor step does

**Model answer:** Refactoring improves the structure of the code without changing its behaviour. In Beck's book its specific job is to "eliminate all of the duplication created in merely getting the test to work," including duplication between the test and the code, such as the hard-coded constant that copies the expected value. Removing that duplication is what turns a fake into a general solution. Since behaviour doesn't change, no new failing test is needed. The existing tests must stay green the whole time and are what make the change safe. Skip refactoring and you get code that passes its tests but is a pile of special cases and patches, which gets harder and harder to extend.

**Key points for full credit:**
- Refactoring changes structure, not behaviour. It removes duplication (including test-to-code duplication) and is where implementation design happens.
- It needs no new red test because behaviour is unchanged. The existing green tests are the safety net, and new behaviour waits for a new failing test.
- Without it, the code becomes a "messy aggregation of code fragments": it works, but is badly designed and gets harder to change.

**Sources:** https://martinfowler.com/bliki/TestDrivenDevelopment.html ; https://stanislaw.github.io/2016-01-25-notes-on-test-driven-development-by-example-by-kent-beck.html (refactor definition and the "eliminate duplication" rule)

## 4. The test list vs. writing all tests first

**Model answer:** Canon TDD starts with a test list: a short written list of the behaviours and edge cases the change should cover, including existing behaviour that must not break. You then turn exactly one item into a runnable test, make it pass, optionally refactor, and repeat. You add new items as you discover them and cross off the ones that pass. Beck lists "convert all the items on the Test List into concrete tests, then make them pass one at a time" as a mistake. Each implementation step teaches you about the interface and the design, so tests written early have to be rewritten (rework). Facing a wall of red tests also leads to "depression and/or boredom". The order in which you take on the tests also shapes the design and the experience.

**Key points for full credit:**
- Up front you write a list of test scenarios, which is behavioural analysis, not concrete test code. Writing all the tests first is a strawman of TDD.
- Only one item becomes a runnable test at a time. The list is a living document: add what you discover, cross off what passes.
- Writing every test first causes rework, because early tests lock in interface guesses that later learning changes. It also loses the short feedback loop, and test order affects the result.

**Source:** https://newsletter.kentbeck.com/p/canon-tdd

## 5. Fake It, Triangulate, Obvious Implementation, and step size

**Model answer:** If the implementation is obvious and you're confident, just type it (Obvious Implementation, "second gear"). If you're unsure, fake it by returning a constant, then remove the duplication between the constant and the test until the real code appears. If you really don't know the right abstraction, triangulate: add a second example that forces you to generalize. When you hit an unexpected red bar, shift down to smaller steps. When confidence returns, shift back up. So TDD isn't a requirement to take tiny steps. It's the ability to take tiny steps when uncertainty calls for it, with step size adjusted to how confident you are.

**Key points for full credit:**
- Choose by confidence or uncertainty. Use Obvious Implementation when sure, Fake It or Triangulate when unsure. Triangulate means generalizing only when two or more examples force it.
- Step size changes during the work. Downshift after a surprise failure, upshift as confidence returns.
- The point is being *able* to take tiny steps, not being required to take them all the time.

**Sources:** https://stanislaw.github.io/2016-01-25-notes-on-test-driven-development-by-example-by-kent-beck.html ; https://www.markhneedham.com/blog/2009/05/24/tdd-making-the-test-green-quickly/

## 6. Interface vs. implementation design

**Model answer:** Writing a test first means writing the calling code before the code exists. So you have to decide how the behaviour will be invoked: names, parameters, return values, and what the caller needs to set up and observe. Those are interface decisions, made from the point of view of a user of the code. How the behaviour is achieved internally (data structures, algorithms, how responsibilities are split) is implementation design, which belongs to the refactor step once the behaviour is pinned down by a passing test. The split helps because you get the interface right without being biased by internals. Implementation choices can then change freely under the protection of tests that depend only on the interface.

**Key points for full credit:**
- The test-writing step forces interface decisions: how a client calls and observes the behaviour.
- Implementation design is deliberately deferred to refactoring, and kept out of the test list as well.
- Keeping them separate yields usable interfaces and leaves the implementation free to change. Fowler calls separating interface from implementation a key element of good design.

**Sources:** https://newsletter.kentbeck.com/p/canon-tdd ; https://martinfowler.com/bliki/TestDrivenDevelopment.html

## 7. Tests that fight refactoring

**Model answer:** The loop depends on refactoring being safe and cheap. Tests should fail when behaviour changes and stay green when only structure changes (Beck's "behavioral" and "structure-insensitive" desiderata). Heavy mocking ties tests to which collaborators get called and how, so restructuring breaks them even when behaviour is preserved. People then avoid refactoring, and Beck noted that heavy mockers often find refactoring hard. Pasting the code's own computed output into the expected values removes the independent check that gives a test its value: the test just records whatever the code does, bugs included. Their tests should check observable behaviour through the interface, with expected values worked out independently, and use mocks sparingly.

**Key points for full credit:**
- Tests coupled to structure or implementation (for example through heavy mocking) break under behaviour-preserving changes. That makes refactoring expensive, so it gets skipped.
- Copying computed values into expected values is a mistake Beck names in Canon TDD: it defeats the double-check, and the test can no longer catch a wrong answer.
- Good tests are behavioural and structure-insensitive (as well as fast, deterministic and specific). Mocks should be used sparingly, not by default.

**Sources:** https://testdesiderata.com/ ; https://newsletter.kentbeck.com/p/canon-tdd ; https://www.martinfowler.com/articles/is-tdd-dead/

## 8. What TDD is for, and when Beck uses it

**Model answer:** For Beck, TDD is a programming and design workflow, not mainly a testing technique. Its goal is "clean code that works" and the confidence that it works, managing fear until "fear ... has been transmuted into boredom." Coverage is a side effect, not the goal. Beck calls test-to-code ratios "a bogus metric," says to "test as little as possible to reach a given level of confidence," and recommends deleting redundant tests, because the same thing tested several ways is coupling, and coupling costs. He describes TDD as one way to get confidence, not the only one. He uses it where problems are "TDDable" and uses other feedback loops (for example regression tests, short feedback cycles, production signals) elsewhere.

**Key points for full credit:**
- The purpose is working, well-designed code and justified confidence (managing fear), with fast feedback. It is not maximizing tests or coverage.
- More tests isn't automatically better. Test enough for confidence, since redundant tests add coupling and cost.
- TDD is one tool among several. Beck applies it where it fits, mixes it with other feedback methods, and says there is "no gold star for following these steps exactly."

**Sources:** https://www.martinfowler.com/articles/is-tdd-dead/ ; https://newsletter.kentbeck.com/p/canon-tdd ; https://stanislaw.github.io/2016-01-25-notes-on-test-driven-development-by-example-by-kent-beck.html
