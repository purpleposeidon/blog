# metapipes

    int metaopen(int fd);
    char metarespect(int fd, char mode);

Metapipes are used to replace hacky escape systems.
Any pipe (here called a "basepipe") can have a metapipe opened on it.
If there is something buffered in the metapipe, attempting to read from the basepipe would block, and vice versa.
The metapipe's read/write mode mirrors that of the basepipe.

Metapipe protocols should be human-readable text. Furthermore, their output should look somewhat reasonable if merged.
(Using a binary protocol on a metapipe would be inconsistent: the base pipe's protocol would then just as well be binary, and in that case why not use a binary protocol with in-band signalling on the base pipe?)

# Respect

There is no guarantee that the other program respects the metapipe.
If one process respects the metapipe but the other does not, communication would jam up.

Both processes need to know the other's mind, and it needs to work with ignorant programs.
We know if we respect metapipes, but the other process may:
 - be ignorant
 - respect metapipes, but be slow to start up
 - respect metapipes, but has exec'd a program that does not
 - be on the other end of an SSH tunnel

`metarespect()` manually introspecs the state of an end of a metapipe per this mode:
    'U': set 'unknown'
    'D': set 'disrespect'
    'R': set 'respect'
    '?': query near end state
    '|': query far end state
    NUL: error
The return value is likewise.

Various actions implicitly "mark respect" or "mark disrespect" to the metapipe.
This decides the respect for an end of a metapipe in unknown state.

IO on a metapipe marks respect on the local end.
IO on a basepipe marks disrespect on the local end.

IO on a metapipe returns EINVAL if either end is disrespected. `(_, 'D') | ('D', _)`
IO on a metapipe would block if the other end has unknown respect. `('R', 'U')`
IO on a metapipe can only move forward if both ends are respected. `('R', 'R')`

# Tabular Output Protocol
This is just TSV, with headers, and with no need to escape anything.

The stuff in quotes is written to the basepipe. (Without the quotes.)
The stuff not in quotes is written to the metapipe. (Without the spaces.)

    # "table name"
    | "column name" : "type" |\t| "typeless column" |
    "row data" \t "data"

Also, elements may write "[:,]"'s in the metapipe to indicate nested arrays & dictionaries.


# Terminal Output Control Protocol
I don't think it's possible to make a good metapipe terminal compatible with modern terminals.
Firstly, escape codes are an enormous clusterfuck already. It'd be best to not make them worse.
Secondly, shells can run multiple programs at once. Some might be metapipe ignorant.
They would stomp on eachother if they shared PTTYs, so they'd all have to get unique ones.
Wouldn't that make the terminal very difficult to implement properly?

So we'll need a new terminal. We could have something like [Weaver](https://github.com/tene/weaver).
Ignorant programs would get nested mini-terminals, and mindful programs would get something cooler.

I think we should have a very simple set of escape codes.
If you want a TUI, don't use metapipes.

    [           begin embed
    ]           end embed; push to stack. (Max stack is 1?)
    #Label      begin a section labeled with a u64 number. Title popped off stack.
    dLabel      deletes section
    jLabel      erases contents of section & begins rewritting.
    +biu…       add style
    -biu…       remove style
    +l          pop URL from embed stack; applies until -l
    +@          pops string from stack; sends to dbus process via some 'void metapipe_button(string)' thing.
    i           pop image data (eg, .png) from embed stack
