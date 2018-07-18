# (WIP) `let else use`

There's an annoyance.

```rust
let foo = if let Some(foo) = foo {
    foo
} else {
    continue;
};
```

Yeah, it sucks. How about this:

```rust
let Some(foo) = foo else use {
    continue;
};
```

# Okay, but why require `use`?

It would conflict with existing syntax:

```rust
let Some(line) = if we_can("dance") {
    if_we_want_to()
} else {
    leave_your_friends(behind)
} else if your_friends() {
    do_not_dance
} else {
    theyre.no_friend_of_mine()
};
```

Certainly we could require parens:

```rust
let Some(line) = (if we_can("dance") {
    if_we_want_to()
} else {
    leave_your_friends(behind)
}) else (if your_friends() {
    do_not_dance
} else {
    theyre.no_friend_of_mine()
});
```

... well, that little hack might just fly in C, but this here is Rust country.

<div align="center">🦀🦀🦀🦀🦀🦀🦀🦀</div>

# More sugar?

If you're just using `return` or `continue`,
then the `use { … }` is unnecessary.

```rust
let Some(line) = blah else return Err("unexpected EOF");
```

The rest of rust has braces tho. More verbose?
```rust
let Some(line) = blah else use {
    return Err("unexpected EOF");
};
```

The first case has like 4 things going on in a single line.
But if it were `continue`, it'd be 3.

# Repetition

Sure, why not?
Every block in a `let else use` must be refutable, except the last one.

```rust
let Some(line) = {
    process1()
} else use {
    process2()
} else use {
    process3()
} else use {
    return Err("Got your nose!");
};
```

# Weird bits
Using it in match wouldn't make sense...

```rust
match line {
    Ok(Some(line)) else use { Ok(Some("<missing line!>")) } => process(line),
    ! => ‽
}
```

What about function arguments!? That'd be *way* too strange!

# There's probably some real code out there that'd be greatly cleaned up by this.

And this would be a great place to put it.
