# Implementing the Node Eventloop

Now we've finally come to the part of this book where we will write som more code.

The Node event loop is a complex piece of software developed over many years. We
will have to simplify things a lot. 

We'll try to implement the parts that are important for us to understand Node a
little better and most importantly use it as an example where we can use our
knowledge from the previous chapters to make something that actually works.

Our main goal here is to explore async concepts, using Node as an example is
mostly for fun.

We want to write something like this:

```rust
/// Think of this function as the javascript program you have written
fn javascript() {
    print("First call to read test.txt");
    Fs::read("test.txt", |result| {
        let text = result.into_string().unwrap();
        let len = text.len();
        print(format!("First count: {} characters.", len));

        print(r#"I want to create a "magic" number based on the text."#);
        Crypto::encrypt(text.len(), |result| {
            let n = result.into_int().unwrap();
            print(format!(r#""Encrypted" number is: {}"#, n));
        })
    });

    print("Registering immediate timeout 1");
    set_timeout(0, |_res| {
        print("Immediate1 timed out");
    });
    print("Registering immediate timeout 2");
    set_timeout(0, |_res| {
        print("Immediate2 timed out");
    });

    // let's read the file again and display the text
    print("Second call to read test.txt");
    Fs::read("test.txt", |result| {
        let text = result.into_string().unwrap();
        let len = text.len();
        print(format!("Second count: {} characters.", len));

        // aaand one more time but not in parallel.
        print("Third call to read test.txt");
        Fs::read("test.txt", |result| {
            let text = result.into_string().unwrap();
            print_content(&text, "file read");
        });
    });

    print("Registering a 3000 and a 500 ms timeout");
    set_timeout(3000, |_res| {
        print("3000ms timer timed out");
        set_timeout(500, |_res| {
            print("500ms timer(nested) timed out");
        });
    });

    print("Registering a 1000 ms timeout");
    set_timeout(1000, |_res| {
        print("SETTIMEOUT");
    });

    // `http_get_slow` let's us define a latency we want to simulate
    print("Registering http get request to google.com");
    Io::http_get_slow("http//www.google.com", 2000, |result| {
        let result = result.into_string().unwrap();
        print_content(result.trim(), "web call");
    });
}

fn main() {
    let rt = Runtime::new();
    rt.run(javascript);
}
```

When running this we'll have print statements at strategic places in our code
so we can get a view on what's actually happening.

Right off the bat you'll see that something is strange with the Rust code we
have written in our example.

The code uses callbacks when we have an async operation we want to execute just
like Javascript does, and we have "magic" modules like `Fs` or `Crypto` that
we can call, just like when you import modules from Node.

Our code here is mostly calling functions that register an event, and stores a
callback to be run when the event is ready.

```rust
set_timeout(0, |_res| {
    print("Immediate1 timed out");
});
```

What happens here is that we register interest in a `timeout` event. And we register
the callback `|_res| { print("Immediate1 timed out"); }`. Now the parameter `_res` is
an argument that is passed in to our callback. In javascript it would be left out, but
since we use a typed language we have created a type called `Js`.

`Js` is an enum that represents Javascript types. In the case of `set_timeout` it's
`Js::undefined`. In the case of `Fs::read` it's an `Js::String` and so on.

T callback is given an **unique Id** and is stored until the event occurs and
we invoke the callback and pass in any arguments we might have. In the case of `Fs::read`
that would be the text representation of the file we read.


When we run this code we'll get something looking like this:

![example_run](./images/example_run.mp4)

And here is what our output will look like:
```
Thread: main	 First call to read test.txt
Thread: main	 Registering immediate timeout 1
Thread: main	 Registered timer event id: 2
Thread: main	 Registering immediate timeout 2
Thread: main	 Registered timer event id: 3
Thread: main	 Second call to read test.txt
Thread: main	 Registering a 3000 and a 500 ms timeout
Thread: main	 Registered timer event id: 5
Thread: main	 Registering a 1000 ms timeout
Thread: main	 Registered timer event id: 6
Thread: main	 Registering http get request to google.com
Thread: pool3	 recived a task of type: File read
Thread: pool2	 recived a task of type: File read
Thread: main	 Event with id: 7 registered.
Thread: main	 ===== TICK 1 =====
Thread: main	 Immediate1 timed out
Thread: main	 Immediate2 timed out
Thread: pool3	 finished running a task of type: File read.
Thread: pool2	 finished running a task of type: File read.
Thread: main	 First count: 39 characters.
Thread: main	 I want to create a "magic" number based on the text.
Thread: pool3	 recived a task of type: Encrypt
Thread: main	 ===== TICK 2 =====
Thread: main	 SETTIMEOUT
Thread: main	 Second count: 39 characters.
Thread: main	 Third call to read test.txt
Thread: main	 ===== TICK 3 =====
Thread: pool2	 recived a task of type: File read
Thread: pool3	 finished running a task of type: Encrypt.
Thread: main	 "Encrypted" number is: 63245986
Thread: main	 ===== TICK 4 =====
Thread: pool2	 finished running a task of type: File read.

===== THREAD main START CONTENT - FILE READ =====
Hello world! This is a text to encrypt!
... [Note: Abbreviated for display] ...
===== END CONTENT =====

Thread: main	 ===== TICK 5 =====
Thread: epoll	 epoll event 7 is ready

===== THREAD main START CONTENT - WEB CALL =====
HTTP/1.1 302 Found
Server: Cowboy
Location: http://http/www.google.com
... [Note: Abbreviated for display] ...
===== END CONTENT =====

Thread: main	 ===== TICK 6 =====
Thread: epoll	 epoll event timeout is ready
Thread: main	 ===== TICK 7 =====
Thread: main	 3000ms timer timed out
Thread: main	 Registered timer event id: 10
Thread: epoll	 epoll event timeout is ready
Thread: main	 ===== TICK 8 =====
Thread: main	 500ms timer(nested) timed out
Thread: pool0	 recived a task of type: Close
Thread: pool1	 recived a task of type: Close
Thread: pool2	 recived a task of type: Close
Thread: pool3	 recived a task of type: Close
Thread: epoll	 recieved event of type: Close
Thread: main	 FINISHED
```

Don't worry, we'll explain everything, but I just wanted to start off by
explaining where we want to end up.


