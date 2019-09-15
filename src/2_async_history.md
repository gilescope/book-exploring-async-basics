# Async history
In the beginning.
Everything was synchronous. Computers had one CPU and it executed a piece of code written by a programmer and then returned. No scheduling, no threads, no multitasking. We're talking back when the days where a program looked like this:

![Image](./images/punched_card_deck.jpg)

There were operating systems though, and when personal computing started to grow in the 80's we had operating systems like DOS, but they usually yielded control of the entire CPU to the program (and the programmer) that was currently running.
This worked fine, but as interactive UI's using a mouse and windowed operating systems became the norm, this simply couldn't work anymore.

## Hyperthreading

As CPU's evolved and added more functionality like several ALUs (Algorithmic Logical Unit) and more logical units in general, the CPU manufacturers realized that the entire CPU was never utilitized fully. For example when an operation only required some parts of the CPU, an instruction could be run on the ALU simultainiously. This became the start of Hyperthreading.¨

You see, on your computer today that it has i.e. 6 cores, and 12 logical cores. This is exactly where Hyperthreading comes in. It "simulates" two cores on the same core by using unused parts of the CPU to drive progress on thread "2" simultainiously as it's running the code on thread "1" by using a number of smart tricks (like the one with the ALU). Now we could actually offload some work on one thread while keeping the UI interactive by responding to events in the second thread event though we only have one CPU core.

You might wonder how this is really different from multicore processors? What about performance? I turns out that Hyperthreading has been improved since the 90's all the time. Since you're not actually running two CPU's there will be some operations that need to wait for each other to finish, and how much of a penalty this gives compared to two seperate cores might depend a bit on exactly what the code is doing, but the numbers I have seen shows "only" a 30 % penalty comparted to two seperate CPU's. In other words, it's pretty good!


## Non-preemptive multitasking

The method used to be able to keep the UI interactive and running background processes, was accomplished by non-preemtive multitasking. This kind of multitasking put the responsibility of letting the OS run other tasks like responding to input from the mouse, or running a background task in the hands of the programmer. Typically the programmeryieldedcontrol to the OS.

Beside offloading a huge responsibility to every programmer writing a program for your platform, this was also error prone. A small mistake in a programs code could halt or crash the entire system. If you remember Windows 95, you also remember the times when a window hung and you could paint the entire screen with it (almost the same way as the end in Solitare, the card game that came with Windows). This was a typical error in the code that was supposed to yield control to the operating system.

If you're not sure about what this is I can recommend my previous book that explains this part of multitasking pretty well. You'll know everything you need about threads, contexts, stacks and scheduling for following along.


## Preemtive multitasking

While non-preemtive multitasking sounded like a good idea, it turned out to create serious problems as well. I will not list them here but as you can imagine, letting every program and programmer out there be responsible for parts of the scheduling of tasks in an operating system will be chaos and ultimately lead to a bad user experience.
So they put the responsibility of scheduling the CPU resources between the programs that requested it (including to OS itself) in the hands of the OS. The OS can stop execution of a process, do something else, and switch back.

In a single core machine you can visualize this as running a program you wrote, and the OS stops to update the mouse position, and witches back to your program. This can happen many times each second, not only to keep the UI responsive but it can also give some time to other background tasks and IO events.

This is now the prevailing way to design an operating system. 


## So how synchronous is the code you write, really ?

As many things this depends on your perspective. From the perspective of the code you write, and you as a programmer, everything will happen in the order you write it.
From the OS perspective it might, or might not, interrupt your code, pause it and run some other code in the meantime before resuming. 
From the perspective of the CPU it will mostly execute* instructions one at a time. They don't care who wrote the code though so when a hardware interrupt happens, they will immediately stop and give control to an interrupt handler, so the CPU doesn't have any concept of asynchronous execution.


However, modern CPU can also do a lot if things in parallel. Most CPUs are pipelined, meaning that the next instruction is loaded while the current is executing. It might have a branch predictor that tries to figure out what instructions to load next. The processor can also reorder instructions by using "out of order execution" if it believes it makes things faster this way without "asking" or "telling" the programmer or the OS so you might not have any guarantee that A happens before B. The CPU offloads some work to separate "coprocessors" like the FPU for for floating point calculations leaving the main CPU ready to do other tasks et cetera.
*As a high level overview, it's OK to model the CPU as operating in a synchronous manner, but lets for now just make a mental note that this is a model with a ton caveats that becomes especially important when talking about parallelism, synchronization primitives like mutexes and atomics and security.