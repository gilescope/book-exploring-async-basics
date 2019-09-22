# Implementing the Node Eventloop

Now we've finally come to the part of this book where we will write som more code.

As an example we'll implement a toy version of the Node eventloop.

We want to write something like this:

```rust
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

    // aaand one more time but not concurrently
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
```

And since this is just a runtime we write to learn we get this output showing us what happens when we run that code:

```
Thread: main     First call to read test.txt
Thread: main     Registering immediate timeout 1
Thread: pool3    recived a task of type: File read
Thread: main     Registered timer event id: 2
Thread: main     Registering immediate timeout 2
Thread: main     Registered timer event id: 3
Thread: main     Registering immediate timeout 3
Thread: main     Registered timer event id: 4
Thread: main     Second call to read test.txt
Thread: main     Registering a 3000 ms timeout
Thread: pool2    recived a task of type: File read
Thread: main     Registered timer event id: 6
Thread: main     Registering a 1000 ms timeout
Thread: main     Registered timer event id: 7
Thread: main     Registering http get request to google.com
Thread: main     Event with id: 8 registered.
Thread: main     ===== TICK 1 =====
Thread: main     Immediate1 timed out
Thread: main     Immediate2 timed out
Thread: main     Immediate3 timed out
Thread: main     ===== TICK 489 =====
Thread: main     SETTIMEOUT
Thread: pool2    finished running a task of type: File read.
Thread: pool3    finished running a task of type: File read.
Thread: main     ===== TICK 1057 =====
Thread: main     First count: 39 characters.
Thread: main     I want to create a "magic" number based on the text.
Thread: main     Second count: 39 characters.
Thread: main     Third call to read test.txt
Thread: pool3    recived a task of type: Encrypt
Thread: pool2    recived a task of type: File read
Thread: epoll    epoll event 8 is ready
Thread: main     ===== TICK 1124 =====

===== THREAD main START CONTENT - WEB CALL =====
HTTP/1.1 302 Found
Server: CowbHTTP/1.1 302 Found
Server: CowbHTTP/1.1 302 Found
Server: Cowboy
Date: Sun, 22 Sep 2019 15:17HTTP/1.1 302 Found
Server: Cowboy
Date: Sun, 22 Sep 2019 15:17:52 GMT
Connection: close
Content-Type: text/html;charset=utf-HTTP/1.1 302 Found
Server: Cowboy
Date: Sun, 22 Sep 2019 15:17:52 GMT
Connection: close
Content-Type: text/html;charset=utf-8
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: OPTIONS
Access-Control-Allow-Methods: GET
Access-Control-AlloHTTP/1.1 302 Found
Server: Cowboy
Date: Sun, 22 Sep 2019 15:17:52 GMT
Connection: close
Content-Type: text/html;charset=utf-8
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: OPTIONS
Access-Control-Allow-Methods: GET
Access-Control-Allow-Methods: POST
Location: http://http/www.google.com
X-Xss-Protection: 1; mode=block
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Content-Length: 0
Via: 1.1 vegur


===== END CONTENT =====

Thread: pool3    finished running a task of type: Encrypt.
Thread: main     ===== TICK 1330 =====
Thread: main     "Encrypted" number is: 63245986
Thread: main     ===== TICK 1597 =====
Thread: main     3000ms timer timed out
Thread: main     Registered timer event id: 11
Thread: main     ===== TICK 1882 =====
Thread: main     500ms timer(nested) timed out
Thread: pool2    finished running a task of type: File read.
Thread: main     ===== TICK 2171 =====

===== THREAD main START CONTENT - FILE READ =====
Hello world! This is a text to encrypt!
===== END CONTENT =====

Thread: main     FINISHED
```

In the following chapters I'll walk you through what's going on here but right from the start we can notice some important things going on here:

The code we write has a lot of simiarities with Javascript and looks very little like idiomatic Rust code.

When you look through the events you'll see they happen in a different order that we wrote them. That tells us that there is some concurrency going on here.

The next chapters will explain everything going on here, so relax, get a cup of tea and stay focused as we explain everything.