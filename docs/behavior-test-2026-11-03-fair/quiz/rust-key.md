# Rust's Borrow Checker: Answer Key

## 1. Move vs Copy, and why Copy excludes Drop

**Model answer:** A move and a copy can both be a plain bitwise copy at runtime. The difference is static: after a move, the compiler treats the source as uninitialized and rejects any later use of it, so the value still has exactly one owner. A `Copy` type leaves the source usable, so there are now two independent values. A type that implements `Drop` owns some resource beyond its own bytes (heap memory, a file handle, and so on). Duplicating its bits would give two owners of that resource, and the destructor would run twice (for example, a double free). So `Copy` and `Drop` cannot go together.

**Key points for full credit:**
- A move is a compile-time ownership transfer: the source binding becomes unusable. The runtime bytes need not differ from a copy.
- `Drop` types manage a resource that must be released exactly once. A bitwise duplicate would create two owners, so the resource would be released twice (double free).

**Source:** https://doc.rust-lang.org/std/marker/trait.Copy.html

## 2. Why shared XOR mutable matters single-threaded

**Model answer:** Holding a reference to an element of a `Vec` while pushing to that `Vec` is the classic case. The push can reallocate the buffer, leaving the reference (or an iterator) pointing into freed memory. That is a use-after-free with no threads involved. Similarly, overwriting an enum through one reference can change its variant while another reference still points at the old variant's data. The rule makes these invalidations impossible to express: while any shared borrow is live, no one can mutate (and so restructure or free) what it points to.

**Key points for full credit:**
- A concrete invalidation example: Vec reallocation, iterator invalidation, or an enum variant change that leaves a dangling reference.
- The danger is memory unsafety (use-after-free or dangling pointers) and broken invariants, not only data races. The rule rules these out statically.

**Source:** https://manishearth.github.io/blog/2015/05/17/the-problem-with-shared-mutability/ (also https://doc.rust-lang.org/std/cell/index.html for the rule itself)

## 3. NLL, liveness, and Drop

**Model answer:** Under NLL, a borrow lasts only as long as the reference is live. A reference is live at a point where its value may still be used later in the control-flow graph, not for the whole lexical scope. If `r` is never used after `v.push(1)`, the borrow is dead there, so the mutation does not conflict. If `r` is held in a type with a `Drop` impl, the destructor running at the end of the scope counts as a use that might read the reference. That keeps the borrow live until the drop, so the `push` now overlaps a live shared borrow and is rejected. Plain references (and types whose lifetimes are marked may-dangle) do not have this effect.

**Key points for full credit:**
- NLL lifetimes are sets of points where the reference may still be used (liveness), not lexical scopes.
- A `Drop` impl makes the end-of-scope drop a use of the borrowed data ("drop-live"), which extends the borrow to the point of drop.

**Source:** https://rust-lang.github.io/rfcs/2094-nll.html

## 4. What `'a` means in `longest`, and why signatures need it

**Model answer:** `'a` is a generic parameter that the compiler fills in at each call site. It gets a region that both arguments are valid for, which is effectively the shorter of the two (their overlap), and the returned reference is only valid within that region. The annotation does not make anything live longer. It states a relationship: the output borrows from the inputs and must not outlive either. Rust puts this in the signature because borrow checking is done one function at a time. Callers rely on the signature as a contract, so the body can change without breaking callers, and checking stays local and tractable.

**Key points for full credit:**
- `'a` is instantiated per call to a region that fits both inputs (the shorter one, or their overlap). Annotations describe relationships and do not change how long anything lives.
- Signatures are the contract: checking is intraprocedural, so callers reason only from the signature, not the body.

**Source:** https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html

## 5. `T: 'static` vs `&'static T`

**Model answer:** Yes, it can. A `String` is fully owned and contains no borrowed references, so it satisfies `T: 'static`. The bound means "`T` contains no non-`'static` borrows, so a value of `T` could be kept alive as long as needed." It does not mean the value lives for the whole program, was made at compile time, or is immutable. A `T: 'static` value can still be created at runtime, mutated, and dropped early. `&'static T` is a reference whose referent must actually stay valid for the rest of the program, such as a string literal, a `static`, or leaked memory.

**Key points for full credit:**
- Owned types with no borrowed data (such as a runtime `String`) satisfy `T: 'static`. The bound is about what the type may borrow, not how long the value lives.
- `&'static T` is a reference to data that must stay valid for the program's remaining duration. That is a much stronger and different requirement.

**Source:** https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md

## 6. Why `&mut T` is invariant

**Model answer:** A `&mut T` lets you write a `T` into the referent. If it were covariant, a `&mut &'static str` could be used as a `&mut &'short str`, and you could store a short-lived reference into a slot whose declared type is `&'static str`. After the short-lived data is dropped, the original variable still "is" a `&'static str` but dangles. Example: `fn assign<T>(r: &mut T, v: T) { *r = v }`, then `let mut s: &'static str = "hi"; { let w = String::new(); assign(&mut s, &w); } println!("{s}");`. Shared references cannot be written through, so shrinking their lifetime is safe and they can be covariant.

**Key points for full credit:**
- Mutability makes the type flow both ways (read and write). Covariance would let a shorter-lived value be written into a longer-lived slot.
- A concrete unsound example: writing a short-lived reference through `&mut &'static str`, which gives a use-after-free. `&T` is read-only, so covariance is safe for it.

**Source:** https://doc.rust-lang.org/nomicon/subtyping.html

## 7. Interior mutability and the shared XOR mutable rule

**Model answer:** Interior mutability does not break the guarantee. It moves where the guarantee is enforced. `UnsafeCell` (the only legal primitive for mutation through `&`) tells the compiler not to assume shared data is immutable. `Cell` keeps that safe by never handing out references to its contents: it only copies or moves values in and out. `RefCell` tracks borrows at runtime and panics on a conflicting `borrow_mut`. You give up compile-time proof: conflicts become runtime panics, and there is a small runtime cost. These types are also not `Sync`, so they are single-threaded only. Thread-safe equivalents are `Mutex`, `RwLock`, and atomics.

**Key points for full credit:**
- The exclusivity rule still holds, but it is enforced differently: `Cell` never hands out inner references, and `RefCell` checks borrows at runtime. `UnsafeCell` is the underlying primitive.
- The cost: violations become runtime panics instead of compile errors, plus runtime overhead. The types are not `Sync`.

**Source:** https://doc.rust-lang.org/std/cell/index.html

## 8. Disjoint field borrows vs borrows through methods

**Model answer:** Direct field access names the exact place being borrowed (`self.x` and `self.y`). The borrow checker tracks struct fields separately, so it sees the two `&mut` borrows as disjoint. A method taking `&mut self` borrows the whole struct for as long as its returned reference lives, because the checker only sees the signature, not which fields the body touches. Two such calls are two overlapping exclusive borrows of `self`. This is deliberate: Rust checks each function against its signature only, so a method's body can change without silently breaking callers. Ways around it include splitting borrows yourself (`let Self { x, y, .. } = self;`), a method that returns both references, or `split_at_mut`-style APIs.

**Key points for full credit:**
- The borrow checker understands disjoint struct fields when they are named directly. A `&mut self` call borrows all of `self`.
- Checking is per-function and based on signatures (signatures are an encapsulation boundary), so the compiler will not look inside method bodies to prove disjointness.

**Source:** https://smallcultfollowing.com/babysteps/blog/2021/11/05/view-types/ (also https://doc.rust-lang.org/nomicon/borrow-splitting.html)
