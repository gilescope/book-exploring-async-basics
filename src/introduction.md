# Introduction

This book is part of a series investigating several aspects and methods of handling async code execution:

- [Green threads explained in 200 lines of Rust](https://app.gitbook.com/@cfsamson/s/green-threads-explained-in-200-lines-of-rust/)
- Investigating Async Basics by Implementing the Node.js Eventloop in Rust (The book you're reading now)
- Investigating Epoll, Kqueue and IOCP with Rust (Will be released October 2. 2019)
- Investigating Rusts Futures (TBD)

Our main goal is to get a solid understanding of the inner secrets of Async code, using that knowledge to demystify Rusts Futures evolving async story.


## What we'll do

- We'll go through the basics of asynchronous code execution, some history and basic definitions
- We'll try to be very precise and define what asynchronous is and what parallel is since this is important to make a good mental model of to understand the rest
- We'll talk about how the Operating System and the CPU and explain the difficulties we try to solve that leads to async code
- We'll talk about interrupts, scheduling, firmware and threads
- We'll go through different methods of handling async, like green threads, threadpools and event queues
- We'll make look into how Nodes eventloop works and how it manages to be so efficient in execution async code
- We'll implement our own working toy version of the Node.js eventloop.

## Disclaimer
- We'll implement a **toy** version of the Node.js eventloop (a bad, but working and conceptually similar eventloop)
- We'll not primarily focus on code quality and safety, though this is important, we want to understand the concepts and ideas behind the code. We will have to take many shortcuts to keep this concise and short. If we don't we end up reimplementing `libuv` at some point :)
- I will however do my best to point out hazards and shortcuts we make. I will try to point out obvious places we could do a better job, and I will not use `unsafe` needlessly unless there is a very good reason to do it.
- These are findings of a few hundred hours of investigations, I will not claim expertise beyond that, but I will guarantee that I try to verify all information more than one source unless it's directly from official documentation.



## Why I write this

I'm curious, and I really hate the feeling of having big gaps in my understanding of a subject. I initially wanted to write a short article about async code and tie it in to Rust Futures, but as I started digging and cover the gaps I had that seems to have expanded into small 4 books on the subject.

Now, some of this information and insight is hard to come by, so I write it down in these books and share my findings with everyone else. I hope you find it as interesting as me.