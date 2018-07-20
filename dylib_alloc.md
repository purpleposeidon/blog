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
static ALLOC: Alloc = Alloc;
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
I've been getting that error a lot, and so far it hasn't helped.
Okay! So let's use solution #2.

It doesn't work. Jemalloc remains embedded in libstd. Oh no!

But you are in luck, for I have come bearing solutions!

Just make `main` a dylib.

Now *everybody* uses the same allocator by default.
You'll just need the weeist li'l shim in `bin/`:
```rust
extern crate app;
fn main() { app::main() }
```

You'll have to specify `#[global_allocator]` using [the system allocator](https://doc.rust-lang.org/unstable-book/library-features/alloc-system.html),
even though obviously that shouldn't be necessary, since it's a dylib and *we all know that* dylibs use the system allocator.
Since that's unstable, you'll have to be on [nightly](just_use_nightly.md).

So in `Cargo.toml` you'll need to set the crate-type for your main library, something like:

```toml
[lib]
crate-type = ["dylib"]
# Note that "cdylib" doesn't work; it must be a "dylib".
# And of course your plugins should already have been ["dylib"].
```

It sort of works? It actually doesn't. Running Valgrind shows the occasional strange memory error, and gdb will happily tab-complete symbols like `je_arena_palloc`. At least now the errors are less common.

Well, we want this to actually work. So where are those jemalloc symbols coming from? We can tell gdb to break whenever a library loads using `set stop-on-solib-events 1` (using `break dlopen` misses stuff). `run`ning first stops sometime during `_start`, and from there we can `c`ontinue and try tab-completing some `je_` symbol. Once we find such a symbol, we'll see that it's coming from libstd.so, and we can also see by running `objdump -CRrt $(dirname $(rustup which rustc))/../lib/libstd* | grep je_ | less`.

Oh, right. libstd still uses the default allocator. It's not like it gets recompiled whenever you change `#[global_allocator]`<sup>[1](#note1)</sup>. So, we need a libstd without jemalloc?

# Grab Your Razors, We're Goin' Yak Shavin'
Let's try to build a libstd without reference to jemalloc.
First we clone the [rust repo](https://github.com/rust-lang/rust/).
We'll want our custom Rust to be otherwise the same as our nightly version. `rustc -vV` will show the commit it was built with; `git checkout` that commit.

So, do we have to take a hacksaw to the Rust sources? No! There's a `config.toml` file with a `use-jemalloc` option. Let's uncomment and set it to false. Now, Rust is built with a python script called x.py. Running `x.py build` will build, and it'll also take something like an hour and a half. ...Yeah. (Maybe `x.py stage1` would be faster? But this is not what I did.)

Eventually you'll come back from your nice walk. Your CPU will stop imitating a jet engine. We can install the custom Rust. Let's first verify that jemalloc is properly sodded off to heck:

```
# Check libstd the same way we checked earlier.
objdump -CRrt build/x86_64-unknown-linux-gnu/stage2/lib/libstd-*.so | grep je_

# Check everything else. Since `find` finds things `objdump` can't handle, use `strings` instead.
# I saw a few unrelated strings.
for f in $(find ./build/x86_64-unknown-linux-gnu/stage2 -type f); do strings $f | grep je_; done

# Okay, now let's install. Don't go removing your rust-src directory, rustup symlinks into it.
rustup toolchain link no-jemalloc ./build/x86_64-unknown-linux-gnu/stage2/
```


<a name="note1">1</a>: This means it's possible to segfault with safe Rust. Laaame. It also suggests that `Cargo.toml` deserves some of the scrutiny that `unsafe` gets.
