# The Operating System

The operating system stands in the center of everything we do as programmers, so there is no way for us to discuss any kind 
of fundamentals in programming without talking about operating systems in a bit of detail. But since we rely on the 
operating system as much as we usually do when performing I/O operations we'll have to learn how to communicate with the
OS.

## Concurrency from the operating systems perspective

> Operating systems has been "faking" synchronous execution since the 90's.

This ties into what I talked about in the first chapter when I said that `concurrent`
needs to be talked about within a reference frame and I explained that the OS
might stop and start your process at any time.

What we call synchronous code is in most cases code that appears as synchronous 
to us as programmers. Neither the OS or the CPU live in a fully synchronous world.

Operating systems uses `preemptive multitasking` and as long as the operating 
system you're running is preemptively scheduling processes, you won't have a 
guarantee that your code instruction by instruction without interruption. 

The operating system will make sure everything that all important processes gets 
some time from the CPU to make progress. 

> This is not as simple when we're talking about modern machines with 4-6-8-12 
> physical cores since you might actually execute code on one of the CPU's 
> uninterrupted if the system is under very little load. The important part here 
> is that you can't know for sure and there is no guarantee that you code will be 
> left to run uninterrupted.


## Teaming up with the OS.

When programming it's often easy to forget how many moving pieces that need to 
cooperate to reach maximum efficiency. When you make a web request, you're not 
asking the CPU or the network card to do something for you, you're asking the 
operating system to talk to the network card for you. 

There is no way for you as a programmer to make your system optimally efficient 
without playing to the operating systems strengths. You basically don't have 
access to the hardware directly. 

The good thing is that most operating systems are more secure and more optimized 
than what we would most likely end up with if we had direct access to the hardware.

However this also means that to understand everything from the ground up, you'll
also need to know how your operating system handles these tasks.


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


