# Http module

The `Http` module is probably the one I personally find interesting. There is
two reasons for this:

1. We create a Http GET request from scratch and wait for the response
2. The method `epoll_registrator.register()` is the result of a huge amount of research and work to get working. Look forward to the next book where we dive into that.

Let's look at the code first and then step through it:

```rust, no_run
struct Http;
impl Http {
    pub fn http_get_slow(url: &str, delay_ms: u32, cb: impl Fn(Js) + 'static + Clone) {
        let rt: &mut Runtime = unsafe { &mut *RUNTIME };
        // Don't worry, http://slowwly.robertomurray.co.uk is a site for simulating a delayed
        // response from a server. Perfect for our use case.
        let adr = "slowwly.robertomurray.co.uk:80";
        let mut stream = minimio::TcpStream::connect(adr).unwrap();
            
        let request = format!(
            "GET /delay/{}/url/http://{} HTTP/1.1\r\n\
             Host: slowwly.robertomurray.co.uk\r\n\
             Connection: close\r\n\
             \r\n",
            delay_ms, url
        );

        stream
            .write_all(request.as_bytes())
            .expect("Error writing to stream");

        let token = rt.generate_cb_identity();
        rt.epoll_registrator
            .register(&mut stream, token, minimio::Interests::readable())
            .unwrap();

        let wrapped = move |_n| {
            let mut stream = stream;
            let mut buffer = String::new();
            stream
                .read_to_string(&mut buffer)
                .expect("Stream read error");
            cb(Js::String(buffer));
        };

        rt.register_event_epoll(token, wrapped);
    }
}
```

First we call the method `http_get_slow` since we're simulating a slow response, again
this is for us to see and control how the events will occur since we're trying to learn.

In the function body, the first thing we do is dereference our runtime. We need to use a bit of its functionality here so we do this right away.

The next step is to create a `minimio::TcpStream`. 

```rust, no_run
let adr = "slowwly.robertomurray.co.uk:80";
let mut stream = minimio::TcpStream::connect(adr).unwrap();
```

> **Now why not the regular `TcpStream` from the standard library?**
>
> Well, you do remember that `kqueue` and `epoll` are readiness based and `IOCP` is
> completion based right? 
> 
> Well, to have single ergonomic API for all platforms
> we need to abstract over something and we (like `mio`) choose to abstract over `TcpStream`. 
>
> While `kqueue` and `epoll` can just read when a `Read` event is ready, we need the `TcpStream` to create a buffer that it hands over to the OS on Windows. So when we call `TcpStream` read, we read from this buffer on Windows.

We connect to `slowwly.robertomurray.co.uk:80` which is just a site [Robert Murray](https://github.com/rob-murray) has created
to simulate slow responses. He's kind enough to let us all use it. We can choose the delay we want on
each request.

Next we construct a http `GET` request. The line breaks, and spacing is important here
as is the two blank lines at the bottom.
```rust, no_run
let request = format!(
            "GET /delay/{}/url/http://{} HTTP/1.1\r\n\
             Host: slowwly.robertomurray.co.uk\r\n\
             Connection: close\r\n\
             \r\n",
            delay_ms, url
        );
```

You might wonder why we use `\r\n\` as line breaks here instead of the standard `\n`?

This is because a http `GET` request expects `CRLF` and not just `LF`, using only `\n` will
not result in a valid `GET` request.

We construct the request by passing in the delay we want and the address we
want to be redirected to.

Next we write this request to our `TcpStream`. In a real implementation this would
have been done by issuing the write in an async manner and then register an event
when the write has happened.

```rust, no_run
stream
    .write_all(request.as_bytes())
    .expect("Error writing to stream");
```

> We write it blocking here since by implementing both `read` and `write` we'll have
> to create much more code, and the **understanding** of how this works will not really
> benefit that much (the understanding of how a web server works would though but thats
> beyond our scope today)

Now we get to the exiting part.

First we need to generate a `token`. This token will follow our `event` and be passed
on to the OS. When the OS returns it also returns this token so we know what event
occurred.

```rust, no_run
let token = rt.generate_cb_identity();
```

For convenience we use the same token as our `callback_id` since it will be unique, and
events <-> callbacks will map 1:1 the way we have designed this.

Our next call actually issues a syscall to the underlying OS and registers our
interest in an event of type `Readable`.

```rust, no_run
rt.epoll_registrator
            .register(&mut stream, token, minimio::Interests::readable())
            .unwrap();
```

The reason we need `&mut stream` here is Windows and `IOCP`. Our stream holds a
buffer that we'll pass on to Windows. **This buffer must not be touched** while
the OS lends it exclusively. This way we leverage Rusts borrowchecker to help us
make sure of that.

> However, this has a drawback. We don't really need it to be `&mut` on `linux` and `macos`
> so on these systems we might get a compiler warning letting us know it doesn't need
> to be `&mut`. There are ways around this though, but in the interest of actually finishing
> both books I had to stop somewhere and this is not the worlds end.

The main point here is that we register an interest to read in a non-blocking manner.

I'll repeat this part of the code so you have it right in front of you while i
explain:

```rust, no_run
    let wrapped = move |_n| {
            let mut stream = stream;
            let mut buffer = String::new();
            stream
                .read_to_string(&mut buffer)
                .expect("Stream read error");
            cb(Js::String(buffer));
        };
```

Since our callback expects a `Js::String` we can't actually pass the buffer. We need
to wrap our callback so we first read out the data to a `String` which we then
can pass to our code.

Lastly we register the I/O event with our runtime

```rust
rt.register_event_epoll(token, wrapped);
```

## Bonus section

Now in a better implementation, we would not issue a blocking call like `read_to_string`
because it might be that at some point this will block if not all the data we need
is present at once. 

The right thing to do then would be to read parts of the
data into a buffer while calling `stream.read(..)` in a loop, and at some point
this might return an `Err::WouldBlock` at which we re-wrap our callback and re-register
our read event to read the rest.


> Another reason this might block is that threads might get what is called a "spurious" wakeup.
>
> What is that? Well, the OS might wake up the thread, without there being any data there. The
> reason for this is kind of hard to get confirmed, but as far as I understand, this can happen if
> the OS is in doubt if an event occurred or not. It does apparently happen that some interrupts
> might get issued without the OS getting notified. This can be caused by unfortunate timing since
> there are some bits of code that needs to be executed "uninterrupted" and there is a way for the
> OS to instruct the CPU to filter some interrupts for short periods. The OS can't be "optimistic" in the
> sense that it assumes the event didn't happen since that would cause the process to wait indefinitely
> if the event did occur. So instead it might just wake up the thread.
> 
> Also, there are performance optimizations
> in operating systems that might cause them to choose to wake up waiting threads in a process for unknown
> reasons. Therefore, accounting for spurious wakeups is part of the "contract" between the programmer
> and the OS.
> 
> Either way, the OS assumes we have logic in place to account for this and just re-register our event
> if it was a spurious wakeup.

In our implementation, whether the data is not fully available or if it was a spurious wakeup, we'll end
up blocking. It's fine, we're just trying to understand. We're not re-implementing `libuv` anyway.

The last part is registering this event with our `epoll_thread`.

Now, we're practically at the finish line. All the interesting parts are covered, we just
need a few more small things to get it all up and running.
