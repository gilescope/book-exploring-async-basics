# Introduction

We’ll dive deep down into the why and how of concurrent programming. I try to cover a lot of ground here. Maybe too much but if you’re like me, and curious I hope you’ll enjoy it anyways.

> Everything in this book will cover the topics for the three major Operating Systems
> Linux, Macos and Windows.

The parts I found the coolest to research and write about were:

- Defining the difference between parallel and concurrent, and finding a mental model to explain why concurrency is valuable.

- Learning to make syscalls on three different platforms using only Rusts standard library.

- Finding a short and easy way to generate a software interrupt and make a syscall using inline assembly.

- Learning how the OS and the CPU handles concurrency, and seeing how the pieces fit together in the light of my previous book about `green threads`.

- Figuring out how the the firmware and the device driver plays a role when we write code that does I/O.

- Solving a question that been nagging me about exactly how the CPU know we’re accessing invalid memory when writing assembly.

- Learning what the Node.js eventloop really is, and why most diagrams of it on the web are pretty misleading.

- Using the knowledge from our research to write a **toy** node.js runtime.

- Digging into what epoll, kqueue and IOCP and digging into the source code of `mio` and `libc` (next book)

- Implementing a very bad, but working cross platform eventloop using nothing but the standard library (next book)

So check this list. Do you know all of this already? No? Are you curious about any of this? We’ll join me as we dig into these topics and venture down the rabbit hole.


This book is part of a series investigating several aspects and methods of handling async code execution:

- [Green threads explained in 200 lines of Rust](https://app.gitbook.com/@cfsamson/s/green-threads-explained-in-200-lines-of-rust/)
- Investigating Async Basics by Implementing the Node.js Eventloop in Rust (The book you're reading now)
- Investigating Epoll, Kqueue and IOCP with Rust (Will be released October 2. 2019)
- Investigating Rusts Futures (TBD)

Our main goal is to get a solid understanding of the inner secrets of Async code, using that knowledge to demystify Rusts Futures evolving async story.


## External dependencies
We will only rely in the standard library for this, making sure that we understand everything and leave no gaps uncovered. While this is a pretty big constraint when using Rust it does require us to answer certain basic questions that would otherwise go unasked.

For this to work I had to make a library for the cross platform epoll/kqueue/IOCP eventloop that is explained in depth in the next book.

## Disclaimer
- We'll implement a **toy** version of the Node.js eventloop (a bad, but working and conceptually similar eventloop)
- We'll not primarily focus on code quality and safety, though this is important, we want to understand the concepts and ideas behind the code. We will have to take many shortcuts to keep this concise and short. If we don't we end up reimplementing `libuv` at some point :)
- I will however do my best to point out hazards and the shortcuts we make. I will try to point out obvious places we could do a better job, and I will not use `unsafe` needlessly unless there is a very good reason to do it.
- The book(s) you're reading is the result of a few hundred hours of investigations. I will not claim expertise beyond that, but I will guarantee that I try to verify all the information from more than one source unless it's directly from official documentation.


## Why I write this

I'm curious, and I really hate the feeling of having big gaps in my understanding of a subject. I initially wanted to write a short article about async code and tie it in to Rust Futures. As I started digging and uncover the gaps I had, that idea for an article have expanded into 4 small books on the subject.

Now, some of this information and insight is hard to come by, so I write it down in these books and share my findings with everyone else. I hope you find it as interesting as me.