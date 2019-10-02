# Modules

We're soon at the end now. Actually, our core runtime is finished, but we need
a way to actually make it work with our "javascripty" code.

The modules here would be the equivalent of "C++" modules in Node, and in theory
we could allow for people to make modules that register work to be either done in
our `threadpool` or our `epoll` event queue.

The first one we'll cover is also the simplest, but it will also introduce us to
why we stored a pointer to our `Runtime` as a global constant.

```rust
pub fn set_timeout(ms: u64, cb: impl Fn(Js) + 'static) {
    let rt = unsafe { &mut *(RUNTIME as *mut Runtime) };
    rt.set_timeout(ms, cb);
}
```

To be able in our "javascript" to just call `set_timeout(...)` without injecting our
runtime into our `javascript` function (which we could do if we wanted to write
better Rust) we need to dereference a mutable pointer to our runtime.

Now, we know this is safe for two reasons.

1. `RUNTIME` is always access from one thread
2. We know that all calls to our `modules` will be in `Runtime.run(...)` at which point we know that `RUNTIME` will be a valid pointer

Now it's not pretty, and that's also why I explain why we do it, and why it's safe here. Normally, all `unsafe`
code should contain a reason why it's used, and why it's safe in a comment.

Once we have referenced the pointer to `RUNTIME` we can call methods on it and in 
this case we simply call `rt.set_timeout(ms, cb)` which sets a timeout as you saw
in the last chapter.
