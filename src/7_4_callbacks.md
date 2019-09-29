# Callbacks

## 2. Process callbacks
The next step is to handle any callbacks we've scheduled to run.

```rust
fn run_callbacks(&mut self) {
    while let Some((callback_id, data)) = self.callbacks_to_run.pop() {
        let cb = self.callback_queue.remove(&callback_id).unwrap();
        cb(data);
        self.pending_events -= 1;
    }
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



Let's recap by looking at what members of the `Runtime` struct we used here:

```rust
pub struct Runtime {
    callback_queue: HashMap<usize, Box<dyn FnOnce(Js)>>,
}
```