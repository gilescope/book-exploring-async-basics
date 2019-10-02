# What is Node?

We have to start with a short explanation of what Node is, just so we're on the same page.

Node is a Javascript runtime allowing Javascript to run on you'r desktop (or server). Javascript was originally designed as a scripting language for the browser which also means that Javascript in itself needs some runtime to interpret it.

Javascript has one advantage from a language design perspective: Everything is designed to be handled asynchronously. An as you know by now, this pretty crucial if we want to make the most out of our hardware especially if you have a lot of I/O operations to take care of.

One such scenario is a Webserver. Webservers handle a lot of I/O tasks whether it's reading from the file system or communicating via the network card.

## Why Node

- Javascript is unavoidable when doing web development. Using Javascript on the server allowed programmers to use the same language both places.
- There is a potential for code reuse between the server and the front end
- The design of Node allows it to make very perfromant web servers
- Working with Json and APIs is very easy when you only deal with Javascript

## Myths and helpfull facts

Let's start off by debunking some myths that might make it easier to follow along when we start to code.

### The Javascript Eventloop

Javascript is a scripting languange and can't do much on it's own. It doesn't hava an event loop. Now in a web browser, the browser provides a runtime, which includes an event loop. And on the server, Node provides this functinality. You might say that Javascript as a language would be difficult to run (due to it's callback based model) without some sort of event loop, that's beside the point.

### Node isn't multithreaded

This isn't true. But the part of Node that "progresses" your code, does indeed run on a single thread. When we say "don't block the event loop" we're referring to this thread since that will prevent Node to progress other tasks.

We'll see exactly why blocking this thread is a problem and how that's
  handled.

### The V8 javascript engine

Now, this is where we need to focus a bit. The V8 engine is a javascript JIT compiler. That means that when you write a `for` loop, the V8 engine translates this to instructions that run on you'r CPU. There are many javascript engines, but Node was originally implemented on top of the V8 engine.

The V8 engine itself can't do much useful for us, it just interprets our Javascript. It can't do I/O, set up a runtime or anything like that. Writing Javascript only with V8 will be a very limited experience.

> Since we write Rust as you saw on the top, we'll not cover the part of translating javascript. We'll just focus on how Node works and handles concurrency since that's our main focus right now.

## Nodes eventloop(s)

Picturing Node as a single eventloop is a standard simple event loop is an oversimplification, but
it's what we'll do in our example.

Node internally divides it's real work into two categories:

### I/O bound tasks

Are handleded by the cross platform epoll/kqueue/IOCP event queue implemented in `libuv` and in our case `minimio`.

### CPU bound tasks

Are handeled by an threadpool. The default size of this threadpool is 4 threads, but that can be configured by the Node runtime.

I/O tasks which can't be handled by the cross platform eventqueue is also handled here which is the case with file reads which we use in our example.

Most C++ extensions for Node uses this threadpool to perform their work and that is one of many reasons they are used for CPU heavy tasks.

## Further information

If you do want to know more about the Node eventloop I have one short page of the `libuv` documentation I can
refer you to and two talks for you that I find great (and correct) on this subject:

[Libuv Design Overview](http://docs.libuv.org/en/v1.x/design.html#design-overview)

This first talk one is made held by [@piscisaureus](https://github.com/piscisaureus) and is an excellent 15 minute overview - i especially recommend this one as its short and to the point:
<iframe width="560" height="315" src="https://www.youtube.com/embed/PNa9OMajw9w" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


The second one is slightly longer but is also an excellent talk held by [Bryan Hughes](https://github.com/nebrius)
<iframe width="560" height="315" src="https://www.youtube.com/embed/zphcsoSJMvM" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


Now, relax, get a cup of tea and sit back while we go through everything together.

