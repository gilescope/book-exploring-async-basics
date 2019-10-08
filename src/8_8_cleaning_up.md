# Cleaning up

Lastly we clean up after ourselves by joining all our threads so we know that
all destructors have ran without errors after we're finished.

```rust, no_run
// We clean up our resources, makes sure all destructors runs.
for thread in self.thread_pool.into_iter() {
    thread.sender.send(Event::close()).expect("threadpool cleanup");
    thread.handle.join().unwrap();
}

self.epoll_registrator.close_loop().unwrap();
self.epoll_thread.join().unwrap();

print("FINISHED");
```

Before we join the thread on our thread pool we send a `Close` event which
unparks the thread and exits the loop we're in.

How we clean up after ourselves in `minimio` will be covered in the book that
covers this topic specifically.

If you want to read more about why it's a good practice to join all threads
and clean up after us before we exit, I can recommend the article [Join Your Threads](https://matklad.github.io/2019/08/23/join-your-threads.html)
written by [@matklad](https://matklad.github.io/).
