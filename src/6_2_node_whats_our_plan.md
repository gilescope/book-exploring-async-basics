# What's our plan

For our plan to work we need a runtime to run our "javascript".

Let's first start with our "javascript":

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
    print("Registering immediate timeout 3");
    set_timeout(0, |_res| {
        print("Immediate3 timed out");
    });
    // let's read the file again and display the text
    print("Second call to read test.txt");
    Fs::read("test.txt", |result| {
        let text = result.into_string().unwrap();
        let len = text.len();
        print(format!("Second count: {} characters.", len));

        // aaand one more time but not in parallell.
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

fn print(t: impl std::fmt::Display) {
    println!("Thread: {}\t {}", current(), t);
}

fn print_content(t: impl std::fmt::Display, descr: &str) {
    println!(
        "\n===== THREAD {} START CONTENT - {} =====",
        current(),
        descr.to_uppercase()
    );
    println!("{}", t);
    println!("===== END CONTENT =====\n");
}
```

Next, let's feed this code into our runtime:

```rust
fn main() {
    let mut rt = Runtime::new();
    rt.run(javascript);
}
```

Now before we go on to implement our runtime we'll need some helper functions:

```rust
fn print(t: impl std::fmt::Display) {
    println!("Thread: {}\t {}", current(), t);
}

fn print_content(t: impl std::fmt::Display, descr: &str) {
    println!(
        "\n===== THREAD {} START CONTENT - {} =====",
        current(),
        descr.to_uppercase()
    );
    println!("{}", t);
    println!("===== END CONTENT =====\n");
}

fn current() -> String {
    thread::current().name().unwrap().to_string()
}

```

Here we define three functions:

`print` which prints out a message that first tells us what thread the message is beeing outputted from, and then a message we provide:

`print_content` does the same as `print` but is a way for us to print out more than a message in a nice way.

`current` is just a shortcut for us to get the name of the current thread. Since we want to track what's happening where we're going to need to print out what thread is issuing what output so this will avoid cluttering up our code too much along the way.


Next we pull in some modules from the standard library and we refer to an external library called `minimio`

```rust
use std::collections::{BTreeMap, HashMap};
use std::fmt;
use std::fs;
use std::io::{Read, Write};
use std::sync::mpsc::{channel, Receiver, Sender};
use std::thread::{self, JoinHandle};
use std::time::{Duration, Instant};

use minimio;
```

## Minimio

Minimio is a cross platform epoll/kqueue/IOCP based event loop that we will cover in the next book. I originally included it here but implementing that for three arcitectures is pretty interesting and needed more space than would fit in this book.

Most modern I/O eventloops uses a cross platform library like this. In Rust we have [`mio`](https://github.com/tokio-rs/mio), Node uses [`libuv`](https://github.com/libuv/libuv) and there are several more. However, creating a cross platform general eventloop is pretty challenging since Windows and Unix has different ways of handeling events. We'll talk much more about this in the next book but let's just make a note of it here.

Let's briefly cover the how this works and why we need this in Node:

### Epoll

`Epoll` is the Linux way of implementing an event queue. In terms of functionality it has a lot of common with `Kqueue`. On a high level these abstractions provide us with this functionality:

1. A handle to an event queue
2. A way for us to register interest for events on a file descriptor and place it in this queue
3. A way for us to wait for this event to occur by letting the OS suspend our thread and wake us up when event is ready

### Kqueue

`Kqueue` is the Macos way of implementing an event queue, well, actually it's the BSD way of doint this that Macos uses. In terms of high level functionality it's similar to `Epoll`.

The differences are in how you interact with the queues and there are some differences in functionality but for the normal use case they are similar enough.

### IOCP

`IOCP` or Input Output Completion Ports is the way Windows handles this type of event queue. This type of queue works differently from `epoll` and `kqueue`. The biggest difference in terms of functionality is that `epoll` and `kqueue` lets you know when an event is ready (i.e. some data is ready to be read). We call this form of model a `readiness based` model.

Windows on the other hand uses a `completion based` model. This means it will let you know when an event has `Completed`. Now this might sound like a minor difference but it's not, especially when you want to write a library.

The major difficulty is that you either need to get `epoll` or `kqueue` to behave like they're `completion based` or you'll have to try to get Windows to behave in a `readiness based` manner. The latter is the way `wepoll` and `mio` does it but this change is very recent and uses a few undocumented parts of the Windows API.

