# Getting working memory allocation in a plugin system

The typical plugin architecture doesn't seem to work very well with Rust these days.
For example, [this person](https://reddit.com/r/rust/comments/8rh5al/segfaulting_when_using_a_dynamic_library/)'s
had trouble. Also me. It's been... pretty rough.
This is because dylibs use malloc, but rlibs use jemalloc.
Obviously the plugins and the main program need to be using the same allocator!
We can indicate what allocator to use via
[this thing](https://doc.rust-lang.org/unstable-book/language-features/global-allocator.html):

```rust
extern crate some_allocator;
use some_allocator::SomeAllocator as Alloc;
#[global_allocator]
ALLOC: Alloc = Alloc;
```

(If we want to use malloc aka [`alloc_system`](https://doc.rust-lang.org/unstable-book/library-features/alloc-system.html), we must use unstable's `#![feature(alloc_system)]`.)

Basically nothing works.
By this I mean that, maybe it works if you have a simple setup, but more complex ones do not.
I don't know what "complex" actually means here,
but it probably involves things like "plugin refers to items in main"
and "you've actually somehow gotten jemalloc's code to run
(because I still have no idea how to do this in an isolated environment;
obvious things like 'give a `Vec` to a library to fiddle with & drop' don't work)".


So we need to use `#[global_allocator]` somehow to unify the allocators.
These are the immediate, obvious options:

1. Leave main alone, tell the dynamic libraries to use jemalloc.
2. Tell main to use `alloc_system`, leave the dynamic libraries alone.
3. Tell everyone to use jemalloc, explicitly.
4. Tell everyone to use `alloc_system`, explicitly.

3 & 4 will not work.
There can be only one crate with `#[global_allocator]`.
1 will not work, because we get this error: "cannot allocate memory in static TLS block".
I've been getting that error a lot, and so far it hasn't helped anything.
Okay! So let's use solution #2.

It doesn't work. Jemalloc remains embedded in libstd. Oh no!

But you are in luck, for I have come bearing solutions! (Sorta.)

Make main a dylib. Now everybody uses the same allocator by default.
You'll just need the weeist li'l shim in `bin/`:
```rust
extern crate app;
fn main() { app::main() }
```

You'll have to specify `#[global_allocator]` using [the system allocator](https://doc.rust-lang.org/unstable-book/library-features/alloc-system.html),
even though obviously that shouldn't be necessary, since it's a dylib and dylibs use the system allocator.
Since that's unstable, you'll have to be on [nightly](just_use_nightly.md).

So in `Cargo.toml` you'll need to set the crate-type for your main library, something like:

```toml
[lib]
crate-type = ["dylib"]
# Note that "cdylib" doesn't work; it must be a "dylib".
# And of course your plugins should already have been ["dylib"].
```

It sort of works? It actually doesn't. Running Valgrind shows the occasional strange memory error, and gdb will happily tab-complete symbols like `je_arena_palloc`. Memory errors at least happen less often this way.
