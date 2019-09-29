# The Threadpool

```rust
fn process_threadpool_events(&mut self, thread_id: usize, callback_id: usize, data: Js {
    self.callbacks_to_run.push((callback_id, data));
    self.available_threads.push(thread_id);
}
```