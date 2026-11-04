# Test-Driven Development (Kent Beck): Comprehension Quiz

1. Beck's first rule is "write new code only if an automated test has failed." Why does it matter that you actually run the new test and see it fail before writing the code, and what would you be missing if it passed the first time you ran it?

2. In the green step Beck says to make the test pass quickly, "committing whatever sins necessary," such as returning a hard-coded constant. Why is it acceptable, and even preferred, to write deliberately bad code at this point, and what would go wrong if you tried to write the clean, final design while getting to green?

3. Martin Fowler calls neglecting the refactor step the most common way to get TDD wrong. What exactly is the refactor step meant to remove or improve, why doesn't it need a new failing test of its own, and what does a codebase look like after a long run of red-green with no refactoring?

4. A common critique of TDD is "I can't write all the tests before I write any code." Explain why this describes something Beck explicitly rejects. What do you write up front instead, how is it used during the loop, and why does Beck advise against turning all of it into runnable tests at once?

5. Beck describes three ways to get to green: Fake It, Triangulate, and Obvious Implementation. How does a practitioner choose between them as they work, and what does this show about the claim that "TDD means taking tiny steps"?

6. Beck separates "interface design" from "implementation design" and places them in different steps of the loop. Which design decisions does writing the test first force, which ones are left for later, and why is that split useful?

7. A team practises test-first, but every refactoring breaks dozens of tests even though behaviour hasn't changed. Their tests mock most collaborators and sometimes assert values pasted in from the code's own output. Using Beck's ideas, explain why this undermines the red-green-refactor loop and what properties their tests should have instead.

8. Someone says "TDD is a testing technique; its goal is maximum test coverage, and real practitioners apply it to every line of code." How would Beck correct this? Say what he considers TDD's actual purpose and how he decides when, and how much, to use it.
