# The Operating System

The operating system stands in the center of everything we do as programmers, so there is no way for us to discuss any kind 
of fundamentals in programming without talking about operating systems in a bit of detail. But since we rely on the 
operating system as much as we usually do when performing I/O operations we'll have to learn how to communicate with the
OS.

## Concurrency from the operating systems perspective






## Writing cross platform abstractions
If you isolate the code needed only for Linux and Macos you'll see that it's not many lines of code to write. But once you
want to make a cross platform variant, the amount of code explodes. This is a problem when writing about this stuff in general,
but we need some basic understanding on how the different operating systems work under the covers. 

My experience in general is that Linux and Macos have simpler api's requiring fewer lines of code, and often (but not always) 
the exact same call works for both systems.

Windows on the other hand is mor complex, requires more "magic" constant numbers, requires you to set up more structures to pass in, 
and way more lines of code. What Windows does have though are very good documentation so even though it's more work you'll also 
find more official documentation.

This is why the Rust community (and other languages often has something similar) gathers around crates like [libc](https://github.com/rust-lang/libc) 
which already have defined most methods and constants you need.

# Threads

Threads are one of the baisc constructs for running code that we programmers create. Like the syscall for outputting text to the console, 
there is a syscall for asking the OS to create a thread. 

Now threads are a bit special, since they are the thing that actually makes us think that we write **synchronous** code at all.

Let's stop for a second and think a bit about the significance of threads for our the problem domain we're investigating here:

The "naive" way of thinking about your code is like this:




Now threads

memory

faking sync



