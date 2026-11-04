# Test-Driven Development as Kent Beck formulated it

## 1. The short version

Kent Beck set out TDD in *Test-Driven Development: By Example* (Addison-Wesley, 2002). It grew out of the "test-first programming" practice in Extreme Programming in the late 1990s. Beck says he *rediscovered* it rather than invented it: as a child he read an old programming book that told you to take a real input tape, type out the output tape you expected by hand, and then program until the actual output matched.

The whole method comes down to **two rules**:

1. **Write new code only if an automated test has failed.**
2. **Eliminate duplication.**

Those two rules produce the familiar rhythm, which Beck calls the "mantra of TDD":

> **Red** — Write a little test that doesn't work, and perhaps doesn't even compile at first.
> **Green** — Make the test work quickly, committing whatever sins necessary in the process.
> **Refactor** — Eliminate all of the duplication created in merely getting the test to work.

The goal is **"clean code that works"**, a phrase from Ron Jeffries. The point of TDD is to split that goal in two. First get code that *works* (red to green). Then make it *clean* (refactor). You never try to do both at once.

---

## 2. The loop, step by step

In the book Beck gives the cycle as five steps:

1. **Quickly add a test.**
2. **Run all tests and see the new one fail.**
3. **Make a little change.**
4. **Run all tests and see them all succeed.**
5. **Refactor to remove duplication.**

Then repeat, many times an hour.

### Red: write one small failing test

- You write the test **before** the code it exercises, even if it refers to classes or methods that don't exist yet. A compile error counts as red.
- The test describes **one small piece of behavior**, the next thing you want the code to do. It is not a full specification.
- You **run it and watch it fail**. This step isn't ceremony:
  - It proves the test can fail. A test that passes before you've written anything is testing nothing, or the feature already exists.
  - It checks that it fails **for the reason you expect**. "Expected 10 but got 0" is the right red. "NullPointerException in the setup" is the wrong red, and it's telling you something else.
- Beck often writes the **assertion first** and works backward: what result do I want, then what objects and calls do I need to get it?

### Green: make it pass, as fast as possible

Here Beck explicitly allows "sins": hard-coded return values, copy-paste, ugly conditionals. The only goal is to get back to a passing state quickly, because while you're green you have a safety net and while you're red you don't. He describes three strategies for getting to green:

| Strategy | What you do | When |
|---|---|---|
| **Fake It ('Til You Make It)** | Return a constant that makes the test pass, then slowly replace constants with variables | When you aren't sure how to implement it, or want the smallest possible step |
| **Triangulate** | Generalize only once you have two or more examples that force it | When you can't yet see the right abstraction |
| **Obvious Implementation** | Just type in the real solution | When it's obvious and you're confident. If you get a surprise red, drop back to smaller steps |

### Refactor: remove duplication with the tests green

- With every test passing, you improve the design **without changing behavior**, running the tests after each small change.
- In Beck's formulation, **duplication** is the main force behind design, and that includes **duplication between the test and the code**. If the test says `assert 10 == five.times(2).amount` and the code says `return 10`, the number 10 appears twice. Removing that duplication (10 is really `5 * 2`) is what turns the fake into a real, general implementation. This is the subtle core of the method. Generalization is driven by removing duplication, not by guessing ahead.
- You don't add new behavior during refactoring. New behavior needs a new red test first.

---

## 3. The test list: the step people forget

Before and during the loop, Beck keeps a **to-do list of tests** on paper: the scenarios he thinks the code will need to handle.

- Pick **one** item from the list, preferably one that teaches you something and that you're confident you can make work.
- When new ideas, edge cases or refactorings come up mid-cycle, **write them on the list instead of acting on them**. That keeps each cycle small and focused.
- Cross items off as they go green. You're done when the list is empty.

In his 2023 essay **"Canon TDD"** (tidyfirst.substack.com/p/canon-tdd), Beck restated the workflow precisely, because he felt critics were attacking versions of TDD he never described:

1. Write a **list** of the test scenarios you want to cover.
2. Turn **exactly one** item on the list into an actual, concrete, runnable test.
3. Change the code to make the test (and all previous tests) pass, **adding items to the list** as you discover them.
4. **Optionally** refactor to improve the implementation design.
5. Until the list is empty, go back to step 2.

He also listed the common mistakes:
- **Writing all the tests up front** before any code. That isn't TDD. You'd lose the feedback each test gives you about the design, and you'd have to rework a pile of tests when the first one teaches you something.
- **Copying the actual computed output into the expected value** of an assertion. That defeats the double-check the test is supposed to give you.
- **Mixing refactoring into the "make it pass" step.** Wear one hat at a time.
- **Refactoring further than you need** for this cycle, or skipping refactoring entirely.

---

## 4. Why write the test *first*?

This is the rule that makes TDD different from "having good tests". The reasons, in roughly the order Beck stresses them:

### a) It forces interface design before implementation design

When you write the test first, you're the **first client** of your own code. You have to decide what the call looks like, what it's named, what it takes and what it returns, before you're tangled up in how it works. Beck separates these two decisions on purpose: the test is where you design the **interface**, and green plus refactor is where you design the **implementation**. Code that's awkward to test is usually awkwardly coupled, and writing the test first shows you that immediately, while it's still cheap to change.

### b) It's the only way to know the test actually tests something

A test written after the code has never been seen to fail. It may be checking nothing, or checking the wrong thing. Red-then-green is a tiny experiment that confirms both the test and the code.

### c) It keeps scope and step size under control

You only write code that a failing test requires. That stops speculative features ("we might need this"). The test also defines when you're done: when it's green, stop.

### d) It manages fear and keeps feedback short

Beck describes TDD as **"a way of managing fear during programming"**, and more generally as **an awareness of the gap between decision and feedback, plus techniques to control that gap**. Every few minutes you're back to all-green, so you always know where you stand. When you're unsure, take tiny steps. When you're confident, take bigger ones. His image is a well-pump bucket or shifting gears: the tests let you choose your step size, and they don't force you to always crawl.

### e) Tests written afterward tend not to get written

Beck describes a vicious circle: the more stress you feel, the less testing you do; the less testing you do, the more errors you make; the more errors, the more stress. Writing tests first breaks that circle, because testing becomes part of how you write code instead of a chore at the end that gets cut under deadline pressure.

### f) Side effects: a regression suite and executable documentation

You end up with a full automated test suite and examples that show how to use every unit. Beck treats these as valuable byproducts, but they aren't the main reason. **TDD is mainly a design and development discipline, not a testing technique.**

---

## 5. A worked example (in the spirit of Beck's Money example)

Part I of the book builds a multi-currency `Money` class. Here is the start of it in Python, one cycle at a time.

**Test list:** `$5 * 2 = $10`; `$5 + 10 CHF = $10 if the rate is 2:1`; equality; rounding?; null?

**Cycle 1: Red.** Write the test before `Dollar` exists:

```python
def test_multiplication():
    five = Dollar(5)
    product = five.times(2)
    assert product.amount == 10
```
It fails: `NameError: Dollar`. That's red.

**Green: Fake It.** Commit the sin:

```python
class Dollar:
    def __init__(self, amount):
        pass
    def times(self, multiplier):
        result = Dollar(0)
        result.amount = 10      # hard-coded
        return result
```
It passes, so you're green.

**Refactor: remove duplication.** The `10` in the code duplicates the `10` in the test. Where does 10 come from? It's 5 * 2. Make that duplication explicit, then remove it:

```python
class Dollar:
    def __init__(self, amount):
        self.amount = amount
    def times(self, multiplier):
        return Dollar(self.amount * multiplier)
```
Run the tests. Still green. Generalization came from removing duplication, not from a second test.

**Cycle 2:** Pick the next item, `equality`. Write `assert Dollar(5) == Dollar(5)`. Red. Fake it with `return True`. Then **triangulate** with `assert Dollar(5) != Dollar(6)`, which forces the real comparison `self.amount == other.amount`. Green. Refactor. Cross it off.

As you go, new ideas ("should `times` have side effects?", "Franc will duplicate Dollar") go **onto the list**, not straight into the code.

---

## 6. Common misunderstandings

| Misconception | What Beck actually says |
|---|---|
| "TDD means writing all the tests first" | One test at a time. Write a list of scenarios, but turn only one into code per cycle. |
| "Refactoring is optional polish" | In the 2002 book it's the third beat of every cycle, and it's how duplication drives the design. ("Canon TDD" marks it optional per cycle, but skipping it habitually leaves you with fast-written, messy code.) |
| "Green should be elegant" | Green should be **fast**. Elegance is the refactor step's job. |
| "Every line needs a test; aim for 100% coverage" | Beck's often-quoted Stack Overflow answer: *"I get paid for code that works, not for tests, so my philosophy is to test as little as possible to reach a given level of confidence."* The rule is about *new behavior* being driven by a failing test, not about coverage numbers. |
| "TDD is a testing practice" | It's a design and feedback practice. The tests are a means and a byproduct. |
| "TDD means unit tests of every class in isolation with heavy mocking" | Beck's tests are about **behavior** and are written at whatever level gives useful feedback. Tests coupled to implementation details make refactoring harder, which works against the whole loop. (This was a central theme of the 2014 "Is TDD Dead?" conversations between Beck, Martin Fowler and David Heinemeier Hansson.) |
| "Steps must always be tiny" | Step size is a dial. Go small when uncertain, and go faster with Obvious Implementation when confident. |

---

## 7. How to actually build the habit

1. Keep a **paper test list** next to you. Every distracting idea goes on it.
2. Before writing any production line, ask: **"Which failing test demands this?"** If none does, write that test first.
3. **Always watch it fail**, and read the failure message. Is it the failure you expected?
4. Get to green by the **shortest path**, even if it's a constant.
5. Then ask: **"What's duplicated?"** That includes between test and code. Remove it in small steps, running tests each time.
6. Commit when green. Keep each cycle to minutes, not hours.
7. If you're stuck red for long, **revert to the last green and take a smaller step**.

Practice on small katas (Bowling Game, Roman Numerals, String Calculator) until the rhythm feels automatic. Then use it on real code.

---

## Sources

- Kent Beck, *Test-Driven Development: By Example*, Addison-Wesley, 2002 (Preface; Part I "The Money Example"; Part III "Patterns for Test-Driven Development": Test List, Assert First, Fake It, Triangulate, Obvious Implementation).
- Kent Beck, "Canon TDD", *Software Design: Tidy First?* newsletter, December 2023: https://tidyfirst.substack.com/p/canon-tdd
- Kent Beck, *Extreme Programming Explained*, 1999 (test-first programming as an XP practice).
- Martin Fowler, "Test Driven Development" (bliki): https://martinfowler.com/bliki/TestDrivenDevelopment.html
- "Is TDD Dead?" conversations (Beck, Fowler, Hansson), 2014: https://martinfowler.com/articles/is-tdd-dead/
