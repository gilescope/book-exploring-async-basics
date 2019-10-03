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
    let res = (task.task)();

    print(format!("finished running a task of type: {}.", task.kind));
    threadp_sender.send((i, task.callback_id, res)).unwrap();
}

print("FINISHED");
```

The first thing we do is to listen on our `Recieve` part of the channel (remember,
we gave the `Send` part to our `main` thread). This function will actually
`park` our thread until we receive a message so it consumes no resources while
waiting.

When we get a `task` we first print out what kind of task we got. The next thing
we do is to run our task `let res = (task.task)();`. This is where the work will
actually be done. We know from the signature of this task that it returns a `Js`
object once it's finished.

The nest thing we do is to print out that we finished running a task, before we
send the `thread_id`, the `callback_id` and the data returned as a `Js` object back
to our main thread.

This will loop