# Rust's Borrow Checker: Ownership, Borrowing and Lifetimes

This guide builds one mental model and then applies it to everything else. Each section has code you can paste into the [Rust Playground](https://play.rust-lang.org) to see the compiler's real messages. If you can predict which snippets compile and which don't, you understand the borrow checker.

---

## 0. The one idea underneath everything

Rust guarantees memory safety without a garbage collector by enforcing two rules at compile time:

1. **Every value has exactly one owner.** When the owner goes away, the value is freed (dropped). No GC and no manual `free`.
2. **At any moment, a value can be either *shared* or *mutated*, never both.** You may have many `&T` (read-only) references, or exactly one `&mut T` (read-write) reference, and **no reference may outlive the value it points to**.

Rule 2 is often summarized as **"aliasing XOR mutation."** The clearest names for the two reference types are:

| Syntax  | Usual name           | More accurate name    | What it promises                                      |
|---------|----------------------|-----------------------|-------------------------------------------------------|
| `&T`    | immutable reference  | **shared** reference  | Others may be reading too, so nobody can write.       |
| `&mut T`| mutable reference    | **unique** reference  | Only you can reach this value right now, so writing is safe. |

The rest of the system exists to enforce these two rules: moves, borrows, lifetimes and every error code covered below.

### Why these rules? The bugs they make impossible

| Bug in C/C++ (or with races, in Java/Go) | Which rule prevents it |
|---|---|
| Use-after-free / dangling pointer | References can't outlive the owner |
| Double free | Only one owner frees |
| Iterator invalidation (mutating a vec while iterating) | Can't mutate while shared |
| Data races between threads | Can't have `&mut` and `&` at the same time (plus `Send`/`Sync`) |
| Memory leaks from forgotten `free` | Owner drops automatically (leaks are still *possible*, e.g. `Rc` cycles, but they don't happen by accident) |

---

## 1. Ownership

### 1.1 Scope and drop

```rust
fn main() {
    {
        let s = String::from("hello"); // s owns a heap buffer
        println!("{s}");
    } // s goes out of scope -> String::drop runs -> heap buffer freed
}
```

Nothing unusual here. It's RAII, like C++ destructors. What's new is how *assignment* behaves.

### 1.2 Move semantics

```rust
let s1 = String::from("hello");
let s2 = s1;          // ownership MOVES from s1 to s2
println!("{s1}");     // error[E0382]: borrow of moved value: `s1`
```

A `String` is a small struct on the stack (pointer, length, capacity) pointing to a heap buffer. `let s2 = s1` copies those three words, so both would point to the same buffer. If both were valid, both would free it at scope end (double free). Rust instead treats `s1` as **moved-from** and statically dead. Only `s2` will drop the buffer.

A move is just a shallow `memcpy` plus a compile-time note that the source is now unusable. No deep copy happens, and nothing extra happens at runtime.

Moves happen on:
- assignment: `let b = a;`
- passing by value to a function: `takes(a);`
- returning from a function
- pushing into a collection, capturing in a `move` closure, pattern-matching by value, etc.

```rust
fn takes(s: String) { println!("{s}"); } // s dropped here

fn gives() -> String { String::from("yours") } // ownership moves out to caller

fn main() {
    let a = String::from("hi");
    takes(a);
    // println!("{a}"); // E0382: a was moved into `takes`

    let b = gives(); // b now owns it
    println!("{b}");
}
```

### 1.3 `Copy` types are the exception

```rust
let x = 5;
let y = x;
println!("{x} {y}"); // fine
```

Types whose bits fully describe them, with no heap resource to free, implement `Copy`: integers, floats, `bool`, `char`, shared references `&T`, and tuples/arrays of `Copy` types. For these, "move" just copies, and the source stays valid. A type can't be both `Copy` and have a custom `Drop`. That makes sense: if copying the bits duplicated ownership of a resource, you'd be back to double-frees.

Note that `&T` is `Copy`, but `&mut T` is **not**. Copying a unique reference would make it non-unique.

### 1.4 `Clone` is the explicit deep copy

```rust
let s1 = String::from("hello");
let s2 = s1.clone(); // allocates a new buffer
println!("{s1} {s2}"); // both valid
```

Rust makes expensive copies visible in the source. When you see `.clone()`, you know an allocation may happen.

### 1.5 Partial moves

```rust
struct Person { name: String, age: u32 }

let p = Person { name: "Ann".into(), age: 30 };
let n = p.name;        // move just the `name` field out
println!("{}", p.age); // OK: `age` wasn't moved (and it's Copy anyway)
// println!("{}", p.name); // E0382
// let q = p;              // E0382: p is partially moved
```

The checker tracks ownership **per field**, which matters later for borrowing too.

### 1.6 You can't move out from behind a reference

```rust
fn first(v: &Vec<String>) -> String {
    v[0] // error[E0507]: cannot move out of index of `Vec<String>`
}
```

You only *borrowed* the vec, so you can't take its contents. Options:
- `v[0].clone()`: copy it
- return `&String` / `&str` instead: `fn first(v: &Vec<String>) -> &str { &v[0] }`
- if you own it mutably: `std::mem::take(&mut v[0])` (leaves `String::new()` behind), `v.remove(0)`, `v.swap_remove(0)`
- if you own the whole vec: `v.into_iter().next()`

---

## 2. Borrowing

Moving values in and out of every function would be tedious, so you **borrow** them instead: you get a reference, and ownership stays put.

```rust
fn len(s: &String) -> usize { s.len() } // borrows, doesn't own

fn shout(s: &mut String) { s.push('!'); } // borrows uniquely, can mutate

fn main() {
    let mut s = String::from("hi");
    let n = len(&s);   // shared borrow, ends after the call
    shout(&mut s);     // unique borrow, ends after the call
    println!("{s} {n}");
}
```

### 2.1 The borrowing rules, precisely

While a borrow is **live** (see 2.3):

| Existing borrow | Can you read via owner? | Write via owner? | Move owner? | Take another `&`? | Take another `&mut`? |
|---|---|---|---|---|---|
| some `&T` | yes | **no** (E0506) | **no** (E0505) | yes | **no** (E0502) |
| one `&mut T` | **no** (E0502/E0503) | **no** (E0506) | **no** (E0505) | **no** (E0502) | **no** (E0499) |

### 2.2 The classic example: iterator invalidation

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];   // shared borrow of v
v.push(4);           // error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
println!("{first}");
```

This isn't pedantry. `push` may reallocate the buffer, and `first` would then point into freed memory. In C++ this is undefined behavior that compiles silently. Rust rejects it because a shared borrow (`first`) is still live while a unique one (`&mut v` for `push`) is requested.

The same applies to:

```rust
for x in &v {
    if *x == 2 { v.push(4); } // E0502
}
```

### 2.3 Borrows last until their *last use*, not the end of the scope (NLL)

Since Rust 2018, the checker uses **Non-Lexical Lifetimes (NLL)**: a borrow is live from where it's created to where it's **last used**, not to the closing `}`.

```rust
let mut s = String::from("hi");
let r = &s;
println!("{r}");   // last use of r -> borrow ends here
let m = &mut s;    // OK!
m.push('!');
println!("{s}");   // OK: m's last use was the line above
```

Swap two lines and it fails:

```rust
let mut s = String::from("hi");
let r = &s;
let m = &mut s;    // E0502: r is still used below
println!("{r}");
```

**Mental model:** draw a line from each borrow's creation to its last use. If a conflicting access falls on that line, you get an error.

### 2.4 Reborrowing: why passing `&mut` around "just works"

`&mut T` isn't `Copy`, yet this compiles:

```rust
fn bump(x: &mut i32) { *x += 1; }

let mut n = 0;
let r = &mut n;
bump(r);  // implicitly bump(&mut *r): a short "reborrow" of r
bump(r);  // fine again, r was never moved
```

The compiler inserts `&mut *r`, a new unique borrow *derived from* `r` that lasts only for the call. While a reborrow is live, the original `r` is frozen. When it ends, `r` is usable again. This is how one `&mut` can be lent out repeatedly.

(Implicit reborrowing happens when the target type is known to be `&mut _`. In generic contexts, such as `fn g<T>(t: T)`, `r` would be *moved* instead, and you'd write `g(&mut *r)` yourself.)

### 2.5 Two-phase borrows

```rust
let mut v = vec![1, 2];
v.push(v.len()); // compiles
```

Strictly, `v.push(...)` takes `&mut v` *before* evaluating the argument `v.len()` (which needs `&v`). The compiler special-cases this with a "two-phase borrow": the `&mut` is *reserved* first and only *activated* when the call actually happens. It's a usability exception. Don't build a mental model around it.

### 2.6 Disjoint borrows: fields yes, through methods no

The checker understands struct fields as separate places:

```rust
struct Game { players: Vec<String>, scores: Vec<u32> }

let mut g = Game { players: vec![], scores: vec![] };
let p = &mut g.players;
let s = &mut g.scores;   // OK: different fields
p.push("a".into());
s.push(1);
```

But a method signature is a *contract* covering all of `self`:

```rust
impl Game {
    fn players_mut(&mut self) -> &mut Vec<String> { &mut self.players }
}
let p = g.players_mut();   // borrows ALL of g uniquely
let s = &mut g.scores;     // E0499: g already mutably borrowed
p.push("a".into());
```

The borrow checker reasons **locally**: it looks at function *signatures*, never at function *bodies*, so `players_mut` is treated as possibly touching everything. Fixes: access fields directly, return both at once (`fn split(&mut self) -> (&mut Vec<String>, &mut Vec<u32>)`), or restructure.

Slices have the same problem with indices (`&mut v[0]` and `&mut v[1]` conflict, because the checker doesn't do arithmetic). The standard library provides safe splitters:

```rust
let mut v = [1, 2, 3, 4];
let (left, right) = v.split_at_mut(2);
left[0] += right[0]; // OK
```

Also see `iter_mut()`, `chunks_mut()`, and `get_disjoint_mut` (stable since Rust 1.86) for slices and `HashMap`.

### 2.7 Closures borrow too

A closure captures what it uses, in the weakest mode that works (by `&`, by `&mut`, or by value). Since Rust 2021 it captures *individual fields*, not whole structs.

```rust
let mut count = 0;
let mut inc = || count += 1; // captures &mut count
// println!("{count}");      // E0502: inc holds &mut count and is used below
inc();
inc();
println!("{count}");         // OK: inc is dead now
```

`move ||` forces capture by value. You need it for threads and for closures returned from functions, because the closure must not hold borrows of locals that are about to die.

---

## 3. Lifetimes

### 3.1 What a lifetime *is*

A **lifetime** is the region of the program during which a reference must remain valid. Every reference has one. Most are inferred.

Three key facts:

1. **Lifetimes are a compile-time concept only.** Nothing exists at runtime.
2. **Annotations never change how long anything lives.** `'a` doesn't extend or shorten anything. It *describes relationships* between references ("the output borrows from this input") so the compiler can check them.
3. **A lifetime is not the same as a variable's scope.** A value lives until it's dropped. A *borrow* of it may be much shorter. `'a` names a borrow region, which must fit inside the value's life.

The simplest lifetime error:

```rust
let r;
{
    let x = 5;
    r = &x;          // error[E0597]: `x` does not live long enough
}                    // x dropped here, while still borrowed
println!("{r}");     // borrow later used here
```

### 3.2 Why functions need annotations

Inside a function body, the compiler infers everything. At a **function boundary**, it reasons only from the signature (the same locality principle as in 2.6). So when a function returns a reference, the signature must say *which input it borrows from*.

```rust
fn longest(x: &str, y: &str) -> &str {  // error[E0106]: missing lifetime specifier
    if x.len() > y.len() { x } else { y }
}
```

The compiler tells you: *"this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`."* Fix it with a lifetime parameter:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

Read it as: *"for some region `'a` during which both `x` and `y` are valid, the result is valid for `'a` as well."* At each call site the compiler picks `'a`. In practice that's the **overlap** of the two borrows, the shorter one.

```rust
let s1 = String::from("long string");
let result;
{
    let s2 = String::from("xyz");
    result = longest(&s1, &s2);  // E0597: `s2` does not live long enough
}
println!("{result}");            // result might point at s2, which is gone
```

You can see that `s1` is longer, but the compiler doesn't evaluate `if`. The signature says "result may borrow from either", so it must assume the worst.

If the output only ever borrows from one input, say so, and the other is freed from the constraint:

```rust
fn first_of<'a>(x: &'a str, _y: &str) -> &'a str { x }
```

### 3.3 Lifetime elision: why you rarely write them

The compiler fills in lifetimes for common patterns using three rules:

1. Each elided lifetime in the **inputs** gets its own distinct parameter.
   `fn f(x: &str, y: &str)` becomes `fn f<'a, 'b>(x: &'a str, y: &'b str)`
2. If there's **exactly one** input lifetime, it's assigned to every elided output lifetime.
   `fn first_word(s: &str) -> &str` becomes `fn first_word<'a>(s: &'a str) -> &'a str`
3. If there's a `&self` or `&mut self`, **self's** lifetime goes to every elided output lifetime.
   `fn name(&self, other: &str) -> &str`: the output borrows from `self`

If the rules don't determine the output lifetime (like `longest`), you get E0106 and must write it yourself.

Rule 3 has a consequence that catches many people. See 3.5.

### 3.4 You can't return a reference to something you created

```rust
fn make() -> &String {        // E0106
    let s = String::from("x");
    &s                        // even with a lifetime: E0515, returns a reference to local data
}
```

No lifetime annotation can fix this. `s` dies when the function returns. **Return the owned value** (`-> String`). Lifetimes only let you return references *derived from inputs* (or `'static` data).

### 3.5 Structs that hold references

A struct holding a reference needs a lifetime parameter. It means *"an instance of this struct can't outlive the data it borrows."*

```rust
struct Parser<'a> {
    input: &'a str,
    pos: usize,
}

impl<'a> Parser<'a> {
    // Returns slices of the ORIGINAL input, tied to 'a, not to &mut self.
    fn next_word(&mut self) -> Option<&'a str> {
        let input: &'a str = self.input;           // copy the &'a str out
        let rest = &input[self.pos..];
        let trimmed = rest.trim_start();
        if trimmed.is_empty() { return None; }
        let start = self.pos + (rest.len() - trimmed.len());
        let end = trimmed
            .find(char::is_whitespace)
            .map_or(input.len(), |i| start + i);
        self.pos = end;
        Some(&input[start..end])
    }
}

fn main() {
    let text = String::from("hello brave new world");
    let mut p = Parser { input: &text, pos: 0 };
    let a = p.next_word().unwrap();
    let b = p.next_word().unwrap(); // OK: `a` borrows `text`, not `p`
    println!("{a} {b}");
}
```

Now change one thing in the signature and let elision rule 3 kick in:

```rust
fn next_word(&mut self) -> Option<&str>   // elided: output tied to &mut self
```

Now `a` is a borrow *of `p`* (uniquely!), so the second `p.next_word()` fails with **E0499** while `a` is alive. The body is identical, but the signature tells callers something different, and callers are checked only against signatures.

**Takeaway:** when a method returns data that really comes from a struct's borrowed field, write the struct's lifetime (`'a`) explicitly in the return type. Elision will tie it to `self`, which is usually much more restrictive than necessary.

A second lesson: structs holding references are a sharp tool. For long-lived data structures, **prefer owning** (`String`, `Vec<T>`) over borrowing (`&str`, `&[T]`). Use borrowed structs for short-lived views such as parsers, iterators and "context" objects passed into a function.

### 3.6 `'static`

`'static` is the lifetime of the whole program run.

- `&'static str`: string literals live in the binary, so `let s: &'static str = "hi";`.
- **`T: 'static` does *not* mean "T lives forever."** It means "T contains no borrows shorter than `'static`", that is, T could be kept around indefinitely *if you wanted*. `String`, `Vec<u8>` and `i32` are all `T: 'static`, even though they get dropped normally.

This is why `std::thread::spawn` requires `F: 'static`. The thread might outlive the current function, so the closure can't borrow locals:

```rust
let v = vec![1, 2, 3];
// std::thread::spawn(|| println!("{v:?}")); // E0373: closure may outlive the current function
std::thread::spawn(move || println!("{v:?}")).join().unwrap(); // OK: v moved in

// Or, to borrow: scoped threads guarantee joining before the scope ends.
let w = vec![4, 5, 6];
std::thread::scope(|s| {
    s.spawn(|| println!("{w:?}")); // OK to borrow w
});
```

If the compiler suggests adding `'static` to fix an error, think twice. It's usually right for threads and `Box<dyn Trait>`, and usually wrong when you're actually trying to return a borrow.

### 3.7 Lifetime bounds and outlives relationships

- `'a: 'b` reads as "`'a` outlives `'b`". Data valid for `'a` is valid for at least `'b`.
- `T: 'a` reads as "every reference inside `T` lives at least `'a`".
- `&'a T` implicitly requires `T: 'a`. You can't have a reference that outlives the thing it points to.

You'll mostly see these in generic code. For example, `struct Wrapper<'a, T: 'a> { r: &'a T }` (the bound is inferred nowadays).

### 3.8 Variance (advanced, but explains some odd errors)

Can a `&'long T` be used where a `&'short T` is expected? Yes. Shared references are **covariant**: a longer-lived borrow can be shortened. That's why `longest(&s1, &s2)` works with differently-scoped strings.

But `&'a mut T` is **invariant in `T`**. You can't treat `&mut &'long str` as `&mut &'short str`, or vice versa. If you could, you could write a short-lived `&str` through it into a slot the caller believes holds a long-lived one, and that creates a dangling reference. When you see errors like *"lifetime may not live long enough"* involving `&mut` of something containing references, invariance is usually the cause. The fix is often to use separate lifetime parameters (`&'a mut Thing<'b>` instead of `&'a mut Thing<'a>`).

The infamous antipattern is `&'a mut Thing<'a>`, which means "borrow `Thing` uniquely for *its entire remaining life*". After one call, you can never use it again.

---

## 4. Reading borrow-checker errors

Each error has a code, and `rustc --explain E0502` prints a full explanation. The error messages usually point to three locations: where the borrow **starts**, where the conflicting access is, and where the borrow is **later used**. That third location is the key. It tells you *why* the borrow is still live.

| Code | Message (paraphrased) | Meaning | Typical fix |
|---|---|---|---|
| E0382 | use of moved value | You used a variable after moving it | Borrow instead (`&x`), `.clone()`, or restructure so the move is last |
| E0499 | cannot borrow as mutable more than once | Two live `&mut` to the same place | Shorten one borrow, split fields, `split_at_mut` |
| E0502 | cannot borrow as mutable because also borrowed as immutable (or vice versa) | Shared and unique overlap | Finish using the `&` first; copy the value out (`let x = v[0];` for `Copy`); collect then mutate |
| E0505 | cannot move out because it is borrowed | Moving a value while a reference to it is live | End the borrow before the move, or clone |
| E0506 | cannot assign because it is borrowed | Writing while a reference is live | Same |
| E0507 | cannot move out of ... behind a reference | Taking ownership through `&`/`&mut` | `.clone()`, `mem::take`, `mem::replace`, `Option::take`, return a reference |
| E0515 | cannot return reference to local | Returning `&` to something that dies | Return the owned value |
| E0597 | does not live long enough | Owner dies while borrow still used | Move the owner to an outer scope, or own the data |
| E0716 | temporary value dropped while borrowed | Borrowed a temporary (e.g. `let s = &String::from("x").to_uppercase()[..];`) | Bind the temporary to a `let` first |
| E0106 | missing lifetime specifier | Signature ambiguous about what output borrows | Add `<'a>` and annotate the relationship |
| E0373 | closure may outlive the current function | Closure borrows locals but needs `'static` | `move` closure, or `thread::scope` |

---

## 5. Toolkit: what to do when you're fighting the checker

From least to most heavyweight:

1. **Reorder or shorten borrows.** Finish reading before you write. Usually the whole fix.
   ```rust
   let len = v.len(); // copy out what you need
   v.push(len);
   ```
2. **Copy out small values** instead of holding references: `let x = v[i];` for `Copy` types.
3. **Use indices instead of references** into collections (`usize` doesn't borrow). This is very common for graphs, arenas and game entities.
4. **Two passes:** collect what to change, then mutate.
   ```rust
   let to_remove: Vec<usize> = v.iter().enumerate()
       .filter(|(_, x)| **x < 0).map(|(i, _)| i).collect();
   // ...then mutate v. Or just: v.retain(|x| *x >= 0);
   ```
5. **Take ownership temporarily** with `std::mem::take`, `std::mem::replace`, `Option::take`. These let you move out of a `&mut` by leaving something valid behind.
   ```rust
   let old = std::mem::take(&mut self.buffer); // self.buffer is now empty
   ```
6. **Use APIs that encode the pattern:** `HashMap::entry`, `Vec::retain`, `split_at_mut`, `iter_mut`, `swap`, `get_disjoint_mut`.
7. **Own instead of borrow** in structs: `String` over `&str`.
8. **`.clone()`.** Perfectly fine when data is small or not on a hot path. Cloning to get unstuck while learning is legitimate. Optimize later if a profiler says so.
9. **Shared ownership:** `Rc<T>` (single-thread) / `Arc<T>` (multi-thread). Multiple owners, freed when the last one drops.
10. **Interior mutability:** `Cell<T>`, `RefCell<T>` (single-thread), `Mutex<T>`, `RwLock<T>`, atomics (multi-thread). These move the aliasing-XOR-mutation check from compile time to **runtime**. `RefCell::borrow_mut` *panics* if a borrow is already active, and `Mutex` blocks. `Rc<RefCell<T>>` and `Arc<Mutex<T>>` are the standard "shared mutable state" idioms.
11. **`unsafe`.** Only for building safe abstractions (like `split_at_mut` itself) when you can prove the invariants by hand. Almost never the right tool for an application-level borrow error.

**Design-level advice:** if you keep fighting the checker in one area, the data structure is usually telling you something. Doubly-linked lists, graphs with back-pointers, and "everything points to everything" object graphs fight single ownership. Arenas plus indices (or crates like `slotmap`/`petgraph`) are the idiomatic answer.

---

## 6. Known limitations: when the checker is wrong

The borrow checker is *sound* (it rejects all unsafe programs) but *incomplete* (it rejects some safe ones). The most famous case is conditionally returning a borrow:

```rust
use std::collections::HashMap;

fn get_or_insert(map: &mut HashMap<u32, String>, k: u32) -> &String {
    if let Some(v) = map.get(&k) {
        return v;                       // borrow returned on this path...
    }
    map.insert(k, String::new());       // E0502 ...but the checker thinks it's still live here
    map.get(&k).unwrap()
}
```

This is safe: if we reach `insert`, the first borrow wasn't returned. The current NLL checker can't see that, because the returned borrow must last for the caller's lifetime, and NLL extends it across the whole function. This is "NLL problem case #3." **Polonius**, the next-generation borrow checker the Rust team has been developing, accepts it, but it isn't the default on stable at the time of writing. Workarounds:

```rust
fn get_or_insert(map: &mut HashMap<u32, String>, k: u32) -> &String {
    map.entry(k).or_default()          // idiomatic
}
// or check first, without holding a borrow:
// if !map.contains_key(&k) { map.insert(k, String::new()); }
// &map[&k]
```

Other known friction points: borrowing through method calls borrows all of `self` (see 2.6), and self-referential structs (a struct holding a reference into itself) aren't expressible in safe Rust without crates like `ouroboros` or `Pin`-based designs. Prefer storing indices or offsets.

When you're sure your code is safe and the checker disagrees, the fix is almost always a small restructure, not `unsafe`.

---

## 7. Test yourself: does it compile?

Predict each one before reading the answer.

**Q1**
```rust
let s = String::from("a");
let t = s;
println!("{s}");
```

**Q2**
```rust
let x = 5;
let y = x;
println!("{x} {y}");
```

**Q3**
```rust
let mut s = String::new();
let r1 = &mut s;
r1.push('a');
let r2 = &mut s;
r2.push('b');
println!("{s}");
```

**Q4**
```rust
let mut v = vec![1, 2, 3];
let first = &v[0];
v.push(4);
```

**Q5**
```rust
let mut v = vec![1, 2, 3];
let first = &v[0];
println!("{first}");
v.push(4);
```

**Q6**
```rust
fn pick(v: &Vec<String>) -> String { v[0] }
```

**Q7**
```rust
fn pick(v: &Vec<String>) -> &String { &v[0] }
```

**Q8**
```rust
let r: &str;
{
    let s = String::from("hi");
    r = &s;
}
println!("{r}");
```

**Q9**
```rust
let r: &str;
{
    let s = "hi";
    r = s;
}
println!("{r}");
```

<details>
<summary>Answers</summary>

1. **No.** E0382: `s` was moved into `t`.
2. **Yes.** `i32` is `Copy`.
3. **Yes.** NLL: `r1`'s last use is before `r2` is created.
4. **Yes, surprisingly.** `first` is never used after `push`, so its borrow already ended. Add `println!("{first}")` after the push and you get E0502.
5. **Yes.** Same reasoning: the borrow ends at the `println!`.
6. **No.** E0507: can't move a `String` out of a borrowed vec. Use `v[0].clone()`.
7. **Yes.** Elision rule 2: output borrows from `v`.
8. **No.** E0597: `s` (the `String`) is dropped at `}` while `r` still borrows it.
9. **Yes.** `"hi"` is a `&'static str`. The variable `s` dies, but the data it points to lives in the binary forever. This shows that lifetimes track the *data borrowed*, not the variable holding the reference.

</details>

---

## 8. One-page summary

- **Ownership:** one owner per value; owner leaving scope drops it. Assigning or passing by value **moves** (source becomes unusable), except for `Copy` types. `.clone()` is an explicit deep copy.
- **Borrowing:** `&T` = shared, read-only, many allowed. `&mut T` = unique, read-write, exactly one. Never both at once. A borrow lasts until its **last use** (NLL).
- **Lifetimes:** compile-time names for "how long this borrow must stay valid." They *describe* relationships and don't change behavior. You write them only at boundaries (function signatures, structs) when elision can't infer which input an output borrows from.
- **Locality:** the checker reasons per function using signatures only. Signatures are contracts. A sloppy signature (e.g. output tied to `&mut self` when it really comes from `'a`) over-constrains callers.
- **When stuck:** reorder, copy out, use indices, `mem::take`, `entry`, own instead of borrow, `clone`, and only then `Rc`/`RefCell`/`Arc`/`Mutex`. If you keep fighting it, rethink the data layout.

---

## 9. Where to go next

- *The Rust Programming Language*, ch. 4 (Ownership) and ch. 10.3 (Lifetimes): https://doc.rust-lang.org/book/
- Brown University's interactive edition of the Book, which adds ownership/permission diagrams and quizzes and is excellent for building intuition: https://rust-book.cs.brown.edu/
- "Common Rust Lifetime Misconceptions" by pretzelhammer, which corrects the misconceptions most people pick up (e.g. about `T: 'static`): https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md
- *Learning Rust With Entirely Too Many Linked Lists*, which shows exactly where ownership hurts and why: https://rust-unofficial.github.io/too-many-lists/
- *The Rustonomicon*, on variance, subtyping and what `unsafe` code must uphold: https://doc.rust-lang.org/nomicon/
- `rustc --explain EXXXX` for any error code, and the Playground to test small hypotheses fast.

The fastest way to build intuition is to write small snippets that you *expect* to fail, predict the error code and the three locations the compiler will point at, then check.
