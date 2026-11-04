# Tailwind CSS: Comprehension Quiz

Answer each question in 2-4 sentences.

1. Adam Wathan argues that "separation of concerns" is the wrong way to frame the HTML/CSS debate and proposes "dependency direction" in its place. Explain the two directions, what each one gives up, and which kind of project is a poor fit for Tailwind under this framing.

2. Critics often call Tailwind "inline styles with extra steps." What do utilities actually do that inline styles cannot, and which part of the critique still holds even after you account for those differences?

3. In Tailwind v4, what happens when you add `--color-brand-500: oklch(...)` inside an `@theme` block, and how is that different from declaring the same variable under `:root`? Why does it matter that theme values are emitted as real CSS custom properties rather than only being used at build time?

4. A developer writes `` className={`bg-${color}-500`} `` and the background never appears, even though `bg-red-500` is used and works fine elsewhere. Explain why this happens, what the idiomatic fix is, and what this tells you about how Tailwind decides which CSS to generate (including why "your CSS stops growing").

5. The same 12-class button markup appears in 30 places across a project. Describe Tailwind's recommended ways of dealing with this, and explain why the docs warn against reaching for `@apply` just to make markup look cleaner.

6. An element has `class="grid flex"` and renders as a grid. Explain why the order of classes in the attribute doesn't decide the result, and why this becomes a real problem for a reusable `<Button className=...>` component that accepts style overrides. How do teams usually deal with it?

7. A critic says: "With Tailwind, rebranding from blue to red means editing a thousand class lists." When is this criticism true, when is it false, and which Tailwind mechanisms address value-level changes versus pattern-level changes?

8. Why does Tailwind's "styles as plain class strings" model make tooling and major-version upgrades harder than with plain CSS or typed CSS-in-JS? Use a concrete example from the v3-to-v4 upgrade.
