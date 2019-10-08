# What's our plan

I'll briefly list what we need to do to get this working here:

## We need two event queues:

1. We need a threadpool to execute our CPU intensive tasks, or tasks that we want
too run asynchronously but not in our OS backed event queue
2. We need to make a simple cross platform `epoll/kqueue/IOCP` eventloop. Now
this turns out to be extremely interesting, but also a lot of code so I split
that off to a separate "companion book" for those that want to explore this further.
We use this library here (called `minimio`). 

> Granted, we're breaking our rule that we don't use any dependencies, but it's our own dependency which we'll explain fully in due time.

## We need a runtime

**Our runtime will:**

1. Store our callbacks to be run in a later point
2. Send tasks to be executed on the thread pool
3. Register interests with the OS (through `minimio`)
4. Poll our two event sources for new events
5. Handle timers
6. Provide a way for "modules" like `Fs` and `Crypto` to register tasks
7. Progress all our tasks until we're finished

## We need a few modules

1. For handling file system tasks `Fs`
2. For handling http calls `Http`
3. For handling cryptological tasks `Crypto`

## We need to make som helpers

We need some helpers to make our code readable and to provide the output we want
to see. In contrast to any real runtime, we're interested in knowing what happens
when. To help with that we define three extra methods:

`print` which prints out a message that first tells us what thread the message is being outputted from, and then a message we provide:

`print_content` does the same as `print` but is a way for us to print out more than a message in a nice way.

`current` is just a shortcut for us to get the name of the current thread. Since we want to track what's happening where we're going to need to print out what thread is issuing what output so this will avoid cluttering up our code too much along the way.


## Minimio

Minimio is a cross platform epoll/kqueue/IOCP based event loop that we will cover in the next book. I originally included it here but implementing that for three architectures turned out to be a bit more involved than I first thought and needed more space than would fit in this book.

It's also easier to just read up on Epoll/Kqueue/IOCP when it's concentrated in a separate book.
