# nameless: Probably a Novel Language Paradigm (Warning: 97% vaporware)

Consider this:

```python
def write_csv_record(output, record):
    output.write(record)
```

In nameless, this could be written as:

```
fn { #WriteFile #CsvRecord }
```

This is not an anonymous function. The name of this function is `implicit`, and `implicit` is overloaded to all heck.
In this case, with two parameters, one being `WriteFile`, and the other being `CsvRecord`.
(In nameless, arguments are unordered, and without duplicates, just like a set.) What do you expect to happen in a function that takes a file and an object? Big mystery!

The `#` means "the argument of type". The argument doesn't need a name, because its type is already has one. Do you often name the parameter after its type, converting to camel\_case, or else using a nickname? You are already using nameless. When the name is entirely different from the type, you stray from the path. It may be a warning. (But I need to investigate this.)
Well if we write the function in a typed language,

```rust
fn write_csv_record(output: &mut Fd, record: &CsvRecord) {
    write!(output, "{}", record);
}
```

`record` shows up 4 times. This is in excess of DRY rule of thumb, "don't repeat anything more than three times". How about once? Nameless gets us once.

<hr>

So in nameless, the only language construct that requires an identifier is the type, and then there is the obligatory off-by-one error; thus the language has named itself "nameless".

There's kind of a leap of faith that everything works out. You'll have to put your faith in the World Spirit, or in Math, or in your IDE's refactor tools. (Since the nameless compiler is also its IDE, they're better than Java's.)
I've been testing out the nameless principle, in real Rust code — to simplify the details: there is a `HashMap<TypeId, Any>`, and I can call functions, and their arguments are populated from the `HashMap`; the truth of my test is in my crate [v9](https://docs.rs/v9/0.1.37/v9/) + my video game — and I've found that it can sometimes be a bit of a pain. Sometimes I might have multiple `f32`s, and I'd have to stick them each in their own wrapper struct, or unite them into a single struct. It would be better if I could add arbitrary tags to types dimensiona-analysis style without boilerplate. But this test is shallow.


(I'm not super-confident about the 'all functions overload `implicit`' thing. Maybe you can name functions other things, or maybe functions can have unique names.)
