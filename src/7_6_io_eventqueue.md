# I/O event queue

The I/O event queue is what handles most of our I/O tasks. Now we'll go through
how we register events to that queue later on, but once an event is ready we
it sends the `event_id` through our channel.

```rust
    fn process_epoll_events(&mut self, event_id: usize) {
        let event_id = event_id as i64;

        let callback_id = *self
            .epoll_event_cb_map
            .get(&event_id)
            .expect("Event not in event map.");

        self.epoll_event_cb_map.remove(&event_id);

        self.callbacks_to_run.push((callback_id, Js::Undefined));
        self.epoll_pending_events -= 1;
    }
```

Now the first thing we do is to cast the `event_id` to an `i64` which both our
methods expect just so we don't need to clutter our code with more casts than we
need.

The second thing we do is to get the `callback_id` which is the Id of the callback
we want to run once the event is finished.

We'll go through why we want to map the `event_id` to a `callback_id` later but
it's just convenient for us to keep these Id's seperated for now.

> You might be wondering how we can `dereference` `self` in this call: `let callback_id = *self...`.
> The reason for this is that we're not actually dereferencing `self`, but we're
> dereferencing the value returned from `self.epoll_event_cb_map.get(&event_id)...`.
> Since this is an `usize`, which is one of our primitive types that implement `Copy`
> this will work fine.

Next up we remove the event from our `event_id <-> callback_id` map with 
`self.epoll_event_cb_map.remove(&event_id)`.

Lastly we add the `callback_id` to the collection of callbacks to run. We pass
in `Js::Undefined` since we'll not actually pass any data along here. You'll see
why when we reach the `[Http module](./8_3_http_module.md) chapter, but the main
point is that the I/O queue doesn't return any data itself, it just tells us that
data is ready to be read.

Lastly it's only for our own bookeeping we decrement the count of outstanding
`epoll_pending_events` so we keep track of how many events we have pending.
