# Introduction

This book aims to take a deep down into the why and how of concurrent programming. First we build
a good foundation of knowledge, before we use that knowledge to implement a toy version of the
Node.js runtime.

> This book is developed in the open and has [it's repository here](https://github.com/cfsamson/book-investigating-async-basics).
> The final code in this book is located here if you want to clone it and play with it or improve it.

Don't block the eventloop! Don't poll in a loop! Increase throughput! Concurrency.
Parallelism.

You've most likely heard and read about this many times before,
and maybe, at some point you've thought you understood everything
only to find yourself confused a moment later. Especially when you want to
understand how it works on three different Operating Systems.

Me too.

So I spent a couple of hundred hours to try to fix that for myself. Then I wrote
this story and now I invite you to join me on that journey.

I warn you though, we need to venture from philosophical heights where we try to
formally define a "task" all the way down to the deep waters where firmware and
other strange creatures rule (I believe some of the more wicked creatures there
are tasked with naming low level OS syscalls and structures on Windows. However, I
have yet to confirm this.).

> Everything in this book will cover the topics for the three major Operating Systems
> Linux, Macos and Windows.

## Who is this book for?

You'll have to be adventurous, curious, and a bit forgiving. Even though we
cover some complex topics we'll have to simplify them significantly to be able
to learn anything from them in a small book. You can probably spend the better
part of a career becoming an expert in several of the fields we cover, so forgive
me already now for not being able to cover all of them with the precision,
thoroughness and respect they deserve.

However, this book might be interesting for you if you:

- Want to know the difference between parallel and concurrent, and finding a mental model to explain why concurrency is valuable.

- Are curious on how to make syscalls on three different platforms using only Rusts standard library.

- Think it's fun to see if you can generate a software interrupt and make a syscall using inline assembly.

- Want to know more about how OS and the CPU handles concurrency.

- Think figuring out how your code, the OS, your CPU, a device driver and some firmware handles I/O is interesting.

- Can accept a detour where we see how the CPU "knows" if a memory address is invalid.

- Have read enough articles about it but want to know more about what the Node.js eventloop really is, and why most diagrams of it on the web are pretty misleading.

- Think using the knowledge from our research to write a **toy** node.js runtime is pretty cool.

- Already know some Rust but want to learn more.

So, what du you think? Is the answer yes? We'll then join me on this venture
where we try to get a better understanding of all these subjects.

> We'll only use Rusts standard library. The reason for this is that we really want to know how tings
> work, and Rusts standard library strikes the perfect balance for this task providing abstractions
> but they're thin enough to let us easily peek under the covers to see how things work.

You don't have to be a Rust programmer to follow along. This book will have numerous chapters where
we explore concepts, and where the code examples are small and easy to understand, but it will
be more code towards the end and you'll get the most out of it by learning the basics first.

I do recommend that you read my book preceding this [Green threads explained in 200 lines of Rust](https://app.gitbook.com/@cfsamson/s/green-threads-explained-in-200-lines-of-rust/)
since I cover quite a bit about Rust, stacks, threads and inline assembly there and
will not repeat everything here. However, it's definitely not a must.

> [You will find everything you need to set up Rust here](https://www.rust-lang.org/tools/install)

## Following along

For this book I use `mdbook`, which has the nice benefit of being able to run
the code we write directly in the book. However, this only runs the Linux version
and since part of the challenge here is to write for three platforms I've included
and explained that code too, but you'll have to copy it over and run it yourself.

## Disclaimer

1. We'll implement a **toy** version of the Node.js eventloop (a bad, but working and conceptually similar eventloop)

2. We'll **not** primarily focus on code quality and safety, though this is important,
I will focus on understanding the concepts and ideas behind the code. We will have to make
many shortcuts to keep this concise and short.

3. I will however do my best to point out hazards and the shortcuts we make.
I will try to point out obvious places we could do a better job or take big
shortcuts.

If you see something that is imprecise or even wrong I really hope you'll consider
contributing to make this better for the next person reading it. 


## Credits

Substantial contributions will be credited here.

## Why I wrote this and companion books

This started as a wish to write an article about Rusts Futures 3.0, but has now
expanded into 3 finished books about concurrency in general and hopefully, at
some point a fourth about Rusts Futures exclusively.

This process has also made me realize why I have vague memories from my childhood
of threats being made about stopping the car and letting me off if I didn't stop
asking "why?" to everything.

Basically, the list below is a result of this urge to understand _why_ while
reading the RFC's and discussions about Rusts async story: 

- [Green threads explained in 200 lines of Rust](https://app.gitbook.com/@cfsamson/s/green-threads-explained-in-200-lines-of-rust/)

A book where we explore green threads by implementing our own green threads in Rust.

- Investigating Async Basics by Implementing the Node.js Eventloop in Rust

This is the book you're reading now.

- Investigating Epoll, Kqueue and IOCP with Rust
 
Will be released October 2. 2019. We implement an extremely simple and limited
cross platform eventloop based on Epoll, Kqueue and IOCP. Even though it's simple
and bad, it will be working and will be "easy" to understand. A good place
to start if you want to dig further.

We use this library in this book, but it was too much to include here.

- Investigating Rusts Futures (TBD)

This book has not even started yet, and I will see if I can provide a useful
alternative or add anything usefull at all since there is so much being written
about this right now.
