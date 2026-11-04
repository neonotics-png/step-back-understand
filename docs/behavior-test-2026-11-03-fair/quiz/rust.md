# Rust's Borrow Checker: Comprehension Quiz

Answer each question in 2-4 sentences.

1. At the machine level, `let y = x;` often compiles to the same bitwise copy whether `x`'s type is `Copy` or not. If the bytes copied are the same, what does "moving" a value actually change, and why can a type not be both `Copy` and implement `Drop`?

2. Rust forbids having a shared reference and a mutable reference to the same data at the same time, even in a single-threaded program where no data race is possible. Give a concrete example of what could go wrong without this rule, and explain what bug the rule prevents.

3. Under non-lexical lifetimes (NLL), the code `let r = &v[0]; v.push(1);` compiles if `r` is never used again, even though `r` is still in scope. Why does it compile? And why might the same code be rejected if `r` were instead stored in a struct that implements `Drop`?

4. In `fn longest<'a>(x: &'a str, y: &'a str) -> &'a str`, what does `'a` actually stand for when the function is called with two references that live for different lengths of time? Why does Rust require this annotation in the signature instead of working it out from the function body?

5. A function has the bound `T: 'static`. Can it accept a `String` built at runtime from user input? What does `T: 'static` mean, and how is it different from `&'static T`?

6. `&'a T` is covariant in `T`, but `&'a mut T` is invariant in `T`. Why must `&mut T` be invariant? Sketch the unsound program that covariance would allow.

7. `Cell<T>` and `RefCell<T>` let you mutate data through a shared `&` reference. How does this fit with Rust's "shared XOR mutable" guarantee, and what do you give up compared with ordinary `&mut` borrows?

8. Inside a method, `let a = &mut self.x; let b = &mut self.y;` compiles, but `let a = self.x_mut(); let b = self.y_mut();` (where each getter takes `&mut self` and returns a reference to one field) does not. Why does the borrow checker treat these differently, and what design principle explains the rejection?
