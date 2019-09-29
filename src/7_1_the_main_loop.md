# The main loop

Let's put our event loop logic in the `run` function of our `Runtime`. The code
which we present on this chapter is the body of this `run` function.

I'll include the whole method last so you can see it all together.

```rust
impl Runtime {
    fn run() {
        ...
    }
}
```

## Initialization


```rust
let rt_ptr: *mut Runtime = self;
unsafe { RUNTIME = rt_ptr as usize };
let mut timers_to_remove = vec![]; 
let mut ticks = 0; // just for us priting out

// First we run our "main" function
f();
```

The first two lines is just a `hack` we use in our code to make it "look" more
like javascript. Here we take the pointer to `self` and set it in the global
variable `RUNTIME`. We could instead pass our `runtime` around but that wouldn't
be very ergonomic. Another option would be to use `lazy_static` crate to initlialize
this field in a safer way, but we'd have to explain what `lazy_static` do to keep
our promise of minimal "magic". To be honest, we only set this once, and it's set at
the start of of our eventloop and not touched until it's finished so in this case we
could argue it's safe to do it like this.

The variable `timers_to_remove` is for us to keep track of the timers we've set.
`ticks` is only a counter for us to keep track of how many times we've looped
to display.

The first thing we do to kick of the code is invoking `f()`. `f` will be the
code we wrote in the `javascript` function in the last chapter. If this is empty
nothing will happen.

## Starting the event loop

```rust
// ===== EVENT LOOP =====
while self.pending_events > 0 {
    ticks += 1;
```
There are two things to note here:

`self.pending_events` isn't in our runtime struct yet so we need to add that. 
This variable keeps track of how many pending events we have, so that when no 
events are left we exit the loop since our eventloop is finished.

So where does these events come from? In our `javascript` function in the 
previous chapters you probably noticed that we called functions like 
`set_timeout` and `Fs::read`. These functions are defined in the Node runtime 
(as they are in ours), and they don't do much except from regestering events. 
So when one of these events are registered this counter is increased.

`ticks` is just increasing a `tick` in the counter.

So our Runtime struct looks like this now:
```rust
pub struct Runtime {
    pending_events: usize,
}
```

## 1. Effectuate timers

`self.effectuate_timers(&mut timers_to_remove);`

We check here if any timers has expired. I couldn't find a better word for it
than `effectuate` but it basically mean that if we have timers that are expired
we schedule the callbacks for the expired timers to run at the first call to `self.run_callbacks()`.

Worth noting here is that timers with a timeout of `0` will already have timed
out by the time we reach this function.

## 2. Callbacks

`self.run_callbacks();`

Now we could have ran the callbacks in the timer `step` but since this is the next
step of our loop we do it here instead.

> This step might seem unnecessary here but in Node it has a function. Some
> types of callbacks will be deferred to the `next tick`, which means that they're
> not run immediately, but deferred to this step on the next loop. We won't implement
> this functionality here but it's worth noting.

## 3. Idle/Prepare

This is a step mostly used by Nodes internals. It's not important for understanding
the big picture here but I included it since it's something you see in Nodes
documentation so you know where we're at in the loop at this point.

## 4. Poll

This is really where everything happens. I refer to the `epoll/kqueue/IOCP` 
eventqueue as `epoll` here just so you know that it's not only `epoll` we're
waiting for. From now on I will refer to the cross platform event queue as `epoll`
in the code.

First we check if the OS has reported any events, and if so we schedule the
corresponding callbacks to run.
```rust
self.process_epoll_events();
```

Next we check if our threadpool has finished any work and if so we schedule their
corresponding callbacks to be run too:

```rust
self.process_threadpool_events();
```

If our `epoll` queue or our `threadpool` registered any callbacks we run them now.

```rust
self.run_callbacks();
```

> There are some important differences between our implementation and Nodes here.
>
> We only check if any events are registered here and then continue on. This is
> suboptimal since if we're wasting cycles by looping when there might be nothing
> to do on the next iteration. 
> 
> Node solves this by calculating the time until the next timeout (in step 1) 
> times out. Let's say that the next timer times out in 10 seconds. In node 10 
> seconds is then passed as a timeout for the `polls` in our poll phase so even 
> though no event has happened it will wake up again and iterate so it executes 
> the next timer when it starts the loop again. As you understand, that means
> that the timer will not run at the exact same time as it times out, but it's
> potentially much more efficient than what we do here.

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
// to use. We release in every callback instead
```

## 7. Give 

I pretty much explain this step in the comments. Typically releasing resources,
like closing sockets, is done here.

## The `run` function

```rust
impl Runtime {
    pub fn run(&mut self, f: impl Fn()) {
        let rt_ptr: *mut Runtime = self;
        unsafe { RUNTIME = rt_ptr as usize };
        let mut ticks = 0; // just for us priting out

        // First we run our "main" function
        f();

        // ===== EVENT LOOP =====
        while self.pending_events > 0 {
            ticks += 1;

            // ===== 2. TIMERS =====
            self.process_expired_timers();

            // NOT PART OF LOOP, JUST FOR US TO SEE WHAT TICK IS EXCECUTING
            if !self.callbacks_to_run.is_empty() {
                print(format!("===== TICK {} =====", ticks));
            }

            // ===== 2. CALLBACKS =====
            // Timer callbacks and if for some reason we have postponed callbacks
            // to run on the next tick. Not possible in our implementation though.
            self.run_callbacks();

            // ===== 3. IDLE/PREPARE =====
            // we won't use this

            // ===== 4. POLL =====
            self.process_epoll_events();
            self.process_threadpool_events();
            self.run_callbacks();

            // ===== 5. CHECK =====
            // an set immidiate function could be added pretty easily but we 
            // won't do that here

            // ===== 6. CLOSE CALLBACKS ======
            // Release resources, we won't do that here, but this is typically
            // where sockets etc are closed.

            // Let the OS have a time slice of our thread so we don't busy loop
            // this could be dynamically set depending on requirements or load.
            thread::park_timeout(std::time::Duration::from_millis(1));
        }
        print("FINISHED");
    }
}
```


## Shortcuts

I'll mention some obvious shortcuts right here so you are aware of them. There are many "exceptions" that we don't cover in our example. We are focusing on the big picture just so we're on the same page. The `process.nextTick` function and the `setImmediate` function are two examples of this. I explained how we did skip the fact that the next timeout will define how long the `poll` phase will potentially block instead of continue the loop like we do here.

We don't cover the case where a server under heavy load might have too many callbacks to reasonably run in one `poll` which means that we could starve our I/O resources in the meantime waiting for them to finish, and probably several similar cases that a production
runtime should care about.

As you'll probably notice, implementing a simple version is more than enough work
for us to cover in this book, but hopefully you'll find yourself in pretty good
shape to dig further once we're finished.

