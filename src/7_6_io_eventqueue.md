# I/O eventqueue

```rust
fn process_epoll_events(&mut self, event_id: usize) {
    let id = self
        .epoll_event_cb_map
        .get(&(event_id as i64))
        .expect("Event not in event map.");
    let callback_id = *id;
    self.epoll_event_cb_map.remove(&(event_id as i64));

    self.callbacks_to_run.push((callback_id, Js::Undefined));
    self.epoll_pending_events -= 1;
}
```