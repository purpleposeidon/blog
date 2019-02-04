# EZ Allocation

You can have an allocator that works like this:

```
fn allocate(&mut self, size: usize) -> *mut () {
    let ret = self.mark;
    self.mark += size;
    ret
}
```

Where `mark` starts off pointing to a page of memory handed out by the kernel.
Each time the user accesses memory that hasn't actually been allocated, a SEGV signal happens.
`siginfo_t` contains information about the address the SEGV occured at.
The handler finds the allocators for that range.
If `mark` is larger than the actual available allocation,
we ask the kernel for more pages of memory, doubling the size each time.
