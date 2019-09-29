# Cleaning up

```rust
// We clean up our resources, makes sure all destructors runs.
for thread in self.thread_pool.into_iter() {
    thread.sender.send(Event::close()).expect("threadpool cleanup");
    thread.handle.join().unwrap();
}

self.epoll_registrator.close_loop().unwrap();
self.epoll_thread.join().unwrap();

print("FINISHED");
```