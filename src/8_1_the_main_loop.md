# The main loop

Let's put our event loop logic in the `run` function of our `Runtime`. The code
which we present on this chapter is the body of this `run` function.

The `run` function on our `Runtime` will consume `self` so it's the last thing that we'll be able to call on this instance of our `Runtime`.

I'll include the whole method last so you can see it all together.

```rust, no_run
impl Runtime {
    fn run(self) {
        ...
    }
}
```

## Initialization


```rust, no_run
let rt_ptr: *mut Runtime = self;
unsafe { RUNTIME = rt_ptr as usize };
let mut timers_to_remove = vec![]; 
let mut ticks = 0; // just for us priting out

// First we run our "main" function
f();
```

The first two lines is just a `hack` we use in our code to make it "look" more
like javascript. We take the pointer to `self` and set it in the global
variable `RUNTIME`. 

We could instead pass our `runtime` around but that wouldn't
be very ergonomic. Another option would be to use `lazy_static` crate to initialize this field in a slightly safer way, but we'd have to explain what `lazy_static` do to keep our promise of minimal "magic".

To be honest, we only set this once, and it's set at the start of of our eventloop and we only access this from the same thread we created it. It might not be pretty but it's safe.

The variable `timers_to_remove` is for us to keep track of the timers we've set. `ticks` is only a counter for us to keep track of how many times we've looped which for display.

The last and least visible part of this code is actually where we kick everythin off, calling `f()`. `f` will be the code we wrote in the `javascript` function in the last chapter. If this is empty nothing will happen.

## Starting the event loop

```rust, no_run
// ===== EVENT LOOP =====
while self.pending_events > 0 {
    ticks += 1;
```

`self.pending_events` keeps track of how many pending events we have, so that when no events are left we exit the loop since our eventloop is finished.

So where does these events come from? In our `javascript` function `f` which we introduced in the chapter [Introducing our main example](./7_0_introducing_our_main_example.md) you probably noticed that we called functions like 
`set_timeout` and `Fs::read`. These functions are defined in the Node runtime 
(as they are in ours), and their main responsibility is to create tasks and register interest on events. When one of these tasks or interests are registered this counter is increased.

`ticks` is just increasing a `tick` in the counter.

## 1. Effectuate timers

`self.effectuate_timers(&mut timers_to_remove);`

This method checks if any timers has expired. If we have timers that have expired we schedule the callbacks for the expired timers to run at the first call to `self.run_callbacks()`.

Worth noting here is that timers with a timeout of `0` will already have timed out by the time we reach this function so their events will be effectuated.

## 2. Callbacks

`self.run_callbacks();`

Now we could have chosen to run the callbacks in the timer `step` but since this is the next step of our loop we do it here instead.

> This step might seem unnecessary here but in Node it has a function. Some
> types of callbacks will be deferred to be run on the next iteration of the loop, which means that they're
> not run immediately. We won't implement
> this functionality in our example but it's worth noting.

## 3. Idle/Prepare

This is a step mostly used by Nodes internals. It's not important for understanding
the big picture here but I included it since it's something you see in Nodes
documentation so you know where we're at in the loop at this point.

## 4. Poll

This is an important step. This is where we'll receive events from our thread pool or our `epoll` event queue. 

I refer to the `epoll/kqueue/IOCP` event queue as `epoll` here just so you know that it's not only `epoll` we're waiting for. From now on I will refer to the cross platform event queue as `epoll` in the code for brevity.

### Calculate time until next timeout (if any)

The first thing we do is to check if we have any timers. If we have timers that will time out we calculate how many milliseconds it is to the first timer to timeout. We'll need this to make sure we don't block and forget about our timers.

```rust, no_run
let next_timeout = self.get_next_timer();

let mut epoll_timeout_lock = self.epoll_timeout.lock().unwrap();
*epoll_timeout_lock = next_timeout;
// We release the lock before we wait in `recv`
drop(epoll_timeout_lock);
```
`self.epoll_timeout` is a `Mutex` so we need to lock it to be able to change the value it holds. Now, this is important, we need to make sure the lock is released before we `poll`. `poll` will suspend our thread, and it will try to read the value in `self.epoll_timeout`. 

If we're still holding the lock we'll end up in a `deadlock`. `drop(epoll_timeout_lock)` releases the lock. We'll explain a bit more in detail how this works in the next chapter.

### Wait for events

```rust, no_run
if let Ok(event) = self.event_reciever.recv() {
    match event {
        PollEvent::Timeout => (),
        PollEvent::Threadpool((thread_id, callback_id, data)) => {
            self.process_threadpool_events(thread_id, callback_id, data);
        }
        PollEvent::Epoll(event_id) => {
            self.process_epoll_events(event_id);
        }
    }
}
self.run_callbacks();
```

Both our `threadpool` threads and our `epoll` thread holds a `sending` part of the channel `self.event_reciever`. If either a thread in the `threadpool` finishes a task, or if the `epoll` thread receives notification that an event is ready a `PollEvent` is sent through the channel and received here.

This will block our main thread until something happens, **or** a timeout occurs.

> **Note:**
> Our `epoll` thread will read the timeout value we set in `self.epoll_timeout`, so if nothing happens
> before the timeout expires it will emit a `PollEvent::Timeout` event which simply causes our main
> event loop to continue and handle that timer.

Depending on whether it was a `PollEvent::Timeout`, `PollEvent::Threadpool` or a `PollEvent::Epoll` type of event that occurred, we handle the event accordingly.

We'll explain these methods in the following chapters.

 ## 5. Check

 ```rust
// ===== CHECK =====
// an set immediate function could be added pretty easily but we won't that here
```

Node implements a check "hook" to the eventloop next. Calls to `setImmidiate`
execute here. I just include it for for completeness but we won't do anything in this phase.


## 6. Close Callbacks

```rust
// ===== CLOSE CALLBACKS ======
// Release resources, we won't do that here, it's just another "hook" for our "extensions"
// to use. We release resources in every callback instead.
```

I pretty much explain this step in the comments. Typically releasing resources,
like closing sockets, is done here.

## Cleaning up

Since our `run` function basically will be the start and end of our `Runtime` we also need to clean up after ourselves. The following code makes sure all threads finish, release their resources and run all destructors:

```rust
// We clean up our resources, makes sure all destructors runs.
for thread in self.thread_pool.into_iter() {
    thread.sender.send(Task::close()).expect("threadpool cleanup");
    thread.handle.join().unwrap();
}

self.epoll_registrator.close_loop().unwrap();
self.epoll_thread.join().unwrap();

print("FINISHED");
```

First we loop through every thread in our `threadpool` and send a "close" `Task` to each of them. Then We call `join` on each `JoinHandle`. Calling `join` waits for the associated thread to finish so we know
all destructors are run.

Next we call `close_loop()` on our `epoll_registrator` to signal the OS event queue that we want to close the loop and release our resources. We also `join` this thread so we don't end our process until all resources are released.

## The final `run` function

```rust, no_run
pub fn run(mut self, f: impl Fn()) {
    let rt_ptr: *mut Runtime = &mut self;
    unsafe { RUNTIME = rt_ptr };

    // just for us priting out during execution
    let mut ticks = 0; 

    // First we run our "main" function
    f();

    // ===== EVENT LOOP =====
    while self.pending_events > 0 {
        ticks += 1;
        // NOT PART OF LOOP, JUST FOR US TO SEE WHAT TICK IS EXCECUTING
        print(format!("===== TICK {} =====", ticks));

        // ===== 2. TIMERS =====
        self.process_expired_timers();

        // ===== 2. CALLBACKS =====
        // Timer callbacks and if for some reason we have postponed callbacks
        // to run on the next tick. Not possible in our implementation though.
        self.run_callbacks();

        // ===== 3. IDLE/PREPARE =====
        // we won't use this

        // ===== 4. POLL =====
        // First we need to check if we have any outstanding events at all
        // and if not we're finished. If not we will wait forever.
        if self.pending_events == 0 {
            break;
        }

        // We want to get the time to the next timeout (if any) and we
        // set the timeout of our epoll wait to the same as the timeout
        // for the next timer. If there is none, we set it to infinite (None)
        let next_timeout = self.get_next_timer();

        let mut epoll_timeout_lock = self.epoll_timeout.lock().unwrap();
        *epoll_timeout_lock = next_timeout;
        // We release the lock before we wait in `recv`
        drop(epoll_timeout_lock);

        // We handle one and one event but multiple events could be returned
        // on the same poll. We won't cover that here though but there are
        // several ways of handling this.
        if let Ok(event) = self.event_reciever.recv() {
            match event {
                PollEvent::Timeout => (),
                PollEvent::Threadpool((thread_id, callback_id, data)) => {
                    self.process_threadpool_events(thread_id, callback_id, data);
                }
                PollEvent::Epoll(event_id) => {
                    self.process_epoll_events(event_id);
                }
            }
        }
        self.run_callbacks();

        // ===== 5. CHECK =====
        // an set immidiate function could be added pretty easily but we
        // won't do that here

        // ===== 6. CLOSE CALLBACKS ======
        // Release resources, we won't do that here, but this is typically
        // where sockets etc are closed.
    }

    // We clean up our resources, makes sure all destructors runs.
    for thread in self.thread_pool.into_iter() {
        thread.sender.send(Task::close()).expect("threadpool cleanup");
        thread.handle.join().unwrap();
    }

    self.epoll_registrator.close_loop().unwrap();
    self.epoll_thread.join().unwrap();

    print("FINISHED");
}
```


## Shortcuts

I'll mention some obvious shortcuts right here so you are aware of them. There are many "exceptions" that we don't cover in our example. We are focusing on the big picture just so we're on the same page. The `process.nextTick` function and the `setImmediate` function are two examples of this. I explained how we did skip the fact that the next timeout will define how long the `poll` phase will potentially block instead of continue the loop like we do here.

We don't cover the case where a server under heavy load might have too many callbacks to reasonably run in one `poll` which means that we could starve our I/O resources in the meantime waiting for them to finish, and probably several similar cases that a production
runtime should care about.

As you'll probably notice, implementing a simple version is more than enough work
for us to cover in this book, but hopefully you'll find yourself in pretty good
shape to dig further once we're finished.

