# Setting up our runtime

## The Threadpool

We still don't have a threadpool or a I/O eventloop running
so the next step is to set this up so we can start focusing on how to work with
our "Node" Runtime.

### Let's take this step by step

The first thing we do is to add a `new` method that returns an instance of our
`Runtime`:

```rust
impl Runtime {
    pub fn new() -> Self {
```
Now the real work starts. First up is our thread pool. The first thing we do is
to set up a channel which our threads can use to send messages to our main thread.

The channel will take a tuple `(usize, usize, Js)` which will be `thread_id`,
`callback_id` and the data returned when we run our `Task`.

The `Receiver` part will be stored in our `Runtime`, and the `Sender` part will
be cloned to each of our threads.

Node defaults to 4 threads which we will copy. This is configurable in `Node` 
but we will take a shortcut and hard code it:

```rust
let (event_sender, event_reciever) = channel::<PollEvent>();
let mut threads = Vec::with_capacity(4);

for i in 0..4 {
    let (evt_sender, evt_reciever) = channel::<Task>();
    let event_sender = event_sender.clone();

    let handle = thread::Builder::new()
        .name(format!("pool{}", i))
        .spawn(move || {

            while let Ok(task) = evt_reciever.recv() {
                print(format!("recived a task of type: {}", task.kind));
                
                if let ThreadPoolTaskKind::Close = task.kind {
                    break;
                };

                let res = (task.task)();
                print(format!("finished running a task of type: {}.", task.kind));

                let event = PollEvent::Threadpool((i, task.callback_id, res));
                event_sender.send(event).expect("threadpool");
            }
        })
        .expect("Couldn't initialize thread pool.");

    let node_thread = NodeThread {
        handle,
        sender: evt_sender,
    };

    threads.push(node_thread);
}

```

Next up is actually creating our threads. `for i in 0..4` is an iterator over the
values 0, 1, 2 and 3. Since we push each thread to a `Vec` these values will be
treated as both the Id of the thread and the index it has in our `Vec`.

Next up we create a new channel which we will use to send messages **to** our
threads. Each thread keeps their `Receiver`, and we'll store the `Send` part
in the struct `NodeThread` which will represent a thread in our threadpool.

```rust
let (evt_sender, evt_reciever) = channel::<Event>();
let threadp_sender = threadp_sender.clone();
```
As you see here, we also clone the `Sender` part which we'll pass on to each thread
so they can send messages to our `main` thread.

After that's done we build our thread. We'll use `thread::Builder::new()` instead
of use `thread::spawn` since we want to give each thread a name. We'll only use this
name when we `print` from our event since it will be clear from which thread
we printed the message. 

```rust
let handle = thread::Builder::new()
        .name(format!("pool{}", i))
        .spawn(move || {
```
You'll also see here that we `spawn` our thread finally and create a closure.

> Why do we need the `move` keyword in this closure?
>
> The reason is that this closure is spawned from the main thread, so any environment
> we close over needs to be owned, since it can't reference any values on the stack
> of the `main` thread. I'll leave you with a relevant quote from [chapter about
> `closures` in TRPL](https://doc.rust-lang.org/1.30.0/book/first-edition/closures.html#closures)
>
> __...they give a closure its own stack frame. Without move, a closure may be
> tied to the stack frame that created it, while a move closure is self-contained.__


The body of our new threads are really simple, most of the lines are about printing
out information for us to see:

```rust
while let Ok(task) = evt_reciever.recv() {
        print(format!("recived a task of type: {}", task.kind));
        
        if let ThreadPoolTaskKind::Close = task.kind {
            break;
        };

        let res = (task.task)();
        print(format!("finished running a task of type: {}.", task.kind));

        let event = PollEvent::Threadpool((i, task.callback_id, res));
        event_sender.send(event).expect("threadpool");
    }
})
```

The first thing we do is to listen on our `Recieve` part of the channel (remember,
we gave the `Send` part to our `main` thread). This function will actually
`park` our thread until we receive a message so it consumes no resources while
waiting.

When we get a `task` we first print out what kind of task we got. 

The next thing we do is to check if this was a `Close` task, if thats true
we break out of our loop which in turn will close the thread.

If it wasn't a `Close` task we run our task `let res = (task.task)();`. This is where the work will
actually be done. We know from the signature of this task that it returns a `Js`
object once it's finished.

The nest thing we do is to print out that we finished running a task, before we
send a `PollEvent::Threadpool` event with `thread_id`, the `callback_id` and the data returned as a `Js` object back
to our main thread.

Back in our `main` thread again we'll finally we store the `JoinHandle`, and the
`Send` part of the channel in our `NodeThread` struct and push it to our
collection of threads (which now represents our threadpool).

## The Epoll Thread

This will handle our Epoll/Kqueue/IOCP thread. This thread will only wait for
incoming events reported by the OS, and once that's done it will send the Id of
the event to our main thread which in turn will actually handle the event and call
the callback.

The code here is a bit more involved, but we'll take it step by step below.

The code looks like this:

```rust
let mut poll = minimio::Poll::new().expect("Error creating epoll queue");
let registrator = poll.registrator();
let epoll_timeout = Arc::new(Mutex::new(None));
let epoll_timeout_clone = epoll_timeout.clone();

let epoll_thread = thread::Builder::new()
    .name("epoll".to_string())
    .spawn(move || {
        let mut events = minimio::Events::with_capacity(1024);
        
        loop {
            let epoll_timeout_handle = epoll_timeout_clone.lock().unwrap();
            let timeout = *epoll_timeout_handle;
            drop(epoll_timeout_handle);

            match poll.poll(&mut events, timeout) {
                Ok(v) if v > 0 => {
                    for i in 0..v {
                        let event = events.get_mut(i).expect("No events in event list.");
                        print(format!("epoll event {} is ready", event.id().value()));
                        
                        let event = PollEvent::Epoll(event.id().value() as usize);
                        event_sender.send(event).expect("epoll event");
                    }
                }
                Ok(v) if v == 0 => {
                    print("epoll event timeout is ready");
                    event_sender.send(PollEvent::Timeout).expect("epoll timeout");
                }
                Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {
                    print("recieved event of type: Close");
                    break;
                }
                Err(e) => panic!("{:?}", e),
                _ => (),
            }
        }
    })
    .expect("Error creating epoll thread");
```

Lets start by initializing some variables:

```rust
let mut poll = minimio::Poll::new().expect("Error creating epoll queue");
let registrator = poll.registrator();
let epoll_timeout = Arc::new(Mutex::new(None));
let epoll_timeout_clone = epoll_timeout.clone();
```

The first thing we do is to instantiate a new `minimio::Poll`. This is the main
entry point into our `kqueue/epoll/iocp` event queue.

> `minimio::Poll` does several things under the hood. It sets up a structure for
> us to store some information about the state of the event queue, and most
> importantly makes a syscall to the underlying OS and asks it for a handle to 
> either an `epoll` instance, a `kqueue` or to an `Completion Port`. We won't
> register anything here yet, but we need this handle to later make sure we register
> interest with the queue we're polling.

Next up is also part of `minimio` we get a `Registrator`. This struct is "detached"
from the `Poll` struct, but it holds a cpoy of the same handle to the event queue.

This way we kan store the `Registrator` in our main thread and send off the `Poll`
instance to our `epoll` thread. Our registrator can only register events to the
queue and that's it.

> How can `Registrator` know that the `epoll` thread hasn't stopped?
>
> We'll cover this in detail in the next book, but both `Poll` and `Registrator`
> holds a reference to an `AtomicBool` which only job is to indicate if the queue
> is "alive" or not. In the `Drop` implemenation of `Poll` we set this flag to
> false in which case a coll to register an event will return an `Err`.

`epoll_timeout` is the time to the next timeout. If there is no more timeouts the
value is `None`. We wrap this in a `Arc<Mutex<>>`, since we'll be writing to
this from the main thread, and reading from it in the `epoll` thread.

`epoll_timeout_clone` is just increasing the ref-count on our `Arc` so that
we can send this to our `epoll` thread.

Next up is spawning our thread. We do this the exact same way as for the thread
pool, but we name the thread `epoll`.
```rust
let epoll_thread = thread::Builder::new()
    .name("epoll".to_string())
    .spawn(move || {
```

Now we're inside the `epoll` thread and will define what this thread needs to
do to poll and handle events.

First we allocate a buffer to hold event objects that we get from our `poll` instance.
These objects contain information about the event that's occurred including a
`token` we pass in when we register the event. This `token` identifies what event
has occurred. In our case the token is a simple `usize`.

```rust
let mut events = minimio::Events::with_capacity(1024);
```
We allocate the buffer here since we only allocate this once when we do it here,
and we want to avoid allocating a new buffer on every turn of our `loop`.

Basically, our `epoll` thread will run a loop which consciously polls for new
events.

```rust
loop {
```

The interesting logic is inside the loop, and first we read the timeout value which
should be synced with the next timeout that expires in our main loop.
```rust
let epoll_timeout_handle = epoll_timeout_clone.lock().unwrap();
let timeout = *epoll_timeout_handle;
drop(epoll_timeout_handle);
```
To do this we first need to `lock` the mutex so we know we have exclusive access
to the `timeout` value. Now, `timeout` is of the type `Option<i32>`, since `i32`
implements the `Copy` trait we can dereference it, which in this case will copy,
the value and store it in our `timeout` variable.

`drop(epoll_timeout_handle)` is not something you'll see often. The `MutexGuard` we
get in return when we call `epoll_timeout_clone.lock().unwrap()` is a [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization)
guard. Which means that it will hold a resource (in this case the lock on the mutex)
until it's deallocated(released). In Rust, the release happens when the value is
`Dropped` which normally is by the end of a scope (`{...}`).

We need to release the lock since the next call will block until an event occurs
which means our lock wouldn't have been released and we would end up in a `deadlock`
when trying to write a value to our `epoll_timeout` in our main thread.

The next part is a handful, but bear in mind that much of what we do here is
printing out information for us to observe.

Calling `poll` will block the loop until an event occurs **or** the timeout has
elapsed. `poll` takes in an exclusive reference to our event buffer, and an `Option<i32>`
as a timeout. A value of `None` will block indefinitely.

> When we say `block` here we mean that the OS parks our thread, and switches
> context to another thread. However, it keeps track over that our `epoll` thread
> listens to events and wakes it up again when any of the events we have registered
> interests to has happened.

```rust
match poll.poll(&mut events, timeout) {
    Ok(v) if v > 0 => {
        for i in 0..v {
            let event = events.get_mut(i).expect("No events in event list.");
            print(format!("epoll event {} is ready", event.id().value()));
            
            let event = PollEvent::Epoll(event.id().value() as usize);
            event_sender.send(event).expect("epoll event");
        }
    }
    Ok(v) if v == 0 => {
        print("epoll event timeout is ready");
        event_sender.send(PollEvent::Timeout).expect("epoll timeout");
    }
    Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {
        print("recieved event of type: Close");
        break;
    }
    Err(e) => panic!("{:?}", e),
    _ => unreachable!(),
}
```

We `match` on the result of the `poll` so when the OS returns we choose what to do.

We basically have 4 cases we are concerned about:

1. We get an Ok(n) where n is larger than 0, in this case we have events to process
2. We get an Ok(n) where n is 0, we know this either is a `spurious` wakeup or that a timeout has occurred
3. We get an Err of kind `Interrupted`, in which case we treat this as a close signal and we close the loop
4. We get an Err which is not of type `Interrupted`, we know something bad has happened, and we `panic!`

If you haven't seen the syntax `Ok(v) if v > 0` before it's what we call a ` match guard`
which lets us refine what we're matching against. In this case, we only match on
values of `v` larger than `0`.

For completeness I'll also explain `Err(ref e) if e.kind()`, the `ref` keyword
tells the compiler that we want a reference to `e` and don't want to take
ownership over it.

The last case `_ => unreachable!()` is just needed since the compiler doesn't realize that
we're matching on all values of `Ok()` here. The value is of type `Ok(usize)` so
it can't be negative, and we're telling the compiler here that we've got all
cases covered.


Lastly we create a `Runtime` struct and store all the data we've intialized so
far into it:

```rust
  Runtime {
    available_threads: (0..4).collect(),
    callbacks_to_run: vec![],
    callback_queue: HashMap::new(),
    epoll_pending_events: 0,
    epoll_registrator: registrator,
    epoll_thread,
    epoll_timeout,
    event_reciever,
    identity_token: 0,
    pending_events: 0,
    thread_pool: threads,
    timers: BTreeMap::new(),
    timers_to_remove: vec![],
}
```

Worth noting is that we know all threads are available here so `(0..4).collect()`
will just create a `Vec<usize>` with the values `[0, 1, 2, 3]`.

In Rust, when we write:
```rust
...
epoll_registrator: registrator,
epoll_thread,
...
```

We're in this case assigning `registrator` to `epoll_registrator` which is a
bit more descriptive. But since we have a variable with the name `epoll_thread`
already we don't need to write `epoll_thread: epoll_thread` since the compiler
figures that out for us.

Now the final initialization code for our runtime breaks all "best practices" of
how long methods you should have but for our case I find it easier to write about
this if we don't need to jump between functions too much and can just cover all this
logic from a-z:

```rust
impl Runtime {
    pub fn new() -> Self {
        // ===== THE REGULAR THREADPOOL =====
        let (event_sender, event_reciever) = channel::<PollEvent>();
        let mut threads = Vec::with_capacity(4);

        for i in 0..4 {
            let (evt_sender, evt_reciever) = channel::<Task>();
            let event_sender = event_sender.clone();

            let handle = thread::Builder::new()
                .name(format!("pool{}", i))
                .spawn(move || {

                    while let Ok(task) = evt_reciever.recv() {
                        print(format!("recived a task of type: {}", task.kind));
                        
                        if let ThreadPoolTaskKind::Close = task.kind {
                            break;
                        };

                        let res = (task.task)();
                        print(format!("finished running a task of type: {}.", task.kind));

                        let event = PollEvent::Threadpool((i, task.callback_id, res));
                        event_sender.send(event).expect("threadpool");
                    }
                })
                .expect("Couldn't initialize thread pool.");

            let node_thread = NodeThread {
                handle,
                sender: evt_sender,
            };

            threads.push(node_thread);
        }

        // ===== EPOLL THREAD =====
        let mut poll = minimio::Poll::new().expect("Error creating epoll queue");
        let registrator = poll.registrator();
        let epoll_timeout = Arc::new(Mutex::new(None));
        let epoll_timeout_clone = epoll_timeout.clone();

        let epoll_thread = thread::Builder::new()
            .name("epoll".to_string())
            .spawn(move || {
                let mut events = minimio::Events::with_capacity(1024);
                
                loop {
                    let epoll_timeout_handle = epoll_timeout_clone.lock().unwrap();
                    let timeout = *epoll_timeout_handle;
                    drop(epoll_timeout_handle);

                    match poll.poll(&mut events, timeout) {
                        Ok(v) if v > 0 => {
                            for i in 0..v {
                                let event = events.get_mut(i).expect("No events in event list.");
                                print(format!("epoll event {} is ready", event.id().value()));
                                
                                let event = PollEvent::Epoll(event.id().value() as usize);
                                event_sender.send(event).expect("epoll event");
                            }
                        }
                        Ok(v) if v == 0 => {
                            print("epoll event timeout is ready");
                            event_sender.send(PollEvent::Timeout).expect("epoll timeout");
                        }
                        Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {
                            print("recieved event of type: Close");
                            break;
                        }
                        Err(e) => panic!("{:?}", e),
                        _ => unreachable!(),
                    }
                }
            })
            .expect("Error creating epoll thread");

        Runtime {
            available_threads: (0..4).collect(),
            callbacks_to_run: vec![],
            callback_queue: HashMap::new(),
            epoll_pending_events: 0,
            epoll_registrator: registrator,
            epoll_thread,
            epoll_timeout,
            event_reciever,
            identity_token: 0,
            pending_events: 0,
            thread_pool: threads,
            timers: BTreeMap::new(),
            timers_to_remove: vec![],
        }
    }
} 
```