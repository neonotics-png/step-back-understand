# Understanding Tailwind CSS through what shaped it

Sources checked 2026-11-03. I read Tailwind's current docs and blog, Adam Wathan's 2017 essay and his own history thread, and the essays and project pages of the approaches Tailwind came from or pushed against. I did not read Tailwind's source code and did not build anything with it, so claims about how it behaves come from its documentation.

**Subject and question.** Tailwind CSS: why does it put many small classes in the markup, why does it hang them on a design-token config, and why does that split developers so sharply?

---

## The explanation

### 1. Tailwind is the end point of an argument about which way dependencies run

Tailwind is easiest to understand as the conclusion of one essay, Adam Wathan's "CSS Utility Classes and 'Separation of Concerns'" (7 Aug 2017). Tailwind's first release followed that October. The essay describes five phases of his own CSS. He started with semantic classes like `.author-bio`. Then he moved through BEM and content-agnostic components (`.media-card`). Then came utilities like `.mar-r-sm`, which made the component class unnecessary, and finally utility-first. He says the turning point was Nicolas Gallagher's "About HTML semantics and front-end architecture" (2012).

**Gallagher, in his own right.** Gallagher was writing for teams that maintained large CSS codebases, and he made one move: he split *content* semantics from *class* semantics. "Content-layer semantics are already served by HTML elements and other attributes." A class is "a hook for CSS and JavaScript" and "should communicate useful information to developers." He concluded that "the most reusable components are those with class names that are independent of the content." Gallagher himself was still arguing for components with content-independent names, not for single-property utilities. Utility classes appear in his article only as a `.u-utilityName` naming pattern.

**What Wathan did with it.** He took Gallagher's licence to stop naming things after content and pushed it further than Gallagher did. He also reframed "separation of concerns" as a question of dependency direction:

- When the CSS depends on the HTML (semantic classes), the HTML can be restyled but the CSS cannot be reused.
- When the HTML depends on the CSS (utilities), the CSS can be reused but the HTML cannot be restyled from the stylesheet.

Wathan chose reuse. Every later fight about Tailwind is a fight over that choice. (documented: the essay names Gallagher and makes the dependency-direction argument.)

### 2. The utility class came from Atomic CSS and Tachyons; Tailwind made it readable and stateful

**Atomic CSS (Thierry Koblentz, Yahoo).** In "Challenging CSS Best Practices" (Smashing Magazine, 21 Oct 2013), Koblentz argued that separation of concerns produces bloated, redundant stylesheets. He proposed classes that each "map... to a single style", applied in the markup, so the work becomes "editing HTML templates instead of style sheets." He reported that My Yahoo was rebuilt this way with a 17.9 KB gzipped stylesheet. His names were cryptic abbreviations such as `Bfc M-10`, `Fl-start`, `Mend-10` and `Fz-s`. Critics called them "unsemantic", and his own editor's note warned that some of the techniques "are not considered to be best practices."

**Tachyons (Adam Morse) and Basscss.** These turned the idea into toolkits.

- **Tachyons** calls itself "a functional css toolkit for designing in the browser." It is built on "small, single-purpose classes", a spacing scale "based on the powers of two", and mobile-first namespaced responsive classes. Most selectors have a specificity of 10, so nothing overrides a class once it is applied. Its names are short: `f1`–`f6`, `ba`, `br2`, `bg-navy`, `w-50`.
- **Morse's essay** "CSS and Scalability" (30 Mar 2016) states the creed: "Content semantics have nothing to do with visual styles," "There should be a one to one mapping between a class and it's definition," and "The real way to scale CSS, is to stop writing CSS."
- **Basscss** calls itself "the original Functional CSS library" of "immutable utilities" with "No Side Effects". Its README credits Nicole Sullivan and Adam Morse among its inspirations.

**What Tailwind kept and changed.** Wathan's essay names Tachyons ("a fantastic project") and Basscss as utility-first frameworks worth using before it introduces Tailwind. That makes them documented prior art. He does not list which of their traits he borrowed, so the trait-by-trait comparison below is my reading of the docs:

- **Kept:**
  - One class does one thing.
  - Styling happens in the markup.
  - Values come from scales.
  - CSS stops growing as the project grows. Morse wrote "stop writing CSS"; Tailwind's docs say "your CSS stops growing."
- **Changed:**
  - **Readable names.** Tailwind spells names out (`text-xl`, `rounded-lg`, `bg-sky-700`), where Koblentz used `Fz-s` and Tachyons used `f4` and `br2`.
  - **Variant prefixes.** Its biggest change is the prefix that turns any utility into a state or breakpoint rule (`hover:bg-sky-700`, `sm:grid-cols-3`, and stacks like `sm:hover:active:disabled:opacity-75`). Tailwind's own docs use this to answer "isn't this just inline styles?": inline styles cannot express hover, focus or media queries, and utilities can.
  - **On-demand generation.** Since the JIT engine (15 Mar 2021), Tailwind generates CSS only for the classes you actually use. That made it practical to enable every variant and to allow arbitrary values like `top-[-113px]`. A fixed stylesheet like Tachyons' (under 14 KB gzipped) could not offer either.

### 3. The config is Refactoring UI's design advice turned into code

Tailwind's answer to "this is just inline styles" is mainly about constraints: "using inline styles, every value is a magic number. With utilities, you're choosing styles from a predefined design system." Wathan's 2017 essay said the same thing: "Utilities force you to choose" from a curated set, while "with inline styles, there are no constraints on what values you choose."

**Where the constraint idea comes from.** Wathan and the designer Steve Schoger had worked together since a 2015 side project (Wathan's history thread). They later wrote *Refactoring UI*. Its colour chapter teaches you to "define a fixed set of shades up front that you can choose from as you work", nine shades from 100 to 900, and to avoid generated lighten and darken functions: "That's how you end up with 35 slightly different blues that all look the same." Tailwind's palettes are that advice turned into code. Every colour has a fixed ladder of shades (now 50 to 950), and `bg-blue-500` names a step on the ladder, not a hex value. This link is a shared author on the record. That the book's advice directly produced the palette design is my inference; neither source says so.

**How the config works today (v4, released 22 Jan 2025).** The config moved from `tailwind.config.js` into CSS:

- A value inside an `@theme` block, such as `--color-mint-500: oklch(0.72 0.11 178)`, does two things:
  - It becomes a normal CSS variable on `:root`.
  - It makes utilities exist: `bg-mint-500`, `text-mint-500`, `fill-mint-500`.
- Namespaces decide what kind of utility each variable produces: `--color-*`, `--text-*`, `--font-*`, `--spacing`, `--radius-*`, `--shadow-*`. `--breakpoint-*` produces variants rather than utilities.
- You can extend the theme by adding variables, override it by redefining them, or delete whole families with `--color-*: initial`. The docs' rule of thumb is to use `@theme` "when you want a design token to map directly to a utility class" and `:root` for variables that should not.
- **Two shifts from v3 are worth noticing:**
  - **Tokens are now real CSS variables at run time.** In v3 they were build-time JavaScript values.
  - **Spacing is now one base unit, not a fixed list.** The default theme sets `--spacing: 0.25rem`, and the padding docs map `p-<number>` to `padding: calc(var(--spacing) * <number>)`. On the docs as written, any number is accepted, so the constraint is the unit rather than a curated list of values. I did not confirm whether fractional numbers are allowed or how this differs from v3's fixed scale in practice.

**The wider "design tokens" idea.** Tailwind's docs use the term "design token". The term itself comes from Salesforce's design system team. The W3C Design Tokens Community Group credits Jina Anne with coining it with Jon Levine at Salesforce, and that group published a first stable token spec in Oct 2025. In that tradition, a token is a named design decision stored once and distributed to every platform. Tailwind's `@theme` matches this: one named value feeding both CSS variables and utility classes. However, I found nothing in which Wathan cites Salesforce or the token community, so this parallel is interpretive, not a lineage.

### 4. Why people love it and why they hate it

With the sources in view, the love and the hate turn out to be two answers to Wathan's dependency question, plus one honest leak.

**The love comes from the Gallagher → Morse → Wathan line.** Tailwind's docs list the payoffs:

- No class naming.
- No switching files.
- Changes only affect the element you edit.
- Chunks of UI can be copied between projects.
- The CSS stops growing.

The State of CSS 2024 tools page shows Tailwind as the most-used framework in its sample (3,657 respondents, against 2,623 for Bootstrap). I did not obtain that page's satisfaction figures. Wathan's Rails World talk (6 Oct 2023) has the honest title "Tailwind CSS: It looks awful, and it works" and concedes "there's literally no words I could possibly say with my mouth that will convince you." He argues instead that "there's no such thing as an HTML methodology," so style kept in markup has nothing to rot.

**The hate comes from the opposite side of the same choice, and from the places where the abstraction leaks.**

- **Inline-styles objection.** Critics read long class lists as inline styles. Builder.io's round-up (Yoav Ganbar, 25 Jan 2023) concedes Tailwind "bloats HTML markup and looks like inline styles". It also quotes Jason Miller saying it made later changes like dark mode "more arduous/brittle".
- **Leaky abstraction.** An undated, unsigned critique on andros.dev makes the sharpest technical point. Class order in the markup does not decide which rule wins ("The markup lies to you"), and arbitrary values like `p-[5px]` bypass the scales: "The system helps you, but it does not save you from yourself." That second point turns Tailwind's own constraint argument against it. Brackets were added in 2021, so the "no magic numbers" promise is a default, not a guarantee.
- **`@apply` and the docs.** Many teams fall back on `@apply` to name a component from utilities. Tailwind's own v3 docs warn against this: "don't use `@apply` just to make things look 'cleaner'". Doing so brings back naming, file switching, global risk and a bigger bundle. The v4 page on duplication points to loops, framework components and template partials instead.

So the debate is coherent once you see this:

- Tailwind only fully pays off when you have a component or partial system that holds each class list once. That is the part of Gallagher's "component" idea Tailwind kept, moved out of CSS and into templates.
- Without that system, you get the repetition critics see.
- With `@apply` everywhere, you get semantic CSS back, and both camps' downsides together.

### What remains distinctive about Tailwind

Single-purpose classes predate Tailwind (Koblentz, Tachyons, Basscss), and so does the idea that named scales should fix the values (Refactoring UI, design tokens). On this reading, three things are Tailwind's own:

- **Combination.** It joins utilities to a user-owned token config, so your design system defines which classes exist.
- **Variants.** Its prefix grammar lets one class name express state, breakpoint, dark mode and container size.
- **Engine.** It generates only the classes you use (JIT, then v4's engine), which makes an unlimited utility space affordable.

Whether these are firsts, or merely the best-known versions, this pass did not establish.

### What you can now recognize

- **The two camps.** When someone says Tailwind "violates separation of concerns", you can name the move being disputed: Wathan's reversal of dependency direction, which grew out of Gallagher. That tells you the argument is about reuse versus restyling, not cleanliness.
- **Class names.** Looking at `md:hover:bg-sky-700`, you can see three layers:
  - the atomic-CSS core (`bg-…`), which goes back to Koblentz and Tachyons;
  - a token from the theme (`sky-700`, a Refactoring-UI-style fixed shade);
  - Tailwind's own variant prefixes (`md:hover:`).
- **Arbitrary values.** Brackets like `p-[13px]` are the spots where a codebase has stepped outside its design system. That is a fair code-review signal.
- **`@apply`.** Heavy use of `@apply`, or long class lists repeated with no component around them, means the codebase is getting the costs of Tailwind without the part that makes it work.

---

## Supporting detail

**Inspected directly (via web fetch):**
- Tailwind docs:
  - "Styling with utility classes": https://tailwindcss.com/docs/styling-with-utility-classes
  - "Theme variables": https://tailwindcss.com/docs/theme
  - "Padding": https://tailwindcss.com/docs/padding
  - v3 "Reusing styles": https://v3.tailwindcss.com/docs/reusing-styles
  - v3 "Theme configuration": https://v3.tailwindcss.com/docs/theme
- Tailwind blog:
  - v4.0 release post (22 Jan 2025): https://tailwindcss.com/blog/tailwindcss-v4
  - JIT post (15 Mar 2021): https://tailwindcss.com/blog/just-in-time-the-next-generation-of-tailwind-css
- Adam Wathan, "CSS Utility Classes and 'Separation of Concerns'" (7 Aug 2017): https://adamwathan.me/css-utility-classes-and-separation-of-concerns/
- Wathan's Tailwind history thread (undated on the page; refers to events through Feb 2020): https://threadreaderapp.com/thread/1289645193038192641.html
- Nicolas Gallagher (15 Mar 2012): https://nicolasgallagher.com/about-html-semantics-front-end-architecture/
- Thierry Koblentz (21 Oct 2013): https://www.smashingmagazine.com/2013/10/challenging-css-best-practices-atomic-approach/
- Tachyons homepage: https://tachyons.io/
- mrmrs, "CSS and Scalability" (30 Mar 2016): https://mrmrs.cc/writing/scalable-css/
- Basscss README: https://github.com/basscss/basscss
- Refactoring UI colour chapter preview: https://www.refactoringui.com/previews/building-your-color-palette
- W3C DTCG post (17 Jun 2026): https://www.w3.org/community/design-tokens/2026/06/17/thank-you-jina-and-val/
- Critiques and debate:
  - Builder.io (Yoav Ganbar, 25 Jan 2023): https://www.builder.io/blog/the-tailwind-css-drama-your-users-don't-care-about
  - andros.dev (author and date not shown): https://en.andros.dev/blog/af3ee191/wh/
  - Rails World 2023 talk summary page: https://www.rubyevents.org/talks/tailwind-css-it-looks-awful-and-it-works
- State of CSS 2024 tools page: https://2024.stateofcss.com/en-US/tools/

**Not inspected:**
- Tailwind's source code and the v0.1 repository.
- The Rails World talk video. Its quotes come from the event page's summary and transcript excerpts.
- The *Refactoring UI* book itself. The spacing-chapter preview URL returned 404, so no spacing-system claim from the book is used.
- Satisfaction and sentiment figures from State of CSS.
- Yahoo's Atomizer project.

**Connections:**

| Shaping source | Trait carried forward, and how it changed | Evidence | Source |
| --- | --- | --- | --- |
| Gallagher (2012) | Class names need not describe content. Wathan pushed this from content-independent components to single-property utilities, and reframed separation of concerns as dependency direction. | documented | Wathan 2017 essay; Gallagher 2012 |
| Semantic CSS / BEM | The stages Wathan went through and rejected. BEM's low specificity and flat classes survive in spirit as single-class utilities. | documented (rejection); interpretive (what survives) | Wathan 2017 essay |
| Tachyons, Basscss | Single-purpose, immutable, scale-based classes in markup. Tailwind made the names readable, added variant prefixes and made the scales configurable. | documented as named prior art; trait mapping interpretive | Wathan 2017 essay; tachyons.io; Basscss README |
| Koblentz / Atomic CSS at Yahoo | One class, one style; styling in templates; small shared stylesheet. Tailwind replaced cryptic abbreviations with readable names. | interpretive: Wathan's essay does not name Atomic CSS or Koblentz | Smashing 2013 |
| Morse, "CSS and Scalability" | "Stop writing CSS"; CSS should not grow with features. The same claim appears in Tailwind's benefits list. | interpretive: not cited by Wathan in the sources read | mrmrs.cc 2016 |
| Inline styles | The foil. Tailwind defines itself against them through constraints, states and media queries. | documented | Tailwind docs; Wathan 2017 |
| Refactoring UI / Steve Schoger | Fixed palettes defined up front (100–900) become Tailwind's 50–950 colour ladders and the theme. | composition (shared author, documented collaboration); the specific design link is interpretive | Refactoring UI preview; Wathan thread |
| Less, then PostCSS/JS | Tailwind began as Less with mixins, made "as configurable as I wanted", then moved to PostCSS and JavaScript (hence `tailwind.config.js`). v4 moves config back into CSS. | documented (history); interpretive ("back into CSS" as a return) | Wathan thread; v4 blog |
| Design tokens (Salesforce, DTCG) | Named design decisions stored once and distributed. `@theme` turns each one into both a CSS variable and utilities. | interpretive: no citation from Tailwind's authors found; Tailwind docs only use the term | W3C DTCG post; Tailwind theme docs |
| JIT engine (2021) | On-demand generation made all variants and arbitrary values possible. It also opened the escape hatch critics say weakens the constraint promise. | documented (feature); interpretive (effect on constraints) | JIT blog post; andros.dev critique |

**Unresolved branches:**
- Whether Wathan read Koblentz or Morse directly. No statement from him on this was found.
- What Tailwind's exact v0.1 config looked like, and when the config file arrived. The history thread does not mention it.
- Whether v4's multiplier-based spacing (`p-<number>`) accepts fractional values, and how strongly that loosens v3's fixed scale in practice.
- Official State of CSS satisfaction and sentiment numbers for Tailwind. Secondary reports exist, but I did not verify them.
- Two details that could not be confirmed:
  - The 2014 date for the coining of "design tokens" appears in only one blog.
  - Accounts disagree on what the KiteTail project was.
- Whether variant prefixes like `hover:` and `md:` were Tailwind's invention or borrowed. Tachyons used namespaced responsive suffixes, but I did not trace who first used prefixes for state.

**What changes for the current task:** There is no current task beyond understanding. If you are choosing Tailwind for a project, the deciding question from this analysis is whether you have a component or partial system that holds each class list once. If you do, you get the payoff Wathan describes. If you don't, expect the problems the critics describe.
