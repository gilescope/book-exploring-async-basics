# Epoll, kqueue and IOCP

Part of Nodes runtime is [libuv](https://github.com/libuv/libuv) which is a cross platform
asynchronous I/O library. `libuv` is not only used in Node but also forms the foundation
of how [Julia](https://julialang.org/) and [Pyuv](https://github.com/saghul/pyuv). Most
languages has bindings for it. 

In Rust we have [mio - Metal IO](https://github.com/tokio-rs/mio). Since we want
to understand how everything works from the bottom, I had to create an extremely
simplified version of such a library. I called it `minimio` for obvious reasons.

> I will write a short book (much shorter than this one) about how this works in
> detail, for now you can visit the code in the it's [Github repository if you're
> curious](https://github.com/cfsamson/examples-minimio). This book will also cover
> `wepoll` which is used as an optimization instead of IOCP in both `mio` and `libuv`. 

However, we'll give each of them a brief introduction here so you know what we're
working with.

## Why and OS backed event queue

If you remember my previous chapters you know that we need to cooperate closely
with the OS to make I/O operations as efficient as possible. Operating systems like
Linux, Macos and Windows provides several ways of performing I/O, both blocking and
non-blocking.

As you probably have understood, blocking means only to us, but we yield control over
our thread to the OS. So blocking operations are the least flexible to use for us
as programmers.

Non-blocking metods needs a way to communicate their state to you so you can
tell if they're ready or not, but instead of you asking for a status every now
and then there are better ways.

> We'll not cover methods like `poll` and `select` but I have an [article for you
> here](https://people.eecs.berkeley.edu/~sangjin/2012/12/21/epoll-vs-kqueue.html)
> if you want to learn a bit about these methods and how they differ from `epoll`.

## Readiness based event queue

Epoll and Kqueue are what we call readiness based event queues. They're called
that since they let you know when an action is ready to be performed. For example
when a socket is ready to be read from.

Basically this happens when we want to read data from a socket:

1. We create an event queue by calling the syscall `epoll_create` or `kqueue`
2. We create a file descriptor representing a socket
3. We register an interest in `Read` events on this socket with a second syscall
4. Next, we call, `epoll_wait` or `kevent` to wait for an event - this will block
5. When the event is ready, our thread is unblocked and we return from one of our
   "wait" methods with data about what event occurred.
6. We call `read` on the socket we created in 2.

## Completion based event queue

IOCP stands for I/O Completion Ports, and in this type of queue you get a
notification when events are completed. For example data is read to a buffer.

This is the basics of what happens in an even queue:

1. We create an event queue by calling the syscall `CreateIoCompletionPort`
2. We create a buffer and a get a handle to a socket
3. We register an interest in `Read` events on this socket with another syscall,
   but this time we also pass inn the buffer we created to which the data will
   be read.
4. Next, we call `GetQueuedCompletionStatusEx` which will block until an event has
   completed
5. Our thread is unblocked and our buffer is filled with the data we're interested in


## Epoll 

`Epoll` is the Linux way of implementing an event queue. In terms of functionality it has a lot of 
common with `Kqueue`. On a high level these abstractions provide us with this functionality:

1. A handle to an event queue
2. A way for us to register interest for events on a file descriptor and place it in this queue
3. A way for us to wait for this event to occur by letting the OS suspend our thread and wake us up when event is ready

### Kqueue

`Kqueue` is the Macos way of implementing an event queue, well, actually it's 
the BSD way of doing this that Macos uses. In terms of high level functionality
it's similar to `Epoll` in concept but different to use.

Some argue it's a bit more complex to use and a bit more abstract and "general".

### IOCP

`IOCP` or Input Output Completion Ports is the way Windows handles this type of event queue. 

This means it will let you know when an event has `Completed`. Now this might
sound like a minor difference but it's not, especially when you want to write a
library since abstracting over both means you'll either have to model `IOCP` as
`readiness based` or model `epoll/kqueue` as completion based.

Lending out a buffer to the OS also provides some challenges since it's very
important that this buffer stays untouched while waiting for an operation to
return.

> My experience investigating this suggests that getting the `readiness based`
> models to behave like the `completion based` model is easier than the other
> way around. This means get IOCP to work first and then fit `epoll` and `kqueue`
> into that design. I'm open to other views as well.

