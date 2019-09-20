# About writing cross platform abstractions

If you isolate the code needed only for Linux and Macos you'll see that it's not many lines of code to write. But once you want to make a cross platform variant, the amount of code explodes. This is a problem when writing about this stuff in general, but we need some basic understanding on how the different operating systems work under the covers. 

My experience in general is that Linux and Macos have simpler API requiring fewer lines of code, and often (but not always) the exact same call works for both systems.

Windows on the other hand is mor complex, requires more "magic" constant numbers, requires you to set up more structures to pass in, and way more lines of code. What Windows does have though are very good documentation so even though it's more work you'll also find the official documentation very good.

This complexity why the Rust community (other languages often has something similar) gathers around crates like [libc](https://github.com/rust-lang/libc) which already have defined most methods and constants you need.
