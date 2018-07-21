# Lifetimes: Basic

What's the difference between the lifetimes of these two functions?

```rust
fn wear_pants_like<'a, 'b>(legs: &'a mut Legs) -> &'b Pants { … }
fn or_like_this   <'c, 'd>(legs: &'d mut Legs) -> &'c Pants { … }
```

<details>
They are the same; the labels can be renamed with no difference in semantics.
</details>

---

Give each lifetime a unique name:

```rust
fn foo(a: &i32, b: &i32) -> &i32 { … }
```
<details>

```rust
fn foo<'a, 'b, 'r>(a: &'a i32, b: &'b i32) -> &'r i32 { … }
```

</details>

```rust
use std::sync::RwLockWriteGuard;
// struct RwLockWriteGuard<'a, T: ?Sized + 'a> { … }

fn bar(a: &Thing, lock: &mut RwLockWriteGuard<Thing>) -> &Thing { … }
```
<details>

```rust
fn bar<'a, 'l, 'g, 'r>(a: &'a Thing, lock: &'l mut RwLockWriteGuard<'g, Thing>) -> &'r Thing { … }
```

</details>

---

List all the possible combinations of explicit lifetimes for `baz`
(using just simple `<'foo>` declarations),
and give an example function body for each.

```rust
struct Unit;
fn baz(bit: bool, a: &Unit, b: &Unit) -> &Unit { … }
```

<details>

```rust
fn baz<'c>(bit: bool, a: &'c Unit, b: &'c Unit) -> &'c Unit {
    if bit {
        a
    } else {
        b
    }
}

fn baz<'a, 'b>(bit: bool, a: &'a Unit, b: &'b Unit) -> &'a Unit {
    a
}

fn baz<'a, 'b>(bit: bool, a: &'a Unit, b: &'b Unit) -> &'b Unit {
    b
}
```

</details>

---

List all meaningful lifetimes for each method in this `impl`:

```rust
impl MyStruct {
    fn thing() -> &Self { … }
    fn widget(&self) -> &MyWidget { … }
}
```

<details>

```rust
fn thing() -> &'static Self;
fn widget<'a>(&'a self) -> &'a MyWidget;
fn widget<'a>(&'a self) -> &'static MyWidget;
```

</details>

---

What is the earliest possible point that `'a` can end in `fn hello<'a>(_: &'a ())`?
<details>
Immediately after the function exits; it can not end during that function.
</details>


When can `'a` start?

<details>
`'a` must be alive for there to be anything to pass to `hello`, so its lifetime begins somewhere between the start of the program and the function call.
</details>

---

Provide an implementation for these functions that requires the indicated lifetime.

```rust
struct MyStruct<'s> {
    data: Vec<&'s i32>,
}
impl<'s> MyStruct<'s> {
    fn my_func_s(&mut self, a: &'s i32) { … }
    fn my_func_a<'a>(&mut self, a: &'a i32) { … }
}
```

<details>

```rust
impl<'s> MyStruct<'s> {
    fn my_func_s(&mut self, a: &'s i32) {
        self.data.push(a);
    }
    fn my_func_a<'a>(&mut self, a: &'a i32) {
        if Some(a) == self.data.last() {
            self.data.pop();
        }
    }
}
```

</details>
















# Lifetimes: Triple Cray


Suppose somebody turned off the borrow checker, and that we have a function:

```rust
fn mystery_function(vec: &mut Vec<&i32>, e: &i32) {
    vec.push(e);
}
```

One of these usages of `mystery_function` is gonna give you a bad time. Which? Why? What do we need to express to avoid it?

```rust
{
    let vec1 = &mut vec![];
    {
        let four = &4;
        mystery_function(vec1, four);
    }
}
{
    let five = &5;
    {
        let vec2 = &mut vec![];
        mystery_function(vec2, five);
    }
}
```

<details>

The first, because `four` goes out of scope while `vec1` still has a reference.
The references put into the `Vec` need to outlive the vector.
This can be done via
```rust
    fn mystery_function<'e, 'v: 'e>(vec: &'v mut Vec<&'e i32>, e: &'e i32) { … }
```
</details>

---

What does `'a: 'b` mean? It's a bit cryptic; what's the reasoning for this syntax?
<details>

`'a` outlives `'b`.
In general `?: ??` means that the thing on the left has *more possible states* than the one on the right.
In `T: Clone`, `T` may very well implement a hundred other traits besides `Clone`.
In the case of lifetimes, `'a` covers more lines of code than `'b`.

</details>

---

What does the `'static` in `Box<Debug + 'static>` mean? Why would it be necessary?
<details>

It means that anything that is put into the box must not have a reference that whose lifetime ends before the program.

...

<!--
... uh.

It is necessary because the contents of the `Box` may hold a reference, and the compiler needs a guarantee that the box will not outlive any such references.
It is necessary because the compiler does not track the lifetime of the box the way it tracks references.
It is necessary because a Box is basically the same thing as a reference? Just that it owns its value...
A Box is like an owning reference. 

-->
</details>

---

When trying to compile code like the following,

```rust
trait Treaty {
    fn new(negotiators: &[u32]) -> Self where Self: Sized;
    fn ratify(&self) {}
}

struct MyTreaty;
impl Treaty for MyTreaty {
    fn new(_: &[u32]) -> Self { unimplemented!(); }
}

fn main() {
    let treaty = {
        let negotiators = vec![1, 2, 3];
        write_treaty::<MyTreaty>(negotiators.as_slice())
    };
    treaty.ratify();
}

fn write_treaty<'a, T: Treaty>(negotiators: &'a [u32]) -> Box<Treaty> {
    Box::new(T::new(negotiators)) as Box<Treaty>
}
```

the compiler complains:

    the parameter type `T` may not live long enough
    consider adding an explicit lifetime bound `T: 'static`
    so that the type `T` will meet its required lifetime bound

What bad behavior is this error preventing?
<details>

It prevents `Box<TraitTreaty>` from outliving a reference to `negotiators`.
</details>

---

What's this `for<'x>` about?

```rust
fn splat_map<'a, F>(list: &mut Vec<i32>, mut f: F)
where F: for<'x> FnMut(&'x i32) -> &'a i32
{ … }
```

<details>

`F` should work with any possible lifetime.
</details>

What's the simplest possible closure that can be `F`?
<details>

```rust
|_| &0
```
</details>

How about a more realistic one?
<details>

```rust
|x| if *x > 0 { &1 } else { &-1 }
```

</details>

Give an example of an invalid `F`.
<details>

```rust
|x| x
```

</details>


<!--

There's something weird about these kinds of situations...
trait MyTrait<'a, T: 'a> {
    ...
}


double-lifetimes. They're pretty yikes!
    &'a Something<'a>
Question: Could the compiler elide the lifetime parameter? What if there were more?

-->
