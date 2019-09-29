# Infrastructure

```rust
   fn get_available_thread(&mut self) -> usize {
        match self.available_threads.pop() {
            Some(thread_id) => thread_id,
            // We would normally return None and the request and not panic!
            None => panic!("Out of threads."),
        }
    }

    /// If we hit max we just wrap around
    fn generate_identity(&mut self) -> usize {
        self.identity_token = self.identity_token.wrapping_add(1);
        self.identity_token
    }

    fn generate_cb_identity(&mut self) -> usize {
        let ident = self.generate_identity();
        let taken = self.callback_queue.contains_key(&ident);

        // if there is a collision or the identity is already there we loop until we find a new one
        // we don't cover the case where there are `usize::MAX` number of callbacks waiting since
        // that if we're fast and queue a new event every nanosecond that will still take 585.5 years
        // to do on a 64 bit system.
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

    /// Adds a callback to the queue and returns the key
    fn add_callback(&mut self, ident: usize, cb: impl FnOnce(Js) + 'static) {
        let boxed_cb = Box::new(cb);
        self.callback_queue.insert(ident, boxed_cb);
    }

    pub fn register_io(&mut self, token: usize, cb: impl FnOnce(Js) + 'static) {
        self.add_callback(token, cb);

        print(format!("Event with id: {} registered.", token));
        self.epoll_event_cb_map.insert(token as i64, token);
        self.pending_events += 1;
        self.epoll_pending_events += 1;
    }

    pub fn register_work(
        &mut self,
        task: impl Fn() -> Js + Send + 'static,
        kind: ThreadPoolEventKind,
        cb: impl FnOnce(Js) + 'static,
    ) {
        let callback_id = self.generate_cb_identity();
        self.add_callback(callback_id, cb);

        let event = Event {
            task: Box::new(task),
            callback_id,
            kind,
        };

        // we are not going to implement a real scheduler here, just a LIFO queue
        let available = self.get_available_thread();
        self.thread_pool[available].sender.send(event).expect("register work");
        self.pending_events += 1;
    }

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