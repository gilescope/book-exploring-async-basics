# The Runtime

We've already introduced most of the Runtime we're planning but it's still a lot
of missing pieces. We still don't have a threadpool or a I/O eventloop running
so the next step is to set this up so we can start focusing on how to work with
our "Node" Runtime.

## Let's take this step by step

The first thing we do is to add a `new` method that returns an instance of our
`Runtime`:

```rust
impl Runtime {
    pub fn new() -> Self {
```
Now the real work starts. Next up is our threadpool. Node defaults to 4 threads
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

The next step is to 


```rust
impl Runtime {
    pub fn new() -> Self {
        // ===== THE REGULAR THREADPOOL =====
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

        // ===== EPOLL THREAD =====
        // Only wakes up when there is a task ready
        let (epoll_sender, epoll_reciever) = channel::<usize>();
        //let (epoll_start_sender, epoll_start_reciever) = channel::<usize>();
        let mut poll = minimio::Poll::new().expect("Error creating epoll queue");
        let registrator = poll.registrator();
        thread::Builder::new()
            .name("epoll".to_string())
            .spawn(move || {
                let mut events = minimio::Events::with_capacity(1024);
                loop {
                    match poll.poll(&mut events) {
                        Ok(v) if v > 0 => {
                            for i in 0..v {
                                let event = events.get_mut(i).expect("No events in event list.");
                                print(format!("epoll event {} is ready", event.id().value()));
                                epoll_sender.send(event.id().value() as usize).unwrap();
                            }
                        }
                        Err(e) => panic!("{:?}", e),
                        _ => (),
                    }
            }
            // FIXME: This thread needs to finish
            print("FINISHED");
            })
            .expect("Error creating epoll thread");

        Runtime {
            thread_pool: threads.into_boxed_slice(),
            available: (0..4).collect(),
            callback_queue: HashMap::new(),
            next_tick_callbacks: vec![],
            identity_token: 0,
            pending_events: 0,
            threadp_reciever,
            epoll_reciever,
            //epoll_queue: queue,
            epoll_pending: 0,
            //epoll_starter: epoll_start_sender,
            epoll_event_cb_map: HashMap::new(),
            timers: BTreeMap::new(),
            epoll_registrator: registrator,
        }
    }
}
```