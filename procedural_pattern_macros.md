# (WIP) Procedural Pattern Macros

Let's write a macro, `#[profiled]`, that instruments a function for us.

```rust
#[profiled]
fn my_function(kitty: &mut Cat, love_and_affection: AffectionLevels) {
    kitty.feed_with(love_and_affection);
    kitty.water_with(love_and_affection);
    kitty.pet_with(love_and_affection);
}
```
This looks pretty easy to parse!

```rust
#[profiled]
pub(::super) extern "rust" unsafe fn r#or_IS_it<I>(i: I) -> I {
    struct DunDunDun;
    impl DunDunDun {
        fn dramatic_zoom() { panic!() }
    }
    i
}
```
You see this? Looks like a mess? I'd better not have to work hard to parse that.

Let's see a possible implementation.

```rust
use std::macros::{Result, TokenStream};

pub macro profiled(_attr: TokenStream, tt: TokenStream) -> Result {
    macro match tt {
        ($head:fn $name:ident $foot:fnshape $body:tt) => {
            // head = pub(::super) extern "rust" unsafe fn
            // name = or_IS_it
            // foot = (i: i8) -> i8
            // body = { struct DunDunDun; … }
            let body = profile_lines(name, &mut 0, body);
            Ok(quote! {
                $fnmods $name $body
            })
        },
        _ => Err(tt.error("expected function")),
    }
}

fn profile_lines(
    name: &str,
    line: &mut usize,
    tt: TokenStream,
) -> Result {
    let mut out = quote! { PROFILER.enter($name); };
    for item_or_statement in tt {
        macro match item_or_statement {
            () => return,
            ($item:item) => out += $item,
            ($expr:expr) => {
                out += insert_profiling(name, line, expr.into())?;
                out += quote! {
                    PROFILER.step($name, $n);
                }
                line += 1;
                // FIXME: what about flow control?
            },
            ($tt:tt) => out += insert_profiling(name, line, tt)?,
            _ => Err(item_or_statement.error("unknown syntax")),
        }
    }
    out += quote! { PROFILER.exit($name) };
    Ok(out)
}
```

Let's see you do THAT with macros-by-example!
Sure, it's *possible*.
And just as surely as it's possible,
everyone who comes into contact with it will want to gouge out their eyes.
What macros-by-example does really well is the matching bit. And then filling out a template.
But if you want to do anything "complicated", it very quickly turns into a Turing tarpit.
For example, the procedural macro is incrementing a line number.
With macros-by-example, you can implement `line += 1` by recursing on an array
and adding a dummy element each iteration,
but keep in mind that
NO THANKS NOT FOR ME I HAVE NO DESIRE TO GET MY HANDS COVERED IN THE TURING TAR.

Plus the errors suck.
This approach gives you the best of both worlds: we have syntax matching;
and we can write our macros in a very nice language we all know and love.

Rust. That language is called Rust. Fantastic language. Not that I'm one to evangelize.
The errors'll probably still suck by default,
but the lack of tar will make them more like `vague` and less like `nightmarish tentacled monstrosity from La Brea`.
Making the error messages decent probably wouldn't be that hard?

(FIXME: The pretense of narrative coherency ends here.)

You could even implement `format!`. Can't do that in macros-by-example.

The other problem with macros-by-example is the impoverished selection of `:syntaxtypes`.
Isn't there an RFC? Get you some that, and we'll be in business.

I think there's some people (the ones actually working on macros2.0) that are worried about API stability?
Well, pattern matching is a transparent thing. No API, so no worry?
Ah, but there'd probably be a bit of introspection you'd want to do,
such as the name of an item, or its visibility modifier.
But at least the syntax is the *bulk* of the API, and it's got a built-in stability guarantee!
There would be the same issues of introducing new syntax that macros-by-example has,
such as old macros not supporting `pub(crate)`.
Unavoidable?

