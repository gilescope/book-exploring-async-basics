# Implementing the Runtime


This is a lot to parse and will be a lot to take in right now. But this is also
the heart of our program. Let's go through and explain everything.




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
important here is that we don't run the callback **immediately**. Node actually registers callbacks to be run on the next `tick`. An exception is the timers since they either have timed out or is a timer with a timeout of `0`. In this case a timer will not wait for the next tick if it has timed out, or in the case if it has a timeout of `0` they will be invoked immediately as you'll see next.

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

> Shortcut. Not all of Nodes callbacks are processed here. Some callbacks is called
> directly in the `poll` phase we'll introduce below. It's not difficult to implement
> but it adds unneccecary complexity to our example so we schedula all callbacks to be
> run in this step of the process. As long as you know this is an oversimplification
> you're going to be alright :)

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

We're choosing to keep track on how many epoll events we're waiting for in
`self.epoll_pending`, so once an event is ready we'll decrement this to reflect
that we have one less event in the I/O eventloop.

If any events has occured we get an `event_id`. Since `event_id`'s can potentially overlap with Id's we have given previous callbacks we use a map where we give this `event` an unique `callback_id` that ties the event to the callback we have registered.

```rust
self.epoll_event_cb_map
        .get(&(event_id as i64))
        .expect("Event not in event map.");
```

Retrieves the `callback_id` we stored with this event which we then remove from the map
`self.epoll_event_cb_map.remove(&(event_id as i64))` so we don't store it indefinitely.



The next two steps is the same as you saw used in the timer segment. We schedule the callback to get run on the next tick.

> One thing to note is that we pass in Js::Undefined here too even though the callback we registered is expecting data. The reason for this is that we wrap the callback to accommodate for the difference between `epoll/kqueue` and `IOCP`. In the case of `epoll/kqueue` we read the data into a buffer we pass in before we call the callback, and in the case of `IOCP` the data is already filled for us.

The next thing to check is our threadpool. As you see here we also use `Channel` here to communicate.

```rust
while let Ok((thread_id, callback_id, data))self.threadp_reciever.try_recv() {
    self.next_tick_callbacks.push((callback_id, data));
    self.available.push(thread_id);
}
```

We get some more data here. Namely a `thread_id`, `callback_id` and `data`. Now we need the `thread_id` to mark this thread as `available` so it can be used on subsequent calls to the threadpool. The `callback_id` we need in all cases to know what callback to invoke. One difference here is that the thread also holds the data we want to pass in to our callback so we also get that an pass that in to our `next_tick_callbacks` so it's available to our callback on the next tick.

Now we introduced a lot of new members of our `Runtime` struct here and it's almost
finished:

```rust
pub struct Runtime {
    callback_queue: HashMap<usize, Box<dyn FnOnce(Js)>>,
    next_tick_callbacks: Vec<(usize, Js)>,
    identity_token: usize,
    pending_events: usize,
    threadp_reciever: Receiver<(usize, usize, Js)>,
    epoll_reciever: Receiver<usize>,
    epoll_pending: usize,
    epoll_event_cb_map: HashMap<i64, usize>,
    timers: BTreeMap<Instant, usize>,
    epoll_registrator: minimio::Registrator,
}
```

## Moving on

Now we've already gotten really far by explaining how our eventloop works already
in the first chapter. Now we just need to set up the infrastructure for this
loop to work.