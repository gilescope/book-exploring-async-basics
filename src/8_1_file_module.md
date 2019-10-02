# File module

The `Fs` module contains File operations. Right now we only expose a `read` method
since that's all we need. As you can imagine we can add all sorts of methods here.

```rust
struct Fs;
impl Fs {
    fn read(path: &'static str, cb: impl Fn(Js) + 'static) {
        let work = move || {
            // Let's simulate that there is a very large file we're reading allowing us to actually
            // observe how the code is executed
            thread::sleep(std::time::Duration::from_secs(1));
            let mut buffer = String::new();
            fs::File::open(&path)
                .unwrap()
                .read_to_string(&mut buffer)
                .unwrap();
            Js::String(buffer)
        };
        let rt = unsafe { &mut *RUNTIME };
        rt.register_work(work, ThreadPoolEventKind::FileRead, cb);
    }
}
```

We simply create an empty `struct Fs;`. This is one of [Rusts Zero Sized Types (ZST)](https://doc.rust-lang.org/nomicon/exotic-sizes.html#zero-sized-types-zsts) and does not occupy any space, but it's still useful for us.

The `read` method takes a file path and a callback as arguments. The callback will get
called once the operation is finished and accepts a `Js` object as an input.

`let work = move || {...` is a closure. This closure is the actual `Task` that we
want to run on our `threadpool`. None of the code in `work` will actually run here,
we only define what we want to do when `work()` gets called.

In our closure we first wait for a second. These operations are so fast (and our file is so small)
that if we want to observe what's going on in any meaningful way we need to slow things down. Let's pretend
its a huge file that takes a second to read.

We read the file into a buffer and then return `Js::String(buffer)`.

> You might remember from the [Infrastructure chapter](./7_9_infrastructure.md) that
> our `register_work` method received a task argument `task: impl Fn() -> Js + Send + 'static`.
> As you see here, our closure returns a `Js`object and takes no arguments, which means
> it conforms to this signature. The `Fn` trait will be automatically derived. 
> `Send` is also an automatically derived trait, which means that we can't implement
> `Send`. However if we tried to send types that are `!Send` to our thread by
> referencing them in our closure we would get an error.

The last part is that we dereference our runtime and call `rt.register_work(work, ThreadPoolEventKind::FileRead, cb)`
to register the task with our `threadpool`.

## Bonus material

You might be wondering why we (and `libuv` and Rusts own `tokio`) do file operations
in the `threadpool` and not in our `epoll-event-queue`? It's I/O  isn't it?

The reason for this is actually several.

First and foremost. The OS will cache a lot of files that are frequently accessed,
and when the data is cached it will be available immediately. There is no real I/O in
such case. And it seems that most programs tend to access the same files over and over
so a cache hit will often be the case. Think of a web server for example, there's often
a very limited amount of data accessed on disk.

Now if we say that data is cached most of the time, so it's readily available, it can
be more expensive to actually register an event with the `epoll-event-queue` - get an immediate
notification that the event is `Ready` and then perform the read. It's better to just
read it in a blocking manner right away.

Even worse, in our design the file will be read on our main thread, which means that
if it's a large file it will still take some time to read it from the OS cache to
your process memory (your buffer) and that will block our entire event loop.

Better do that in the thread pool.

Secondly, the support for async file operations is limited and to a varying degree
well implemented. The only system that does this pretty good is Windows since it
uses a `completion` based model (which means it can let you know when the data is
read into your buffer). 

It makes sense for a completion based model to try to do this asynchronously, but
since the real effect are so small and the code complexity is high (especially when you're
writing a server that is cross platform) most implementations find that using a thread pool
gives good enough performance.

To sum i all up:

**Threadpool:**

- Less code complexity
- Good performance
- Very little penalty in most use cases (like web servers)
- Blocking file operations are well optimized on most platforms

**Async file I/O:**

- Increased code complexity
- Poor and limited APIs (Linux and Macos has different limitations)
- Weak platform support (does not work very well with a readiness based model)
- Little to no real gain for most use cases

You want to know more you say? Of course, [I have an article for you](https://blog.libtorrent.org/2012/10/asynchronous-disk-io/) if you want to get to know even more about this specific topic.
