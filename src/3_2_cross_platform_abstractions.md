# About writing cross platform abstractions

If you isolate the code needed only for Linux and Macos you'll see that it's not many lines of code to write. But once you want to make a cross platform variant, the amount of code explodes. 

This is a recurring problem when we're curious about how this works on three different platforms, but we need some basic understanding on how the different operating systems work under the covers.

My experience in general is that Linux and Macos have simpler API requiring fewer lines of code, and often (but not always) the exact same call works for both systems.

Windows on the other hand is more complex, requires you to set up more structures to pass information (instead of using primitives), and often way more lines of code. What Windows does have is very good documentation so even though it's more work you'll also find the official documentation very helpful.

This complexity why the Rust community (other languages often has something similar) gathers around crates like [libc](https://github.com/rust-lang/libc) which already have defined most methods and constants you need.

## A note about "hidden" complexity

There is a lot of "hidden" complexity when writing cross platform code at this
level. One hurdle is to get something working, which can prove to be quite a
challenge. Getting it to work **correctly** and **safely** while covering all
edge cases is an additional challenge. 

**Are we 100% sure that all valid `utf-8` code points which we use in Rust is valid 
`utf-16` encoded Unicode that Windows will display correctly?**

I think so, but being 100 % sure about [this is not as easy as one might think](https://en.wikipedia.org/wiki/Comparison_of_Unicode_encodings).
