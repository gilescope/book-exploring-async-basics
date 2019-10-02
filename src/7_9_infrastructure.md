# Infrastructure

Now, for everything to work we need some helpers to make our infrastructure work.

First of all, we need a way to get the `id` of an available thread.

```rust
   fn get_available_thread(&mut self) -> usize {
        match self.available_threads.pop() {
            Some(thread_id) => thread_id,
            // We would normally return None and the request and not panic!
            None => panic!("Out of threads."),
        }
    }
```
As you see, we take one huge shortcut here. If we run out of threads, we `panic!`.
This is not good, and we should rather implement logic to queue these requests
and run them as soon as a thread is available. However, our code is already getting
long, and it's not very important for our goal of learning about `async`.

Maybe this implementing such a queue is a good reader-exercise? Feel free to fork
the repository and go ahead :)

The next thing we need to do is to create an unique identity for our callbacks.
```rust

/// If we hit max we just wrap around
fn generate_identity(&mut self) -> usize {
    self.identity_token = self.identity_token.wrapping_add(1);
    self.identity_token
}

fn generate_cb_identity(&mut self) -> usize {
    let ident = self.generate_identity();
    let taken = self.callback_queue.contains_key(&ident);

    // if there is a collision or the identity is already there we loop until we
    // find a new one. We don't cover the case where there are `usize::max_value()`
    // number of callbacks waiting since that if we're fast and queue a new event
    // every nanosecond that will stilltake 585 years to do on a 64 bit system.
    if !taken {
        ident
    } else {
        loop {
            let possible_ident = self.generate_identity();
            if self.callback_queue.contains_key(&possible_ident) {
                break possible_ident;
            }
        }
    }
}
```
The function `generate_cb_identity` is where it all happens, `genereate_identity` is just
a small function so we try to avoid the long functions we had in the introduction.

>Now, there are some important considerations to be aware of. Even though we use
>several threads, we use a regular `usize` here and the reason for that is that
>it's only one thread that will be generating Id's. This could cause problems if
>several threads tried to `read` and `generate` new Id's at the same time.

We use the `wrapping_add` method on `usize` to get the next Id, this means that
when we reach `18446744073709551615` we wrap around to 0 again.

We do check of our callback_queue contains our key (even though that is unlikely
by design), and if it's taken we just generate a new one until we find a available
one.

Next up is the method we use to add a callback to our `callback_queue`:
```rust
/// Adds a callback to the queue and returns the key
fn add_callback(&mut self, ident: usize, cb: impl FnOnce(Js) + 'static) {
    let boxed_cb = Box::new(cb);
    self.callback_queue.insert(ident, boxed_cb);
}
```
If you haven't seen the signature `cb: impl impl FnOnce(Js) + 'static` before I'll
explain it briefly here.

The `impl ...` means that we accept an arguments that implements the trait `FnOnce(Js)`
with a `'static` lifetime. [FnOnce](https://doc.rust-lang.org/std/ops/trait.FnOnce.html) is
a trait implemented by `closures`. There are three main traits a `closure` can implement
in Rust and `FnOnce` is the one you'll use if you take ownership over a captured
variable and consume it.

Since you consume the variable a `closure` implementing `FnOnce` can only be called
once. Our closure takes ownership over the `Js` parameter we pass in and consumes
it. It's implicit that `FnOnce` returns `()` in this case so we don't have to write
`FnOnce(Js) -> ()`.

Since callbacks are meant to only be called once, this is a perfectly fine bound
for us to use here.

Now, traits doesn't have a size so for the compiler to be able to allocate space
for it on the stack we either need to take a reference `&FnOnce(Js)` or place it
on the heap using `Box`. We do the latter since that's the only thing that makes
sense for our use case. Box is a pointer to a heap allocated variable which we do
know the size of so we store that reference in our `callback_queue` HashMap.

>What makes a closure?
> A function in rust can be defined as easily as `|| { }`. If this is all we write
> it's the same as a function pointer, equivalent to just referencing `my_method` (whithout parenthesis).
> It becomes a `closure` as soon as you "close" over your environment by referencing
> variables that's not owned by the `function`.
> 
> `Fn` traits are automatically implemented, and whether it implements `Fn`, `FnMut`
> or `FnOnce` depend whether you take ownership over a non-copy variable, take a shared
> reference `&` or an exclusive reference `&mut` (often called a mutable reference).

Now that we got some closure basics out of the way we can move on. The next method
is how we register `I/O` work. This is how we register an `epoll` event with our runtime:

```rust
pub fn register_event_epoll(&mut self, token: usize, cb: impl FnOnce(Js) + 'static) {
    self.add_callback(token, cb);

    print(format!("Event with id: {} registered.", token));
    self.pending_events += 1;
    self.epoll_pending_events += 1;
}
```

The first thing we do is to add the callback to our `callback_queue`, calling the
method we explained previously. Next we do a print statement, just since we want
to print out the flow of our program we need to add this at strategic places.

> One important thing to note here. Our `token` in this case is already guaranteed
> to be unique. We generate it in the `Http` module (which is the only one registering
> events by using this method in our example). The reason for this will become clear
> in a few short chapters. Just note that we don't need to call `generate_cb_identity` here.

We increase the counters on both `pending_events` and `epoll_pending_events`.

Our next method registers work for the thread pool

```rust
pub fn register_event_threadpool(
    &mut self,
    task: impl Fn() -> Js + Send + 'static,
    kind: ThreadPoolTaskKind,
    cb: impl FnOnce(Js) + 'static,
) {
    let callback_id = self.generate_cb_identity();
    self.add_callback(callback_id, cb);

    let event = Task {
        task: Box::new(task),
        callback_id,
        kind,
    };

    // we are not going to implement a real scheduler here, just a LIFO queue
    let available = self.get_available_thread();
    self.thread_pool[available].sender.send(event).expect("register work");
    self.pending_events += 1;
}
```
Let's first have a look at the arguments to this function (aside from `&mut self`).

`task: impl Fn() -> Js + Send + 'static` is a task we want to run on a separate
thread. This closure has the bond: `Fn() -> Js + Send + 'static` which means
it's a `closure` that takes no arguments, but returns a type of `Js`. It needs to
be `'Send` since we're sending this task to another thread.

`kind: ThreadPoolTaskKind` lets us know what kind of task this. We do this for
two reasons:

1. We need to be able to signal a `Close` event to our threads
2. We want to be able to print the kind of task each event received.

As you understand, we don't have to create a `Kind` for every task, but since we
want to print out what the thread received we need some way of judging what kind
of task each thread received.

The last argument `cb: impl FnOnce(Js) + 'static` is our callback. It's not a coincidence
that our `task` returns a type of `Js` and our callback takes a `Js` as an argument. The
result of the work we do in our thread is the input to our callback. This closure doesn't
need to be `Send` since we don't pass the callback itself to the thread pool.

Next we generate a new identity with `self.generate_cb_identity()` and we add the
callback to our callback queue.

Then we construct a new `Event`, and as I have shown earlier, we need to `Box` the
closure.

Now, the last part could be made arbitrarily complex. This is where you decide how
you want to schedule your work to the thread pool. In our case we just get an
available thread (and `panic!` if we're out of thread - ouch), and we send our task
to the thread which then runs it until it's finished.

You could make priorities based on `TaskKind`, you could try to decide which tasks
are short and which are long and prioritize them based on load. A lot of exiting
stuff could be done here. We will choose the simplest possible one though and just
push them directly to a thread in the order they come.

The last part of the "infrastructure" is a function to set a timeout. 

```rust
    fn set_timeout(&mut self, ms: u64, cb: impl Fn(Js) + 'static) {
        // Is it theoretically possible to get two equal instants? If so we'll have a bug...
        let now = Instant::now();
        let cb_id = self.generate_cb_identity();
        self.add_callback(cb_id, cb);
        let timeout = now + Duration::from_millis(ms);
        self.timers.insert(timeout, cb_id);
        self.pending_events += 1;
        print(format!("Registered timer event id: {}", cb_id));
    }
```

Set timeout uses `std::time::Instant` to get a representation of "now". It's the first
thing we do since the user expects the timeout to be calculated from "now", and some
of our operations here might take a little time.

We generate an identity for the callback `cb` passed in to `set_timeout` and add
that callback to our callback queue.

We add the `duration` in milliseconds to our `Instant` so we know at what time
our timeout times out.

We insert the `callback_id` instant to our `BtreeMap` with the calculated `Instant` as
the key.

We increase the counter for `pending_events` and print out a message for us to
be able to follow the flow of our program.

> This might be a good time to talk briefly about our choice of a `BTreeMap` as
> the collection we store timers in.
>
> From the documentation we can read _"In theory, a binary search tree (BST) is the optimal choice for a sorted map, as a perfectly balanced BST performs the theoretical minimum amount of comparisons necessary to find an element (log2n)."_
> Now, this isn't a Binary Tree but a BTree. While a BST allocates one node for each value, a BTree allocates
> a small `Vec` of values for each node. Modern computers reads much more data than we normally ask for
> into its caches, and thats one reason they love contagious parts of memory. A BTree will result in
> a more optimal "cache efficiency" which often trumps the gains of the theoretically more optimal
> algorithm in a true BST.
>
> Lastly, since we're talking about searching sorted collections here, and timeouts, is a perfect example
> of such, we'll of course use this when it's so readily available to us in Rusts standard library.
