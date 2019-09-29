# Setting up our runtime

## The Threadpool

We still don't have a threadpool or a I/O eventloop running
so the next step is to set this up so we can start focusing on how to work with
our "Node" Runtime.

## Let's take this step by step

The first thing we do is to add a `new` method that returns an instance of our
`Runtime`:

```rust
impl Runtime {
    pub fn new() -> Self {
```
Now the real work starts. Next up is our thread pool. Node defaults to 4 threads
which we will copy. This is configurable in `Node` but we will take a shortcut
and hard code it:

```rust
let (threadp_sender, threadp_reciever) = channel::<(usize, usize, Js)>();
let mut threads = Vec::with_capacity(4);

for i in 0..4 {
    let (evt_sender, evt_reciever) = channel::<Event>();
    let threadp_sender = threadp_sender.clone();

    let handle = thread::Builder::new()
        .name(format!("pool{}", i))
        .spawn(move || {

            while let Ok(event) = evt_reciever.recv() {

                print(format!("recived a task of type: {}", event.kind));
                let res = (event.task)();

                print(format!("finished running a task of type: {}.", event.kind));
                threadp_sender.send((i, event.callback_id, res)).unwrap();
            }

            print("FINISHED");
        })
        .expect("Couldn't initialize thread pool.");

    let node_thread = NodeThread {
        handle,
        sender: evt_sender,
    };

    threads.push(node_thread);
}

```

There is a lot going on here so let's break it down.

First we set up a channel to communicate with our threadpool. The channel will
be of type `Channel<(usize, usize, JS)>` as you notice we use a `tuple` here instead
of a struct, but we try to keep this simple now and won't introduce any new
functionality.

```rust
let (threadp_sender, threadp_reciever) = channel::<(usize, usize, Js)>();
```
