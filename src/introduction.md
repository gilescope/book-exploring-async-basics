# Introduction

This book aims to take a deep down into the why and how of concurrent programming. First we build
a good foundation of knowledge, before we use that knowledge to implement a toy version of the
Node.js runtime.

> This book is developed in the open and has [it's repository here](https://github.com/cfsamson/book-investigating-async-basics).
> The final code in this book is located here if you want to clone it and play with it or improve it.

Don't block! Dont poll! Increase throughput! Don't block the eventloop! Async code. Concurreny. Paralellism. You've most
likely read this before, and maybe, at some point you've thoguht you've understood everything
only to find yourself confused a moment later. Join me as we try to fix that. I warn you though, we need to venture
from philosophical heights where we try to formally define a "task" all the way down to the deep waters where
firmware and other strange creatures rule.


> Everything in this book will cover the topics for the three major Operating Systems
> Linux, Macos and Windows. 

# Who is this book for?

You'll have to be adventorous, and curious, and a bit forgiving. Even though we cover some complex 
topics we'll have to simplify them significantly to be able to learn anything from them in a small book. 
You can probably spend the better part of a carrear to be an expert in several of the fields we cover, 
so forgive me already now for not beeing able to cover all of them with the precision and thoroughness
they deserve.

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

So, what du you think? Is the answer yes? We'll then join me on this venture where we try to get a better understanding of all these subjects.

> We'll only use Rusts standard library. The reason for this is that we really want to know how tings
> work, and Rusts standard library strikes the perfect balance for this task. 

You don't have to be a Rust programmer to follow along. This book will have numerous chapters where
we explore concepts, and where the code examples are small and easy to understand. But if you want
to follow along when we try to write more code or clonde the repo and play around with the code you
should probably learn the basics first.

> [You will find everything you need to set up Rust here](https://www.rust-lang.org/tools/install)

## Following along

For this book I use `mdbook`, which has the nice benefit of 


## Disclaimer
- We'll implement a **toy** version of the Node.js eventloop (a bad, but working and conceptually similar eventloop)
- We'll **not** primarily focus on code quality and safety, though this is important, we want to understand the concepts and ideas behind the code. We will have to take many shortcuts to keep this concise and short. If we don't we end up reimplementing `libuv` at some point :)
- I will however do my best to point out hazards and the shortcuts we make. I will try to point out obvious places we could do a better job, and I will not use `unsafe` needlessly unless there is a very good reason to do it.
- The book(s) you're reading is the result of a few hundred hours of investigations. I will not claim expertise beyond that, but I will guarantee that I try to verify all the information from more than one source unless it's directly from official documentation.


## Why I write this

I'm curious, and I really hate the feeling of having big gaps in my understanding of a subject. I initially wanted to write a short article about async code and tie it in to Rust Futures. As I started digging and uncover the gaps I had, that idea for an article have expanded into 4 small books on the subject.

Now, some of this information and insight is hard to come by, so I write it down in these books and share my findings with everyone else. I hope you find it as interesting as me.




This book is part of a series investigating several aspects and methods of handling async code execution:

- [Green threads explained in 200 lines of Rust](https://app.gitbook.com/@cfsamson/s/green-threads-explained-in-200-lines-of-rust/)
- Investigating Async Basics by Implementing the Node.js Eventloop in Rust (The book you're reading now)
- Investigating Epoll, Kqueue and IOCP with Rust (Will be released October 2. 2019)
- Investigating Rusts Futures (TBD)
