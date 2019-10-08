# Introduction

**Don't block the event loop! Don't poll in a busy loop! Increase throughput! Use async I/O! Cuncurrency is not parallelism!**

You've most likely heard and read claims like these many times before,
and maybe, at some point you've thought you understood everything
only to find yourself confused a moment later. Especially when you want to
understand how it works on a fundamental level.

**Me too.**

So I spent a couple of hundred hours to try to fix that for myself. I wrote
this book as a result of that reserch, and now I invite you to join me as we try
to unvail the secrets of async programming.

This book aims to take a look at the **why** and **how** of concurrent programming. First we build
a good foundation of basic knowledge, before we use that knowledge to investigate how Node.js works
by building a Node-inspired runtime.

> This book is developed in the open and has [it's repository here](https://github.com/cfsamson/book-investigating-async-basics).
> The book and the [accomanying code](https://github.com/cfsamson/examples-node-eventloop) is MIT licensed so feel free to clone away
> and play with it.

I warn you though, we need to venture from philosophical heights where we try to
formally define a "task" all the way down to the deep waters where firmware and
other strange creatures rule (I believe some of the more wicked creatures there
are tasked with naming low level OS syscalls and structures on Windows. However, I
have yet to confirm this).

> Everything in this book will cover the topics for the three major Operating Systems
> Linux, Macos and Windows. We'll also only cover the details on how this works
> on 64 bit systems.

## Who is this book for?

I originally started out wanting to explore the fundamentals and inner workings
of Rusts Futures. Reading through RFC's, motivations and discussions I realized
that to really understand the **why** and **how** of Rust's Futures, I needed a very good
understanding of how async code works in general, and the different strategies to handle it.

**This book might be interesting if you:**

- Want to take a deep dive into what concurrency is and strategies on how to deal with it

- Are curious on how to make syscalls on three different platforms, and do it on three different abstraction levels.

- Want to know more about how the OS, CPU and hardware handles concurrency.

- Want to learn the basics of Epoll, Kqueue and IOCP.

- Think using our research to write a **toy** node.js runtime is pretty cool.

- Want to know more about what the Node eventloop really is, and why most diagrams of it on the web are pretty misleading.

- Already know some Rust but want to learn more.

So, what du you think? Is the answer yes to some of these questions? Well, then join me on this venture
as we try to get a better understanding of all these subjects.

> We'll only use Rusts standard library. The reason for this is that we really want to know how tings
> work, and Rusts standard library strikes the perfect balance for this task providing abstractions
> but they're thin enough to let us easily peek under the covers and see what really happens.

## Following along

Even though I use `mdbook`, which has the nice benefit of being able to run
the code we write directly in the book, we're working with I/O and cross
platform syscalls in this book which is not a good fit for the Rust playground. 

My best recommendation is to create a project on your local
computer and follow along by copying the code over and run it locally.

[You can also clone or download the example code from the git repository](https://github.com/cfsamson/examples-node-eventloop)

## Prerequisites

You don't have to be a Rust programmer to follow along. This book will have numerous chapters where
we explore concepts, and where the code examples are small and easy to understand, but it will
be more code towards the end and you'll get the most out of it by learning the basics first. In this case [The Rust Programming Language](https://doc.rust-lang.org/book/title-page.html) is the best place to start.

I do recommend that you read my book preceding this [Green threads explained in 200 lines of Rust](https://app.gitbook.com/@cfsamson/s/green-threads-explained-in-200-lines-of-rust/)
since I cover quite a bit about Rust basics, stacks, threads and inline assembly there and
will not repeat everything here. However, it's definitely not a must.

> [You will find everything you need to set up Rust here](https://www.rust-lang.org/tools/install)

## Disclaimer

1. We'll implement a **toy** version of the Node.js eventloop (a bad, but working and conceptually similar eventloop)

2. We'll **not** primarily focus on code quality and safety, though this is important,
I will focus on understanding the concepts and ideas behind the code. We will have to make
many shortcuts to keep this concise and short.

3. I will however do my best to point out hazards and the shortcuts we make.
I will try to point out obvious places we could do a better job or take big
shortcuts.

> Even though we
> cover some complex topics we'll have to simplify them significantly to be able
> to learn anything from them in a small(ish) book. You can probably spend the better
> part of a career becoming an expert in several of the fields we cover, so forgive
> me already now for not being able to cover all of them with the precision,
> thoroughness and respect they deserve. 

## Credits

Substantial contributions will be credited here.

## Contributing

I have no other interest in this than to share knowledge that can be hard to
come by and make it easier for the next curious person to understand. If you want to
contribute to make this better there are two places to go:

1. [The base repo for this book](https://github.com/cfsamson/book-investigating-async-basics) for all feedback and content changes
2. [The base repo for the code example we use](https://github.com/cfsamson/examples-node-eventloop) for all improvements to the example code

Everything from spelling mistakes to correcting errors or inaccuracies are greatly appreciated. It will only make this book better for the next person reading it.

## Why I wrote this and its companion books

This started as a wish to write an article about Rusts Futures 3.0. The result so far is
3 books about concurrency in general and hopefully, at some point a fourth about Rusts Futures exclusively.

This process has also made me realize why I have vague memories from my childhood
of threats being made about stopping the car and letting me off if I didn't stop
asking "why?" to everything.

Basically, the list below is a result of this urge to understand _why_ while
reading the RFC's and discussions about Rusts async story: 

- [Green threads explained in 200 lines of Rust](https://app.gitbook.com/@cfsamson/s/green-threads-explained-in-200-lines-of-rust/)

- [Exploring Async Basics by Implementing the Node Event Loop in Rust](https://cfsamson.github.io/book-investigating-async-basics/) (this book)

- [Exploring Epoll, Kqueue and IOCP with Rust](https://github.com/cfsamson/book-exploring-epoll-kqueue-iocp) a companion book to the "Async Basics" book

- Exploring Rusts Futures (TBD) - a different look on the **why** and **how** of Rusts futures
