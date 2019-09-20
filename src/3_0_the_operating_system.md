# The Operating System

The operating system stands in the center of everything we do as programmers,
so there is no way for us to discuss any kind of fundamentals in programming
without talking about operating systems in a bit of detail. 

## Concurrency from the operating systems perspective

> Operating systems has been "faking" synchronous execution since the 90's.

This ties into what I talked about in the first chapter when I said that `concurrent`
needs to be talked about within a reference frame and I explained that the OS
might stop and start your process at any time.

What we call synchronous code is in most cases code that appears as synchronous 
to us as programmers. Neither the OS or the CPU live in a fully synchronous world.

Operating systems uses `preemptive multitasking` and as long as the operating 
system you're running is preemptively scheduling processes, you won't have a 
guarantee that your code runs instruction by instruction without interruption. 

The operating system will make sure that all important processes gets some time from the CPU to make progress.

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
access to the hardware directly. This assumes that you're writing code that runs
on an operating system though. But our focus here is written in the context of
Linux, Macos and Windows so cooperating with the OS to be as efficient as possible
is unavoidable.

However this also means that to understand everything from the ground up, you'll
also need to know how your operating system handles these tasks.

To be able to work with the operating system, we'll need to know how we can communicate with it and that's exactly what we're going to go through next.

