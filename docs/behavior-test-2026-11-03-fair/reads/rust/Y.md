# Rust's borrow checker, through what shaped it

Sources checked 2026-11-03. No Rust compiler was available on this machine, so no code was compiled. Error codes and messages below are as quoted in the Rust Book, and the short code sketches are mine, written to the Book's rules.

**Subject and question.** Rust's borrow checker: the ownership, borrowing and lifetime rules it enforces. The open question is why the rules are these and not others. Why does `let s2 = s1;` kill `s1`? Why can you have many `&T` or one `&mut T`, but never both? And what is `'a` actually naming?

## The explanation

### 1. The rules as Rust states them

The Book's ownership rules: "Each value in Rust has an owner. There can only be one owner at a time. When the owner goes out of scope, the value will be dropped." Its borrowing rules: "At any given time, you can have either one mutable reference or any number of immutable references. References must always be valid." Lifetimes are how the second borrowing rule gets checked. "Every reference in Rust has a lifetime, which is the scope for which that reference is valid", and annotations "describe the relationships of the lifetimes of multiple references to each other without affecting the lifetimes."

Rust's own Reference says plainly that "Rust is not a particularly original language". On ownership and borrowing it names two sources: "C++: references, RAII, smart pointers, move semantics" and "ML Kit, Cyclone: region based memory management". Studying those two, plus linear types and Rust's own 2012–2022 history, explains almost every rule above.

### 2. C++ gave the "owner" and the "drop". Rust made the move a compile-time fact

**What C++ is like here.** In C++, RAII ties a resource's release to the end of an object's lifetime. `unique_ptr` and move semantics let one owner hand a resource to another. But a C++ move is a convention, not a guarantee. `std::move` "is exactly equivalent to a `static_cast` to an rvalue reference type"; it moves nothing itself. Standard-library objects that have been moved from are left "in a 'valid but unspecified state'", so you can still use them and their destructors still run. (cppreference, `std::move`.)

**What Rust kept and what it changed.** Rust kept the owner-releases-at-scope-end rule, and the Book names RAII as the model for `drop`. *(documented)* What Rust changed is what happens at the source of a move. In Rust a move is a typing event. After `let s2 = s1;` for a `String`, the compiler treats `s1` as dead: using it is `error[E0382]: borrow of moved value: s1`, and no destructor runs for it. The Book gives the reason in one line: if both `s1` and `s2` freed the buffer you would get "a double free error". When the compiler can't tell statically whether a value was moved, for example after a conditional move, it falls back to a hidden "drop flag" on the stack (Rustonomicon, *Drop Flags*). Even then the rule is the same: a moved-from value is never touched or dropped again.

That one change is the whole of rule two ("one owner at a time"). C++ gives you single ownership as a library discipline. Rust makes it the default for every non-`Copy` value and checks it.

### 3. Linear types say why "use at most once" is the right rule, and Wadler already had "many readers or one writer"

**What linear types are like, in their own right.** Philip Wadler's *Linear types can change the world!* (1990) starts: "Values belonging to a linear type must be used exactly once: like the world, they cannot be duplicated or destroyed." The point was to let a functional language update state in place safely. If a value has exactly one reference, you can overwrite or free it without a garbage collector. Then Wadler relaxes his own rule. In my reading of the paper's text (extracted roughly from the PostScript): "Pure linearity is, in fact, a stronger constraint than necessary. It is ok to have more than one reference to a value, so long as the value is being 'read'; only when the value is 'written' … is it necessary to guarantee that a single reference exists." He compares this to the readers-and-writers problem: "permission to write must be unique, but permission to read may be freely duplicated." His construct for it, `let!(x) y = u in v`, gives `u` temporary read-only access to `x`, with two conditions. The read access must be finished before `v` runs. And nothing in `u`'s result may be `x` or part of `x`, "otherwise, read access to x … could be passed outside the scope of u".

**Mapped onto Rust.** Read Wadler's `let!` with Rust in mind and you see three things:
- a temporary shared view of a uniquely owned value: `&T`;
- a guarantee that the shared view ends before the owner is written again: the borrow ends before `&mut` or a move;
- a rule that no part of the shared view may escape the scope that granted it: a lifetime.

Rust changes two things. Its values are **affine**, not linear: you may let a value go unused, and `drop` cleans it up. And the "read scope" is not a syntactic block but whatever region of code the reference is still used in (section 6).

**Evidence, said plainly.** The resemblance is close, but it is a comparison. No Rust designer statement citing Wadler's `let!` turned up. One academic survey (Ferdowsi 2022) states that Rust was "inspired by [Wadler 1990] for its notion of Ownership", but it gives no primary source for that. *(interpretive)*

### 4. Cyclone is where `'a` comes from, in substance and in syntax

**What Cyclone was like.** Cyclone (Grossman, Morrisett, Jim, Hicks, Wang, Cheney, PLDI 2002) was "a type-safe programming language derived from C", built to make dangling-pointer dereferences "a compile-time error" with "no run-time checks". Every pointer type carries the region it points into, so `int*ρ` means "a pointer to an int in region ρ". A stack block labeled `L` is a region `ρL` whose lifetime is the block. If you store `&x` in a variable that outlives `x`'s block, the program is rejected, because you "cannot write down a (well-formed) type for the result". Four features matter for Rust:

- **Region-polymorphic functions.** `char?ρ strcpy<ρ, ρ2>(char?ρ d, const char?ρ2 s);` The function is generic over regions, and using the same name `ρ` on the first parameter and the result says the returned pointer "must point to the same region as the first argument."
- **Outlives subtyping.** Cyclone regions are created and freed last-in-first-out. That LIFO order "induces an 'outlives' relationship", and a pointer into a longer-lived region may be used where a shorter-lived one is expected.
- **Defaults instead of annotations.** Inside a function body, region names are inferred. Prototypes are never inferred, so modules compile separately. If you leave a parameter's region out, it gets "fresh region names that are generalized implicitly". A missing region on a return type defaults to the heap. The paper admits this default "fails for functions like strcpy that return one of their parameters."
- **Unique pointers and borrowing** (Hicks, Morrisett, Grossman, Jim, ISMM 2004). A unique pointer `T*`U` is "guaranteed to be the only (usable) reference". Copying it "is treated as consuming the pointer", and using it after that is rejected. To lend it, `let alias<`s> fft_state *`s a = x;` makes a "borrowed pointer", "a second-class copy of a unique pointer that cannot be deallocated and cannot 'escape'". It gets a fresh region name `` `s `` that exists only in that block, and "throughout the scope of the alias, the original unique pointer x is considered to be consumed." The compiler also inserts these alias declarations automatically around calls.

**Mapped onto Rust, trait by trait.** Read `fn longest<'a>(x: &'a str, y: &'a str) -> &'a str` next to Cyclone's `strcpy<ρ, ρ2>` prototype and it is the same idea:
- a function generic over the lifetime of the memory it is handed;
- with name-equality used to say "the result points where this input points";
- written in signatures and inferred in bodies;
- where a reference type with a longer lifetime can stand in for a shorter one.

Ferdowsi's survey says the same: "lifetimes in Rust are very similar to regions and region annotations in Cyclone, including their syntax, subtyping, and intraprocedural inference". The Book's "a reference must not outlive the data" is Cyclone's rule against dangling pointers, applied to all references.

Rust's three lifetime-elision rules (RFC 141, 2014) do the job Cyclone's defaults did. RFC 141 found they removed 147 of 169 required annotations in the standard library, about 87%. The first rule, that each elided input becomes "a distinct lifetime parameter", is Cyclone's fresh-name default. Rust then avoids Cyclone's documented `strcpy` failure. Instead of defaulting an output to the heap, it ties the output to the single input, or to `self`, and otherwise makes you write it.

Rust's `&x` and `&mut x` together do what Cyclone's `let alias` did. They lend a uniquely owned value under a fresh, scope-limited name, and the owner is frozen until the loan ends. Rust made this the ordinary way of passing anything. In Cyclone it was an opt-in feature of one pointer kind.

**What Rust dropped.** A Cyclone region is a real allocation arena: you can `rnew(r)` into it, and leaving the block frees it. A Rust lifetime allocates nothing and frees nothing. It is purely a static name for "the stretch of code where this borrow must stay valid". Ferdowsi: in Rust "lifetimes are automatically determined by the compiler, and cannot be explicitly set or used (except for generic lifetime parameters)."

**Evidence.** That Cyclone's regions shaped Rust is *documented* (Rust Reference). The specific correspondences above (polymorphism, outlives, elision ≈ defaults, borrow ≈ `let alias`) are *interpretive*: drawn by me and, for the first three, by one survey author. Neither RFC 141 nor any Rust designer text I found cites Cyclone for them.

**One layer further back: Tofte and Talpin.** Cyclone describes itself as "following the seminal work of Tofte and Talpin": regions in an ML compiler, the ML Kit, where "the compiler automatically infers all region annotations". The Cyclone authors' verdict on that is the reason both Cyclone and Rust write lifetimes in signatures: "small changes to a program can have drastic, unintuitive effects on object lifetimes", so "one must understand the analysis and try to control it indirectly". Cyclone answered with annotations in prototypes and inference only inside function bodies. Rust has the same split, and Ferdowsi notes that "Rust, similar to Cyclone, does not infer lifetimes in function signatures." Whether Rust's designers made that choice because of the ML Kit experience is *unknown*. Nothing I found says so.

### 5. Rust's own turn: `&mut` means "unique", and that one rule pays for four things

The rule "one `&mut` or many `&`" was not inherited whole. It was settled inside Rust in 2012. At the time, Rust had garbage-collected `@` boxes, owned `~` boxes, and a "purity" system. Borrow errors about "aliasable, mutable" data were common and hard to understand. Niko Matsakis's post *Imagine never hearing the phrase "aliasable, mutable" again* (2012-11-18) proposed "to change the semantics of `&mut` to say that an `&mut` pointer is guaranteed to be the only way to mutate the thing that it points at." He described himself as least comfortable with exactly that part. Patrick Walton's summary of the new checker (2013-01-21) gives the rule: "Whenever you take a pointer to an object, you may not modify that object as long as that pointer exists, except through that pointer." He motivates it with iterator invalidation. In 2014 Matsakis went further (*Focusing on ownership*): "What the type system really cares about is uniqueness … Mutability becomes a sideshow that is derived from uniqueness." He proposed renaming `&mut` to `&uniq`, which did not happen, and called `&T` "a shared reference". *(documented)*

That rule turned out to serve four purposes at once, and the sources name each one:
- **Memory safety in one thread.** You can't push to a `Vec` while holding `&v[0]`, because the push might reallocate (Walton's iterator-invalidation case).
- **No data races.** The Book defines a data race as two pointers accessing the same data at once, at least one writing, with no synchronization. That is exactly "aliasing plus mutation", which the rule forbids. The 2015 *Fearless Concurrency* post: "the same tools that make Rust safe also help you tackle concurrency", and "Memory safety bugs and concurrency bugs often come down to code accessing data when it shouldn't." Matsakis 2014: the problems "arise when you have both aliasing and mutability."
- **Optimization.** The Rustonomicon (*Aliasing*) shows a function `compute(input: &u32, output: &mut u32)` that can keep `*input` in a register, because "`&mut` isn't allowed to be aliased". In most languages that optimization is unsound without whole-program analysis.
- **A typestate replacement.** Rust's early typestate system, from NIL and Hermes, was removed in Rust 0.4. Walton argued (*Typestate is dead*, 2012-12-26) that uniqueness provides "most of the safety guarantees that typestate enabled". A method that consumes `File<Open>` and returns `File<Closed>` means "you must first surrender your open file."

The escape hatch follows from the same idea. If `&` means "shared", not "read-only", then shared mutation needs a type that checks it some other way. Matsakis: "you can mutate it using some kind of cell type", which is today's `Cell`/`RefCell`/`Mutex`. His 2012 proposal had exactly this split: static checks for borrows of locals, dynamic checks for `@mut` boxes.

### 6. What "the scope of a reference" means has changed twice

The first borrow checker tied a reference's lifetime to the lexical block of the variable holding it. RFC 2094 (*Non-lexical lifetimes*) records the cost: "assigning a reference into a variable means that its lifetime must be as large as the entire scope of that variable". It replaced that with a lifetime defined as a "set of points in the control-flow graph" where the reference may still be used. That is why the Book can now say a reference's scope "continues through the last time that reference is used". NLL became the default for all code in Rust 1.63 (2022). *(documented)*

NLL still rejects one well-known safe pattern, the RFC's "problem case #3". In that pattern, a function looks up a key with `get_mut` and returns the found reference to its caller. Otherwise it inserts a value. The checker thinks the first borrow is still live on the insert path. The next checker, Polonius, changes the definition again: a lifetime becomes "a set of loans", the borrow expressions it may have come from (Matsakis, 2018-04-27). The 2022 NLL announcement calls this "fixing a known limitation of the borrow checker." As of the Rust Project's 2026 goal page, Polonius is a working prototype on nightly. It accepts "NLL problem case 3", it is not stable, and it has one known soundness issue left. *(documented)*

### 7. Putting it back together: what the borrow checker is

Each source above contributes one piece:
- From C++, an owner with a destructor.
- From the linear-types line of work, the idea that an owned value is used at most once and that many readers or one writer is the safe relaxation. The link to Rust is a comparison, not an attested lineage.
- From Cyclone, the machinery that makes a borrow safe. This influence is documented. The trait-by-trait mapping is a comparison. The machinery is: references typed by the region they point into; functions generic over those regions; outlives subtyping; explicit signatures with inference inside bodies; defaults to keep annotations rare; and a borrowed alias of a unique value that cannot escape its scope.

What looks distinctively Rust's own, on these sources, comes down to four points. I did not survey other languages to confirm the uniqueness of any of them.
1. Uniqueness is the default for every value, not an opt-in pointer kind like Cyclone's `` `U ``. So the borrow is the normal way to pass data.
2. `&mut` was redefined as "unique", which made one rule do the work of memory safety, race freedom and alias-based optimization together.
3. A lifetime is purely a static region of code, with no allocation behind it. Rust has refined that region twice: from lexical blocks, to control-flow points (NLL), to sets of loans (Polonius, not yet stable).
4. When the static rule is too strict, there are named runtime-checked escape hatches (`RefCell`, `Mutex`) and an `unsafe` boundary, instead of a garbage-collected fallback. Cyclone kept a GC as its "safety net".

The borrow checker is not a sum of these sources. Rust put them together under one goal: no garbage collector, and safety in both one thread and many.

## What you can now recognize

- **`E0382 borrow of moved value`** is Cyclone's "consumed" unique pointer and a linear value "used" once. It is the compiler enforcing that a moved-from value is dead, which C++ only asks you to assume. It is not a style complaint.
- **`fn longest<'a>(x: &'a str, y: &'a str) -> &'a str`** reads as a region-polymorphic prototype in Cyclone's style: "whatever region the caller's data lives in, the result points into the same one, so it can't outlive the shorter of the two." `'a` names where the data lives. It does not set how long anything lives.
- **Elision** is a set of default annotations, Cyclone-style, with one known-bad default (output defaults to the heap) replaced by an error. When the compiler says "missing lifetime specifier" on a return type, it is declining to guess which input you are returning from. That is exactly the case Cyclone's authors said defaults can't settle.
- **"Cannot borrow as mutable because it is also borrowed as immutable"** is the readers-writers rule. Think "unique" whenever you see `&mut`. The question to ask is "is anything else still able to see this?", not "am I changing it?" That is also why `RefCell` lets you mutate through `&`.
- **"Still borrowed" errors that seem to ignore that you stopped using the reference.** Since NLL, a borrow lasts until last use, not end of block. If the checker still objects to code that returns a `get_mut` result from one branch and inserts in the other, you have probably hit NLL problem case #3. That is a known limitation of today's analysis, which Polonius accepts, not a hidden bug in your logic.

## Supporting detail

**Inspected directly**
- Rust Book chapters 4.1 (ownership), 4.2 (references and borrowing), 10.3 (lifetimes): https://doc.rust-lang.org/book/
- Rust Reference, *Influences*: https://doc.rust-lang.org/reference/influences.html
- RFC 141 (lifetime elision): https://rust-lang.github.io/rfcs/0141-lifetime-elision.html; RFC 2094 (NLL): https://rust-lang.github.io/rfcs/2094-nll.html
- Rustonomicon *Drop Flags* and *Aliasing*: https://doc.rust-lang.org/nomicon/drop-flags.html, https://doc.rust-lang.org/nomicon/aliasing.html
- Rust blog: *Fearless Concurrency* (2015-04-10), https://blog.rust-lang.org/2015/04/10/Fearless-Concurrency/; *NLL by default* (2022-08-05), https://blog.rust-lang.org/2022/08/05/nll-by-default/
- Rust Project goal, Polonius (2026): https://goals.rust-lang.org/2026/polonius.html
- Matsakis blog posts 2012-11-18, 2014-05-13, 2018-04-27 (smallcultfollowing.com/babysteps); Walton posts 2012-12-26 and 2013-01-21 (pcwalton.github.io)
- Full text, extracted locally: Grossman et al., *Region-Based Memory Management in Cyclone*, PLDI 2002, https://www.cs.tufts.edu/comp/150BUGS/cyclone-2002.pdf; Hicks et al., *Experience With Safe Manual Memory-Management in Cyclone*, ISMM 2004, https://www.cs.umd.edu/~mwh/papers/ismm.pdf; Ferdowsi, *The Usability of Advanced Type Systems: Rust as a Case Study*, 2022, https://arxiv.org/abs/2301.02308
- Wadler, *Linear types can change the world!* (1990), https://homepages.inf.ed.ac.uk/wadler/papers/linear/linear.ps. Text pulled out of the PostScript by a rough script with spacing lost; quotes above were re-spaced by me.
- cppreference `std::move`: https://en.cppreference.com/w/cpp/utility/move

**Not inspected:** no Rust code compiled (no toolchain present); the Tofte–Talpin 1997 paper itself (profiled only through Cyclone's related-work section); the rustc source; Graydon Hoare's early design notes; Cyclone's "Pointers with Restricted Aliasing" manual page.

**Connections**

| Shaping source | Trait carried forward, and how it changed | Evidence | Source |
| --- | --- | --- | --- |
| C++ RAII, smart pointers, move semantics | Owner frees at scope end (`drop`). Move changed from a library convention that leaves a "valid but unspecified" object behind into a static invalidation with no destructor for the source | documented (influence); transformation from the cited docs | Rust Reference *Influences*; Book 4.1; cppreference `std::move`; Nomicon *Drop Flags* |
| ML Kit / Cyclone regions | Reference types carry the region they point into; dangling references are a type error | documented | Rust Reference *Influences* |
| Cyclone region polymorphism and outlives subtyping | `<ρ,ρ2>` prototypes and LIFO "outlives" → `<'a>` lifetime parameters and lifetime subtyping; regions lose their allocation role | interpretive (stated as similarity by one survey) | Cyclone PLDI 2002 §2.2–2.3; Ferdowsi 2022 §3 |
| Cyclone default annotations | Fresh name per elided input → elision rule 1; Cyclone's heap default for outputs, which fails for `strcpy`, → Rust's single-input / `self` rules, else an error | interpretive (RFC 141 cites no prior art) | Cyclone PLDI 2002 §2.4; RFC 141 |
| Cyclone signature-explicit, body-inferred regions | Same split in Rust | interpretive | Cyclone PLDI 2002 §2.4; Ferdowsi 2022 footnote 4 |
| Cyclone unique pointers + `let alias` borrowing | Copy consumes → move; scoped alias with fresh region, original consumed meanwhile → `&`/`&mut` borrow with owner frozen; made default and implicit | interpretive (no Rust source names the 2004 paper) | Hicks et al. ISMM 2004 §3, §3.1 |
| Tofte–Talpin / ML Kit full inference (second layer) | Cyclone rejected whole-annotation inference as unpredictable; Rust has the same stance | Cyclone↔TT documented; Rust's reason unknown | Cyclone PLDI 2002 §1, §7; Ferdowsi 2022 |
| Wadler 1990 linear types, `let!` | Use-once → affine use-at-most-once; many readers or one writer and no escape of read access → `&`/`&mut` and lifetimes | interpretive (Ferdowsi asserts inspiration without a primary citation) | Wadler 1990 §4; Ferdowsi 2022 §2.1, §3 |
| Matsakis 2012 "aliasable, mutable" | `&mut` redefined as the only way to mutate its target | documented | smallcultfollowing.com/babysteps/blog/2012/11/18/imagine-never-hearing-the-phrase-aliasable/ |
| Walton 2013 new borrow check | "may not modify … as long as that pointer exists, except through that pointer"; motivated by iterator invalidation | documented | pcwalton.github.io/_posts/2013-01-21-the-new-borrow-check-in-a-nutshell.html |
| Matsakis 2014 "Focusing on ownership" | Uniqueness, not mutability, is the core; `&T` = shared; cells for shared mutation | documented | smallcultfollowing.com/babysteps/blog/2014/05/13/focusing-on-ownership/ |
| Same rule → data-race freedom | Aliasing plus mutation forbidden statically also excludes data races | documented | Book 4.2; *Fearless Concurrency* 2015 |
| Same rule → alias optimization | `&mut` non-aliasing lets the compiler reorder reads | documented | Nomicon *Aliasing* |
| NIL/Hermes typestate | Removed in 0.4; uniqueness and consuming moves recover most of its guarantees | documented (as Walton's argument) | Rust Reference *Influences*; pcwalton 2012-12-26 |
| RFC 2094 NLL | Lifetime = set of control-flow points; default in 1.63 | documented | RFC 2094; Rust blog 2022-08-05 |
| Polonius | Lifetime = set of loans; accepts NLL problem case 3; not yet stable | documented | Matsakis 2018-04-27; goals.rust-lang.org/2026/polonius.html |

**Unresolved branches**
- Whether Rust's designers took the borrow-with-frozen-owner design from Cyclone's 2004 `let alias` specifically, or reached it on their own. Also whether they ever cited Wadler's `let!`. No first-person statement was found for either.
- Ownership types (Clarke, Potter, Noble 1998): Ferdowsi says they are "not directly related to Rust's notion of Ownership" despite the name. Not studied further.
- Graydon Hoare's own account of where `~` unique pointers came from. Searches returned only secondary summaries and one unreliable generated page, which was not used.
- The exact current status of Polonius beyond the 2026 goal page (feature gate, timeline).

**What changes for the current task:** there is no code task here. For learning Rust, the practical changes are in "What you can now recognize": treat `&mut` as "unique", read lifetime parameters as names for where data lives, and recognize NLL problem case #3 when it comes up.
