# The Threadpool

The threadpool is where we'll process the CPU intensive tasks and the I/O tasks
which can't be reasonably handled by `epoll, kqueue or IOCP`. One of these tasks
are file system operations.

The reason for doing file I/O in the thread pool is complex, but the main takeaway
is that due to how files are cached and how the hard drive works, most often the
file I/O will be `Ready` almost immediately, so waiting for that in a event queue
has very little effect in practice.

The second reason is that while Windows do have a completion based model, Linux
and Macos doesn't. Reading a file into a buffer which your process controls can take some
time, and if we do that in our main loop it will block a little bit which we really try
to avoid.

By doing this in a thread pool we make sure that these operations won't block our
main event loop and only notify us once the data is ready for us in memory.

The code we need to add to process events from the thread pool is short an simple:

```rust, no_run
fn process_threadpool_events(&mut self, thread_id: usize, callback_id: usize, data: Js {
    self.callbacks_to_run.push((callback_id, data));
    self.available_threads.push(thread_id);
}
```

We take `thread_id`, `callback_id` and `data` as arguments. We get this through
the channel we have shared with our `threadpool` threads.

Once we have that information we push our callbacks into the queue of `callbacks_to_run`
which will run on the next call to `run_callbacks` we went through in the previous chapter.

Last we take the `id` of the thread that sent the finished task and put it into
our pool of available threads.
