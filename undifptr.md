# Generic Over Mutability

It's annoying to have to implement `get()` and `get_mut()` for everything when they're basically the same thing.
We could introduce Yet Another pointer type, "undifferentiated pointers".

I'll use the syntax `&mut?` here, tho it isn't terribly easy on the eyes. The semantics:

1. `&T` and `&mut T` automatically coerece into `&mut? T`.
2. `&mut? T` automatically coerces into `&T`.
3. `&mut? T` can coerce into `&mut T` only if the start of the lifetime is in scope, and is borrowed from something mutable.

So if you have some `&mut? T`, you'll be able to call `&T` methods, but you can only call `&mut T` if the compiler knows the thing it came from is mutable.

    impl i32 {
        fn get(&mut? self) -> &mut? i32 {
            self
        }
    }
    let mut x = 0;
    *x.get() += 1;
    let y: &i32 = x.get();


`libstd` could have many backwards-compatible changes, along the lines of:

    impl Vec<T> {
        fn first(&mut? self) -> Option<&mut? T> { … }
        #[deprecated]
        fn first_mut(&mut self) -> Option<&mut T> { … }
    }

However the deprecation would cause a lot of code churn, even if softly deprecated.



It might be possible to implement this in a library if every function involved had some kind of `#[syntax_transform]` thing.
