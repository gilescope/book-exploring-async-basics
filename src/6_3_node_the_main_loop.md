# The main loop

Before we implement the eventlopp we need to set up our Runtime so we can save all our state there:

As of now, our Runtime struct will look like this: 

```rust
pub struct Runtime {
}

```

Don't worry, we'll fill in the fields as we go along but I didn't want you to stop now an try to figure out what everything is.

Let's get back on track. And talk a bit about the eventloop, which probably is the most interesting part of code in this book since there has been som much written about it:

```rust
impl Runtime {

    pub fn run(&mut self, f: impl Fn()) {
        let rt_ptr: *mut Runtime = self;
        unsafe { RUNTIME = rt_ptr as usize };
        let mut timers_to_remove = vec![]; 
        let mut ticks = 0; // just for us priting out

        // First we run our "main" function
        f();

        // ===== EVENT LOOP =====
        while self.pending_events > 0 {
            ticks += 1;

            // ===== TIMERS =====

            self.timers
                .range(..=Instant::now())
                .for_each(|(k, _)| timers_to_remove.push(*k));

            while let Some(key) = timers_to_remove.pop() {
                let callback_id = self.timers.remove(&key).unwrap();
                self.next_tick_callbacks.push((callback_id, Js::Undefined));
            }

            // NOT PART OF LOOP, JUST FOR US TO SEE WHAT TICK IS EXCECUTING
            if !self.next_tick_callbacks.is_empty() {
                print(format!("===== TICK {} =====", ticks));
            }

            // ===== CALLBACKS =====
            while let Some((callback_id, data)) = self.next_tick_callbacks.pop() {
                let cb = self.callback_queue.remove(&callback_id).unwrap();
                cb(data);
                self.pending_events -= 1;
            }

            // ===== IDLE/PREPARE =====
            // we won't use this

            // ===== POLL =====

            // First poll any epoll/kqueue/IOCP
            while let Ok(event_id) = self.epoll_reciever.try_recv() {
                let id = self
                    .epoll_event_cb_map
                    .get(&(event_id as i64))
                    .expect("Event not in event map.");
                let callback_id = *id;
                self.epoll_event_cb_map.remove(&(event_id as i64));

                self.next_tick_callbacks.push((callback_id, Js::Undefined));
                self.epoll_pending -= 1;
            }

            // then check if there is any results from the threadpool
            while let Ok((thread_id, callback_id, data)) = self.threadp_reciever.try_recv() {
                self.next_tick_callbacks.push((callback_id, data));
                self.available.push(thread_id);
            }

            // ===== CHECK =====
            // an set immidiate function could be added pretty easily but we won't do that here

            // ===== CLOSE CALLBACKS ======
            // Release resources, we won't do that here, it's just another "hook" for our "extensions"
            // to use. We release in every callback instead

            // Let the OS have a time slice of our thread so we don't busy loop
            // this could be dynamically set depending on requirements or load.
            thread::park_timeout(std::time::Duration::from_millis(1));
        }
        print("FINISHED");
    }
}
```

This is a lot to parse and will be a lot to take in right now. But this is also
the heart of our program. Let's go through and explain everything.

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

The first thing we do to kick of the code is invoking `f()`. `f` will be the code we wrote in the `javascript` function in the last chapter. It's the code the user wrote.

```rust
// ===== EVENT LOOP =====
while self.pending_events > 0 {
    ticks += 1;
```
The next thing we do is to start our eventloop. There are two things to note here:

`self.pending_events` isn't in our runtime struct yet so we need to add that. This variable keeps track of how many pending events we have, so that when no events are left we exit the loop since our eventloop is finished.

So where does these events come from? In our `javascript` function in the previous chapters you probably noticed that we called functions like `set_timeout` and `Fs::read`. These functions are defined in the Node runtime (as they are in ours), and they don't do much except from regestering events. So when one of these events are registered this counter is increased.

`ticks` is just increasing a `tick` in the counter.

So our Runtime struct looks like this now:
```rust
pub struct Runtime {
    pending_events: usize,
}

```
## 1. Check timers
The first step in the event loop is checking the timers:

```rust
// ===== TIMERS =====
self.timers
    .range(..=Instant::now())
    .for_each(|(k, _)| timers_to_remove.push(*k));

while let Some(key) = timers_to_remove.pop() {
    let callback_id = self.timers.remove(&key).unwrap();
    self.next_tick_callbacks.push((callback_id, Js::Undefined));
}

// NOT PART OF LOOP, JUST FOR US TO SEE WHAT TICK IS EXCECUTING
if !self.next_tick_callbacks.is_empty() {
    print(format!("===== TICK {} =====", ticks));
}
```

The first thing to note here is that we check `self.timers` and to understand the
rest of the syntax we'll have to look what kind of collection this is.

Now I chose a `BTreeMap<Instant, usize>` for this collection. The reason is that
i want to have many `Instant`'s chronologically. When I add a timer, I calculate
at what instance it's supposed to be run and I add that to this collection.

> BTrees are a very good data structure when you know that your keys will be ordered.

Choosing a `BTreeMap` here allows me to get a range `range(..=Instant::noew())`
which is from the start of the map, up until or equal to the instant NOW.

Now I take every key in this range and add it to `timers_to_remove`, and the reason
for this is that I found no good way to both get a range and remove the key's in one
operation without allocating a small buffer every time. You can iterate over the range
but due to the ownership rules you can't remove them at the same time, and we want to
remove the timers, we've run.

The eventloop will run repeatedly so avoiding any allocations inside the loop is smart. There is no need to have this overhead.

```rust
while let Some(key) = timers_to_remove.pop() {
    let callback_id = self.timers.remove(&key).unwrap();
    self.next_tick_callbacks.push((callback_id, Js::Undefined));
}
```

The next step is to take every timer that has expired, remove the timer from our `self.timers` collection and get their `callback_id`.

As I explained in the previous chapter, this is an unique Id for this callback. What's
important here is that **we don't run the callback **immediately**. Node actually registers callbacks to be run on the next `tick`. An exception is the timers since they either have timed out or is a timer with a timeout of `0`. In this case a timer will not wait for the next tick if it has timed out, or in the case if it has a timeout of `0` they will be invoked immediately as you'll see next.

Anyway, for now we add the callback id's to `self.next_tick_callbacks`.

Before we go on. Let's update our `Runtime` struct to reflect what we've seen:

```rust
pub struct Runtime {
    pending_events: usize,
    next_tick_callbacks: Vec<(usize, Js)>,
    timers: BTreeMap<Instant, usize>,
}
```
## 2. Process callbacks
The next step is to handle any callbacks we've scheduled to run.

```rust
// ===== CALLBACKS =====
while let Some((callback_id, data)) = self.next_tick_callbacks.pop() {
    let cb = self.callback_queue.remove(&callback_id).unwrap();
    cb(data);
    self.pending_events -= 1;
}
```

Here we `pop` off all callbacks that are scheduled to run. As you see from our last update on the `Runtime` struct. `next_tick_callbacks` is an array of callback_id and an argument type of `Js`.

So when we've got a `callback_id` we find the corresponding callback we have stored in `self.callback_queue` and remove the entry. What we get in return is a callback of type
`Box<dyn FnOnce(Js)>`. We're going to explain this type more later but it's basically a closure stored on the heap that takes one argument of type `Js`.

`cb(data)` runs the code in this closure. After it's done it's time to decrease our counter of pending events: `self.pending_events -= 1;`.

> Now, this step is important. As you might understand, any long running code in this callback is going to block our `eventloop`, preventing it from progressing. So no new callbacks are handleded and no new events are registered. This is why it's bad to write code that blocks the eventloop.



Let's update our Runtime struct again:

```rust
pub struct Runtime {
    callback_queue: HashMap<usize, Box<dyn FnOnce(Js)>>,
    pending_events: usize,
    next_tick_callbacks: Vec<(usize, Js)>,
    timers: BTreeMap<Instant, usize>,
}
```
## 3. Idle/prepare
```rust
// ===== IDLE/PREPARE =====
// we won't use this
```

The Idle/Prepare step is reportedly used internally by Node. They're not so interesting for us in understanding Nodes eventloop so we skip this step.

## 4. Poll

The next phase is to check if any events are ready from either our threadpool or our eventqueue.

```rust
 // ===== POLL =====
// First poll any epoll/kqueue/IOCP
while let Ok(event_id) = self.epoll_reciever.try_recv() {
    let id = self
        .epoll_event_cb_map
        .get(&(event_id as i64))
        .expect("Event not in event map.");
    let callback_id = *id;
    self.epoll_event_cb_map.remove(&(event_id as i64));

    self.next_tick_callbacks.push((callback_id, Js::Undefined));
    self.epoll_pending -= 1;
}

// then check if there is any results from the threadpool
while let Ok((thread_id, callback_id, data))self.threadp_reciever.try_recv() {
    self.next_tick_callbacks.push((callback_id, data));
    self.available.push(thread_id);
}
```

There is a lot going on here so let's step through it:

First we check our `epoll/kqueue/IOCP` event queue and see if anything events are ready.

The first thing we do is to check if there are any incoming messages on our channel
`self.epoll_reciever.try_recv()`, as you'll see when we define this in our `Runtime` I chose to implement this using Rusts channels which is a good fit for this. No need to make it more complicated than it is.

If any events has occured we get an `event_id`. Since `event_id`'s can potentially overlap with Id's we have given previous callbacks we use a map where we give this `event` an unique `callback_id` that ties the event to the callback we have registered.

```rust
self.epoll_event_cb_map
        .get(&(event_id as i64))
        .expect("Event not in event map.");
```

Retrieves the `callback_id` we stored with this event which we then remove from the map
`self.epoll_event_cb_map.remove(&(event_id as i64))` so we don't store it indefinately.

The next two steps is the same as you saw used in the timer segment. We schedule the callback to get run on the next tick.

> One thing to note is that we pass in Js::Undefined here too even though the callback we registered is expecting data. The reason for this is that we wrap the callback to accomodate for the difference between `epoll/kqueue` and `IOCP`. In the case of `epoll/kqueue` we read the data into a buffer we pass in before we call the callback, and in the case of `IOCP` the data is already filled for us.

The next thing to check is our threadpool. As you see here we also use `Channel` here to communicate.

```rust
while let Ok((thread_id, callback_id, data))self.threadp_reciever.try_recv() {
    self.next_tick_callbacks.push((callback_id, data));
    self.available.push(thread_id);
}
```
We get some more data here. Namely a `thread_id`, `callback_id` and `data`. Now we need the `thread_id` to mark this thread as `available` so it can be used on subsequent calls to the threadpool. The `callback_id` we need in all cases to know what callback to invoke. One difference here is that the thread also holds the data we want to pass in to our callback so we also get that an pass that in to our `next_tick_callbacks` so it's available to our callback on the next tick.


 ## 5. Check

 ```rust
// ===== CHECK =====
// an set immidiate function could be added pretty easily but we won't that here
```

Node implements a check "hook" to the eventloop next. This is where 

## Shortcuts
This is the event loop. There are several things we could do here to make it a better implementation. One is to set a max backlog of callbacks to execute in a single tick, so we don't starve the threadpool or file handlers. Another is to dynamically decide if/and how long the thread could be allowed to be parked for example by looking at the backlog of events, and if there is any backlog disable it. Some of our Vec's will only grow, and not resize, so if we have a period of very high load,the memory will stay higher than we need until a restart. This could be dealt with or a differentdata structure could be used.