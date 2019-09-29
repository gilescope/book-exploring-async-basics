# Timers

## 1. Check expired timers
The first step in the event loop is checking for expired timers, and we do this 
in the `self.check_expired_timers()` function

```rust
    fn process_expired_timers(&mut self) {
        // Need an intermediate variable to please the borrowchecker
        let timers_to_remove = &mut self.timers_to_remove;

        self.timers
            .range(..=Instant::now())
            .for_each(|(k, _)| timers_to_remove.push(*k));

        while let Some(key) = self.timers_to_remove.pop() {
            let callback_id = self.timers.remove(&key).unwrap();
            self.callbacks_to_run.push((callback_id, Js::Undefined));
        }
    }

    fn get_next_timer(&self) -> Option<i32> {
        self.timers.iter().nth(0).map(|(&instant, _)| {
            let mut time_to_next_timeout = instant - Instant::now();
            if time_to_next_timeout < Duration::new(0, 0) {
                time_to_next_timeout = Duration::new(0, 0);
            }
            time_to_next_timeout.as_millis() as i32
        })
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

Before we continue, let's recap by looking what members of the `Runtime` struct
we used here:

```rust, no_run, noplaypen
pub struct Runtime {
    pending_events: usize,
    next_tick_callbacks: Vec<(usize, Js)>,
    timers: BTreeMap<Instant, usize>,
}
```