# Introduction

This book is part of a series investigating several aspects and methods of handling async code execution:

- [Green threads explained in 200 lines of Rust](https://app.gitbook.com/@cfsamson/s/green-threads-explained-in-200-lines-of-rust/)
- Investigating Async Basics by Implementing the Node.js Eventloop in Rust (The book you're reading now)
- Investigating Epoll, Kqueue and IOCP with Rust (Will be released October 2. 2019)
- Investigating Rusts Futures (TBD)

Our main goal is to get a solid understanding of the inner secrets of Async code, using that knowledge to demystify Rusts Futures evolving async story.


## What we'll do

- Go through the basics of asynchronous code execution, some history and basic definitions
- Be very precise and define the difference between async and parallel
- Talk about how the Operating System and the CPU in regards to I/O and async
- Dig into how interrupts, scheduling, firmware and threads relate to our subject
- Go through different methods of handling async, like green threads, threadpools and event queues
- Look at how Nodes eventloop works and how it manages to be so efficient in execution async code
- Implement our own working toy version of the Node.js eventloop using our knowledge

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