# Implementing the Node Eventloop

Now we've finally come to the part of this book where we will write som more code.

The Node event loop is a complex piece of software developed over many years. We
will have to simplify things a lot. There are many edge cases that we won't cover,
and there are many intricacies that we'll not mention or implement. 

However, I will try to implement the parts that are important for us to understand Node better and most importantly use it as an example where we can use our knowledge from the previous chapters to make something that actually works.

What we will do is to look at how it conceptually works. Our main goal here is
to explore async concepts, using Node as an example is part curiosity and part
for fun.

We want to write something like this:

```rust
/// Think of this function as the javascript program you have written
fn javascript() {
    print("First call to read test.txt");
    Fs::read("test.txt", |result| {
        let text = result.into_string().unwrap();
        let len = text.len();
        print(format!("First count: {} characters.", len));

        print(r#"I want to create a "magic" number based on the text."#);
        Crypto::encrypt(text.len(), |result| {
            let n = result.into_int().unwrap();
            print(format!(r#""Encrypted" number is: {}"#, n));
        })
    });

    print("Registering immediate timeout 1");
    set_timeout(0, |_res| {
        print("Immediate1 timed out");
    });
    print("Registering immediate timeout 2");
    set_timeout(0, |_res| {
        print("Immediate2 timed out");
    });

    // let's read the file again and display the text
    print("Second call to read test.txt");
    Fs::read("test.txt", |result| {
        let text = result.into_string().unwrap();
        let len = text.len();
        print(format!("Second count: {} characters.", len));

        // aaand one more time but not in parallel.
        print("Third call to read test.txt");
        Fs::read("test.txt", |result| {
            let text = result.into_string().unwrap();
            print_content(&text, "file read");
        });
    });

    print("Registering a 3000 and a 500 ms timeout");
    set_timeout(3000, |_res| {
        print("3000ms timer timed out");
        set_timeout(500, |_res| {
            print("500ms timer(nested) timed out");
        });
    });

    print("Registering a 1000 ms timeout");
    set_timeout(1000, |_res| {
        print("SETTIMEOUT");
    });

    // `http_get_slow` let's us define a latency we want to simulate
    print("Registering http get request to google.com");
    Io::http_get_slow("http//www.google.com", 2000, |result| {
        let result = result.into_string().unwrap();
        print_content(result.trim(), "web call");
    });
}

fn main() {
    let rt = Runtime::new();
    rt.run(javascript);
}
#
#// ===== THIS IS OUR "NODE LIBRARY" =====
#use std::collections::{BTreeMap, HashMap};
#use std::fmt;
#use std::fs;
#use std::io::{self, Read, Write};
#use std::sync::mpsc::{channel, Receiver, Sender};
#use std::sync::{Arc, Mutex};
#use std::thread::{self, JoinHandle};
#use std::time::{Duration, Instant};
#
#const DEFAULT_TIMEOUT_MS: Option<i32> = None;
#static mut RUNTIME: usize = 0;
#
#struct Event {
#    task: Box<dyn Fn() -> Js + Send + 'static>,
#    callback_id: usize,
#    kind: EventKind,
#}
#
#impl Event {
#    fn close() -> Self {
#        Event {
#            task: Box::new(|| Js::Undefined),
#            callback_id: 0,
#            kind: EventKind::Close,
#        }
#    }
#}
#
#pub enum EventKind {
#    FileRead,
#    Encrypt,
#    Close,
#}
#
#impl fmt::Display for EventKind {
#    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
#        use EventKind::*;
#        match self {
#            FileRead => write!(f, "File read"),
#            Encrypt => write!(f, "Encrypt"),
#            Close => write!(f, "Close"),
#        }
#    }
#}
#
##[derive(Debug)]
#pub enum Js {
#    Undefined,
#    String(String),
#    Int(usize),
#}
#
#impl Js {
#    /// Convenience method since we know the types
#    fn into_string(self) -> Option<String> {
#        match self {
#            Js::String(s) => Some(s),
#            _ => None,
#        }
#    }
#
#    /// Convenience method since we know the types
#    fn into_int(self) -> Option<usize> {
#        match self {
#            Js::Int(n) => Some(n),
#            _ => None,
#        }
#    }
#}
#
#/// NodeTheread represents a thread in our threadpool. Each event has a Joinhandle
#/// and a transmitter part of a channel which is used to inform our main loop
#/// about what events has occurred.
##[derive(Debug)]
#struct NodeThread {
#    pub(crate) handle: JoinHandle<()>,
#    sender: Sender<Event>,
#}
#
#pub struct Runtime {
#    /// Available threads for the threadpool
#    available_threads: Vec<usize>,
#    /// Callbacks scheduled to run
#    callbacks_to_run: Vec<(usize, Js)>,
#    /// All registered callbacks
#    callback_queue: HashMap<usize, Box<dyn FnOnce(Js)>>,
#    /// Maps an epoll event to a callback
#    epoll_event_cb_map: HashMap<i64, usize>,
#    /// Number of pending epoll events, only used by us to print for this example
#    epoll_pending_events: usize,
#    /// Our event registrator which registers interest in events with the OS
#    epoll_registrator: minimio::Registrator,
#    // The handle to our epoll thread
#    epoll_thread: thread::JoinHandle<()>,
#    /// None = infinite, Some(n) = timeout in n ms, Some(0) = immidiate
#    epoll_timeout: Arc<Mutex<Option<i32>>>,
#    /// Channel used by both our threadpool and our epoll thread to send events
#    /// to the main loop
#    event_reciever: Receiver<PollEvent>,
#    /// Creates an unique identity for our callbacks
#    identity_token: usize,
#    /// The number of events pending. When this is zero, we're done
#    pending_events: usize,
#    /// Handles to our threads in the threadpool
#    thread_pool: Vec<NodeThread>,
#    /// Holds all our timers, and an Id for the callback to run once they expire
#    timers: BTreeMap<Instant, usize>,
#    /// A struct to temporarely hold timers to remove. We let Runtinme have
#    /// ownership so we can reuse the same memory
#    timers_to_remove: Vec<Instant>,
#}
#
#/// Describes the three main events our eventloop handles
#enum PollEvent {
#    Threadpool((usize, usize, Js)),
#    Epoll(usize),
#    Timeout,
#}
#
#impl Runtime {
#    pub fn new() -> Self {
#        // ===== THE REGULAR THREADPOOL =====
#        let (event_sender, event_reciever) = channel::<PollEvent>();
#        let mut threads = Vec::with_capacity(4);
#        for i in 0..4 {
#            let (evt_sender, evt_reciever) = channel::<Event>();
#            let event_sender = event_sender.clone();
#            let handle = thread::Builder::new()
#                .name(format!("pool{}", i))
#                .spawn(move || {
#                    while let Ok(event) = evt_reciever.recv() {
#                        print(format!("recived a task of type: {}", event.kind));
#                        // check if we're closing the loop
#                        if let EventKind::Close = event.kind {
#                            break;
#                        };
#
#                        let res = (event.task)();
#                        print(format!("finished running a task of type: {}.", event.kind));
#
#                        let event = PollEvent::Threadpool((i, event.callback_id, res));
#                        event_sender.send(event).unwrap();
#                    }
#                })
#                .expect("Couldn't initialize thread pool.");
#
#            let node_thread = NodeThread {
#                handle,
#                sender: evt_sender,
#            };
#
#            threads.push(node_thread);
#        }
#
#        // ===== EPOLL THREAD =====
#        let mut poll = minimio::Poll::new().expect("Error creating epoll queue");
#        let registrator = poll.registrator();
#        let epoll_timeout = Arc::new(Mutex::new(DEFAULT_TIMEOUT_MS));
#        let epoll_timeout_clone = epoll_timeout.clone();
#
#        let epoll_thread = thread::Builder::new()
#            .name("epoll".to_string())
#            .spawn(move || {
#                let mut events = minimio::Events::with_capacity(1024);
#                loop {
#                    let epoll_timeout_handle = epoll_timeout_clone.lock().unwrap();
#                    let timeout = *epoll_timeout_handle;
#                    drop(epoll_timeout_handle);
#
#                    match poll.poll(&mut events, timeout) {
#                        Ok(v) if v > 0 => {
#                            for i in 0..v {
#                                let event = events.get_mut(i).expect("No events in event list.");
#                                print(format!("epoll event {} is ready", event.id().value()));
#                                let event = PollEvent::Epoll(event.id().value() as usize);
#                                event_sender.send(event).unwrap();
#                            }
#                        }
#                        Ok(v) if v == 0 => {
#                            print("epoll event timeout is ready");
#                            event_sender.send(PollEvent::Timeout).unwrap()
#                        }
#                        Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {
#                            print("recieved event of type: Close");
#                            break;
#                        }
#                        Err(e) => panic!("{:?}", e),
#                        _ => (),
#                    }
#                }
#            })
#            .expect("Error creating epoll thread");
#
#        Runtime {
#            available_threads: (0..4).collect(),
#            callbacks_to_run: vec![],
#            callback_queue: HashMap::new(),
#            epoll_event_cb_map: HashMap::new(),
#            epoll_pending_events: 0,
#            epoll_registrator: registrator,
#            epoll_thread,
#            epoll_timeout,
#            event_reciever,
#            identity_token: 0,
#            pending_events: 0,
#            thread_pool: threads,
#            timers: BTreeMap::new(),
#            timers_to_remove: vec![],
#        }
#    }
#
#    /// This is the event loop. There are several things we could do here to
#    /// make it a better implementation. One is to set a max backlog of callbacks
#    /// to execute in a single tick, so we don't starve the threadpool or file
#    /// handlers. Another is to dynamically decide if/and how long the thread
#    /// could be allowed to be parked for example by looking at the backlog of
#    /// events, and if there is any backlog disable it. Some of our Vec's will
#    /// only grow, and not resize, so if we have a period of very high load, the
#    /// memory will stay higher than we need until a restart. This could be
#    /// dealt by using a different kind of data structure like a `LinkedList`.
#    pub fn run(mut self, f: impl Fn()) {
#        let rt_ptr: *mut Runtime = &mut self;
#        unsafe { RUNTIME = rt_ptr as usize };
#        let mut ticks = 0; // just for us priting out during execution
#
#        // First we run our "main" function
#        f();
#
#        // ===== EVENT LOOP =====
#        while self.pending_events > 0 {
#            ticks += 1;
#            // NOT PART OF LOOP, JUST FOR US TO SEE WHAT TICK IS EXCECUTING
#            print(format!("===== TICK {} =====", ticks));
#
#            // ===== 2. TIMERS =====
#            self.process_expired_timers();
#
#            // ===== 2. CALLBACKS =====
#            // Timer callbacks and if for some reason we have postponed callbacks
#            // to run on the next tick. Not possible in our implementation though.
#            self.run_callbacks();
#
#            // ===== 3. IDLE/PREPARE =====
#            // we won't use this
#
#            // ===== 4. POLL =====
#            // First we need to check if we have any outstanding events at all
#            // and if not we're finished. If not we will wait forever.
#            if self.pending_events == 0 {
#                break;
#            }
#
#            // We want to get the time to the next timeout (if any) and we
#            // set the timeout of our epoll wait to the same as the timeout
#            // for the next timer. If there is none, we set it to infinite (None)
#            let next_timeout = self.get_next_timer();
#
#            // We release the lock before we wait
#            let mut epoll_timeout_lock = self.epoll_timeout.lock().unwrap();
#            *epoll_timeout_lock = next_timeout;
#            drop(epoll_timeout_lock);
#
#            // We handle one and one event but multiple events could be returned
#            // on the same poll. We won't cover that here though but there are
#            // several ways of handling this.
#            if let Ok(event) = self.event_reciever.recv() {
#                match event {
#                    PollEvent::Timeout => (),
#                    PollEvent::Threadpool((thread_id, callback_id, data)) => {
#                        self.process_threadpool_events(thread_id, callback_id, data);
#                    }
#                    PollEvent::Epoll(event_id) => {
#                        self.process_epoll_events(event_id);
#                    }
#                }
#            }
#            self.run_callbacks();
#
#            // ===== 5. CHECK =====
#            // an set immidiate function could be added pretty easily but we
#            // won't do that here
#
#            // ===== 6. CLOSE CALLBACKS ======
#            // Release resources, we won't do that here, but this is typically
#            // where sockets etc are closed.
#        }
#
#        // We clean up our resources, makes sure all destructors runs.
#        for thread in self.thread_pool.into_iter() {
#            thread.sender.send(Event::close()).unwrap();
#            thread.handle.join().unwrap();
#        }
#        self.epoll_registrator.close_loop().unwrap();
#        self.epoll_thread.join().unwrap();
#        print("FINISHED");
#    }
#
#    fn process_expired_timers(&mut self) {
#        // Need an intermediate variable to please the borrowchecker
#        let timers_to_remove = &mut self.timers_to_remove;
#
#        self.timers
#            .range(..=Instant::now())
#            .for_each(|(k, _)| timers_to_remove.push(*k));
#
#        while let Some(key) = self.timers_to_remove.pop() {
#            let callback_id = self.timers.remove(&key).unwrap();
#            self.callbacks_to_run.push((callback_id, Js::Undefined));
#        }
#    }
#
#    fn get_next_timer(&self) -> Option<i32> {
#        self.timers.iter().nth(0).map(|(&instant, _)| {
#            let mut time_to_next_timeout = instant - Instant::now();
#            if time_to_next_timeout < Duration::new(0, 0) {
#                time_to_next_timeout = Duration::new(0, 0);
#            }
#            time_to_next_timeout.as_millis() as i32
#        })
#    }
#
#    fn run_callbacks(&mut self) {
#        while let Some((callback_id, data)) = self.callbacks_to_run.pop() {
#            let cb = self.callback_queue.remove(&callback_id).unwrap();
#            cb(data);
#            self.pending_events -= 1;
#        }
#    }
#
#    fn process_epoll_events(&mut self, event_id: usize) {
#        let id = self
#            .epoll_event_cb_map
#            .get(&(event_id as i64))
#            .expect("Event not in event map.");
#
#        let callback_id = *id;
#        self.epoll_event_cb_map.remove(&(event_id as i64));
#
#        self.callbacks_to_run.push((callback_id, Js::Undefined));
#        self.epoll_pending_events -= 1;
#    }
#
#    fn process_threadpool_events(&mut self, thread_id: usize, callback_id: usize, data: Js) {
#        self.callbacks_to_run.push((callback_id, data));
#        self.available_threads.push(thread_id);
#    }
#
#    fn get_available_thread(&mut self) -> usize {
#        match self.available_threads.pop() {
#            Some(thread_id) => thread_id,
#            // We would normally return None and the request and not panic!
#            None => panic!("Out of threads."),
#        }
#    }
#
#    /// If we hit max we just wrap around
#    fn generate_identity(&mut self) -> usize {
#        self.identity_token = self.identity_token.wrapping_add(1);
#        self.identity_token
#    }
#
#    fn generate_cb_identity(&mut self) -> usize {
#        let ident = self.generate_identity();
#        let taken = self.callback_queue.contains_key(&ident);
#
#        // if there is a collision or the identity is already there we loop until we find a new one
#        // we don't cover the case where there are `usize::MAX` number of callbacks waiting since
#        // that if we're fast and queue a new event every nanosecond that will still take 585.5 years
#        // to do on a 64 bit system.
#        if !taken {
#            ident
#        } else {
#            loop {
#                let possible_ident = self.generate_identity();
#                if self.callback_queue.contains_key(&possible_ident) {
#                    break possible_ident;
#                }
#            }
#        }
#    }
#
#    /// Adds a callback to the queue and returns the key
#    fn add_callback(&mut self, ident: usize, cb: impl FnOnce(Js) + 'static) {
#        let boxed_cb = Box::new(cb);
#        self.callback_queue.insert(ident, boxed_cb);
#    }
#
#    pub fn register_io(&mut self, token: usize, cb: impl FnOnce(Js) + 'static) {
#        self.add_callback(token, cb);
#
#        print(format!("Event with id: {} registered.", token));
#        self.epoll_event_cb_map.insert(token as i64, token);
#        self.pending_events += 1;
#        self.epoll_pending_events += 1;
#    }
#
#    pub fn register_work(
#        &mut self,
#        task: impl Fn() -> Js + Send + 'static,
#        kind: EventKind,
#        cb: impl FnOnce(Js) + 'static,
#    ) {
#        let callback_id = self.generate_cb_identity();
#        self.add_callback(callback_id, cb);
#
#        let event = Event {
#            task: Box::new(task),
#            callback_id,
#            kind,
#        };
#
#        // we are not going to implement a real scheduler here, just a LIFO queue
#        let available = self.get_available_thread();
#        self.thread_pool[available].sender.send(event).unwrap();
#        self.pending_events += 1;
#    }
#
#    fn set_timeout(&mut self, ms: u64, cb: impl Fn(Js) + 'static) {
#        // Is it theoretically possible to get two equal instants? If so we will have a bug...
#        let now = Instant::now();
#        let cb_id = self.generate_cb_identity();
#        self.add_callback(cb_id, cb);
#        let timeout = now + Duration::from_millis(ms);
#        self.timers.insert(timeout, cb_id);
#        self.pending_events += 1;
#        print(format!("Registered timer event id: {}", cb_id));
#    }
#}
#
#pub fn set_timeout(ms: u64, cb: impl Fn(Js) + 'static) {
#    let rt = unsafe { &mut *(RUNTIME as *mut Runtime) };
#    rt.set_timeout(ms, cb);
#}
#
#// ===== THIS IS PLUGINS CREATED IN C++ FOR THE NODE RUNTIME OR PART OF THE RUNTIME ITSELF =====
#// The pointer dereferencing of our runtime is not striclty needed but is mostly for trying to
#// emulate a bit of the same feeling as when you use modules in javascript. We could pass the runtime in
#// as a reference to our startup function.
#
#struct Crypto;
#impl Crypto {
#    fn encrypt(n: usize, cb: impl Fn(Js) + 'static + Clone) {
#        let work = move || {
#            fn fibonacchi(n: usize) -> usize {
#                match n {
#                    0 => 0,
#                    1 => 1,
#                    _ => fibonacchi(n - 1) + fibonacchi(n - 2),
#                }
#            }
#
#            let fib = fibonacchi(n);
#            Js::Int(fib)
#        };
#
#        let rt = unsafe { &mut *(RUNTIME as *mut Runtime) };
#        rt.register_work(work, EventKind::Encrypt, cb);
#    }
#}
#
#struct Fs;
#impl Fs {
#    fn read(path: &'static str, cb: impl Fn(Js) + 'static) {
#        let work = move || {
#            // Let's simulate that there is a very large file we're reading allowing us to actually
#            // observe how the code is executed
#            thread::sleep(std::time::Duration::from_secs(1));
#            let mut buffer = String::new();
#            fs::File::open(&path)
#                .unwrap()
#                .read_to_string(&mut buffer)
#                .unwrap();
#            Js::String(buffer)
#        };
#        let rt = unsafe { &mut *(RUNTIME as *mut Runtime) };
#        rt.register_work(work, EventKind::FileRead, cb);
#    }
#}
#
#struct Io;
#impl Io {
#    pub fn http_get_slow(url: &str, delay_ms: u32, cb: impl Fn(Js) + 'static + Clone) {
#        let rt: &mut Runtime = unsafe { &mut *(RUNTIME as *mut Runtime) };
#        // Don't worry, http://slowwly.robertomurray.co.uk is a site for simulating a delayed
#        // response from a server. Perfect for our use case.
#        let mut stream: minimio::TcpStream =
#            minimio::TcpStream::connect("slowwly.robertomurray.co.uk:80").unwrap();
#        let request = format!(
#            "GET /delay/{}/url/http://{} HTTP/1.1\r\n\
#             Host: slowwly.robertomurray.co.uk\r\n\
#             Connection: close\r\n\
#             \r\n",
#            delay_ms, url
#        );
#
#        stream
#            .write_all(request.as_bytes())
#            .expect("Error writing to stream");
#
#        let token = rt.generate_cb_identity();
#        rt.epoll_registrator
#            .register(&mut stream, token, minimio::Interests::readable())
#            .unwrap();
#
#        let wrapped = move |_n| {
#            let mut stream = stream;
#            let mut buffer = String::new();
#            stream
#                .read_to_string(&mut buffer)
#                .expect("Stream read error");
#
#            cb(Js::String(buffer));
#        };
#
#        rt.register_io(token, wrapped);
#    }
#}
#
#fn print(t: impl std::fmt::Display) {
#    println!("Thread: {}\t {}", current(), t);
#}
#
#fn print_content(t: impl std::fmt::Display, descr: &str) {
#    println!(
#        "\n===== THREAD {} START CONTENT - {} =====",
#        current(),
#        descr.to_uppercase()
#    );
#    println!("{}", t);
#    println!("===== END CONTENT =====\n");
#}
#
#fn current() -> String {
#    thread::current().name().unwrap().to_string()
#}
#
#// ===== MINIMIO - ONLY LINUX FOR PLAYGORUND =====
#mod minimio {
#    use std::io;
#    use std::sync::{
#        atomic::{AtomicBool, AtomicUsize, Ordering},
#        Arc,
#    };
#
#    #[cfg(target_os = "linux")]
#    pub use linux::{Event, Registrator, Selector, TcpStream};
#
#    pub type Events = Vec<Event>;
#    static TOKEN: Token = Token(AtomicUsize::new(0));
#
#    pub struct Token(AtomicUsize);
#    impl Token {
#        pub fn next(&self) -> usize {
#            self.0.fetch_add(1, Ordering::Relaxed)
#        }
#
#        pub fn value(&self) -> usize {
#            self.0.load(Ordering::Relaxed)
#        }
#
#        pub fn new(val: usize) -> Self {
#            Token(AtomicUsize::new(val))
#        }
#    }
#
#    impl std::cmp::PartialEq for Token {
#        fn eq(&self, other: &Self) -> bool {
#            self.value() == other.value()
#        }
#    }
#
#    #[derive(Debug)]
#    pub struct Poll {
#        registry: Registry,
#        is_poll_dead: Arc<AtomicBool>,
#    }
#
#    #[derive(Debug)]
#    pub struct Registry {
#        selector: Selector,
#    }
#
#    impl Poll {
#        pub fn new() -> io::Result<Poll> {
#            Selector::new().map(|selector| Poll {
#                registry: Registry { selector },
#                is_poll_dead: Arc::new(AtomicBool::new(false)),
#            })
#        }
#
#        pub fn registrator(&self) -> Registrator {
#            self.registry
#                .selector
#                .registrator(self.is_poll_dead.clone())
#        }
#
#        // We require a `&mut TcpStram` here, but we only need it for Windows. Now
#        // there are ways to deal with this, but either way we need to in reality
#        // mutate our `TcpSocket` on Windows the way we have designed this. If we
#        // implement this as part of a Runtime which we control there are ways for
#        // us to guarantee that the buffer is not touched or moved, so we could
#        // mutate it without the user knowing safely, but this is not a Runtime so
#        // we can't know that.
#        pub fn register_with_id(
#            &self,
#            stream: &mut TcpStream,
#            interests: Interests,
#            token: usize,
#        ) -> io::Result<Token> {
#            self.registry
#                .selector
#                .registrator(self.is_poll_dead.clone())
#                .register(stream, token, interests)?;
#            Ok(Token::new(token))
#        }
#
#        pub fn register(&self, stream: &mut TcpStream, interests: Interests) -> io::Result<Token> {
#            let token = TOKEN.next();
#            self.register_with_id(stream, interests, token)
#        }
#
#        /// Polls the event loop. The thread yields to the OS while witing for either
#        /// an event to retur or a timeout to occur. A negative timeout will be treated
#        /// as a timeout of 0.
#        pub fn poll(&mut self, events: &mut Events, timeout_ms: Option<i32>) -> io::Result<usize> {
#            // A negative timout is converted to a 0 timeout
#            let timeout = timeout_ms.map(|n| if n < 0 { 0 } else { n });
#            loop {
#                let res = self.registry.selector.select(events, timeout);
#                match res {
#                    Ok(()) => break,
#                    Err(ref e) if e.kind() == io::ErrorKind::Interrupted => (),
#                    Err(e) => return Err(e),
#                };
#            }
#
#            if self.is_poll_dead.load(Ordering::SeqCst) {
#                return Err(io::Error::new(io::ErrorKind::Interrupted, "Poll closed."));
#            }
#
#            Ok(events.len())
#        }
#    }
#
#    pub const WRITABLE: u8 = 0b0000_0001;
#    pub const READABLE: u8 = 0b0000_0010;
#
#    pub struct Interests(u8);
#    impl Interests {
#        pub fn readable() -> Self {
#            Interests(READABLE)
#        }
#    }
#    impl Interests {
#        pub fn is_readable(&self) -> bool {
#            self.0 & READABLE != 0
#        }
#
#        pub fn is_writable(&self) -> bool {
#            self.0 & WRITABLE != 0
#        }
#    }
#
#    mod linux {
#        use super::{Events, Interests, Token, TOKEN};
#        use std::io::{self, IoSliceMut, Read, Write};
#        use std::net;
#        use std::os::unix::io::{AsRawFd, RawFd};
#        use std::sync::{
#            atomic::{AtomicBool, Ordering},
#            Arc,
#        };
#
#        pub struct Registrator {
#            fd: RawFd,
#            is_poll_dead: Arc<AtomicBool>,
#        }
#
#        impl Registrator {
#            pub fn register(
#                &self,
#                stream: &TcpStream,
#                token: usize,
#                interests: Interests,
#            ) -> io::Result<()> {
#                if self.is_poll_dead.load(Ordering::SeqCst) {
#                    return Err(io::Error::new(
#                        io::ErrorKind::Interrupted,
#                        "Poll instance closed.",
#                    ));
#                }
#                let fd = stream.as_raw_fd();
#                if interests.is_readable() {
#                    // We register the id (or most oftenly referred to as a Token) to the `udata` field
#                    // if the `Kevent`
#                    let mut event = ffi::Event::new(ffi::EPOLLIN | ffi::EPOLLONESHOT, token);
#                    epoll_ctl(self.fd, ffi::EPOLL_CTL_ADD, fd, &mut event)?;
#                };
#
#                if interests.is_writable() {
#                    unimplemented!();
#                }
#
#                Ok(())
#            }
#
#            pub fn close_loop(&self) -> io::Result<()> {
#                if self
#                    .is_poll_dead
#                    .compare_and_swap(false, true, Ordering::SeqCst)
#                {
#                    return Err(io::Error::new(
#                        io::ErrorKind::Interrupted,
#                        "Poll instance closed.",
#                    ));
#                }
#
#                // This is a little hacky but works for our needs right now
#                let wake_fd = eventfd(1, 0)?;
#                let mut event = ffi::Event::new(ffi::EPOLLIN, 0);
#                epoll_ctl(self.fd, ffi::EPOLL_CTL_ADD, wake_fd, &mut event)?;
#
#                Ok(())
#            }
#        }
#
#        #[derive(Debug)]
#        pub struct Selector {
#            id: usize,
#            fd: RawFd,
#        }
#
#        impl Selector {
#            fn new_with_id(id: usize) -> io::Result<Self> {
#                println!("EPOLL CREATE");
#                Ok(Selector {
#                    id,
#                    fd: epoll_create()?,
#                })
#            }
#
#            pub fn new() -> io::Result<Self> {
#                Selector::new_with_id(TOKEN.next())
#            }
#
#            pub fn id(&self) -> usize {
#                self.id
#            }
#
#            /// This function blocks and waits until an event has been recieved. `timeout` None means
#            /// the poll will never time out.
#            pub fn select(&self, events: &mut Events, timeout_ms: Option<i32>) -> io::Result<()> {
#                events.clear();
#                let timeout = timeout_ms.unwrap_or(-1);
#                epoll_wait(self.fd, events, 1024, timeout).map(|n_events| {
#                    // This is safe because `syscall_kevent` ensures that `n_events` are
#                    // assigned. We could check for a valid token for each event to verify so this is
#                    // just a performance optimization used in `mio` and copied here.
#                    unsafe { events.set_len(n_events as usize) };
#                })
#            }
#
#            pub fn registrator(&self, is_poll_dead: Arc<AtomicBool>) -> Registrator {
#                Registrator {
#                    fd: self.fd,
#                    is_poll_dead,
#                }
#            }
#        }
#
#        impl Drop for Selector {
#            fn drop(&mut self) {
#                match close_fd(self.fd) {
#                    Ok(..) => (),
#                    Err(e) => {
#                        if !std::thread::panicking() {
#                            panic!(e);
#                        }
#                    }
#                }
#            }
#        }
#
#        pub type Event = ffi::Event;
#        impl Event {
#            pub fn id(&self) -> Token {
#                Token::new(self.data().as_usize())
#            }
#        }
#
#        pub struct TcpStream {
#            inner: net::TcpStream,
#        }
#
#        impl TcpStream {
#            pub fn connect(adr: impl net::ToSocketAddrs) -> io::Result<Self> {
#                // actually we should set this to non-blocking before we call connect which is not something
#                // we get from the stdlib but could do with a syscall. Let's skip that step in this example.
#                // In other words this will block shortly establishing a connection to the remote server
#                let stream = net::TcpStream::connect(adr)?;
#                stream.set_nonblocking(true)?;
#
#                Ok(TcpStream { inner: stream })
#            }
#        }
#
#        impl Read for TcpStream {
#            fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
#                // If we let the socket operate non-blocking we could get an error of kind `WouldBlock`,
#                // that means there is more data to read but we would block if we waited for it to arrive.
#                // The right thing to do is to re-register the event, getting notified once more
#                // data is available. We'll not do that in our implementation since we're making an example
#                // and instead we make the socket blocking again while we read from it
#                self.inner.set_nonblocking(false)?;
#
#                (&self.inner).read(buf)
#            }
#
#            /// Copies data to fill each buffer in order, with the final buffer possibly only beeing
#            /// partially filled. Now as we'll see this is like it's made for our use case when abstracting
#            /// over IOCP AND epoll/kqueue (since we need to buffer anyways).
#            ///
#            /// IoSliceMut is like `&mut [u8]` but it's guaranteed to be ABI compatible with the `iovec`
#            /// type on unix platforms and `WSABUF` on Windows. Perfect for us.
#            fn read_vectored(&mut self, bufs: &mut [IoSliceMut]) -> io::Result<usize> {
#                (&self.inner).read_vectored(bufs)
#            }
#        }
#
#        impl Write for TcpStream {
#            fn write(&mut self, buf: &[u8]) -> io::Result<usize> {
#                self.inner.write(buf)
#            }
#
#            fn flush(&mut self) -> io::Result<()> {
#                self.inner.flush()
#            }
#        }
#
#        impl AsRawFd for TcpStream {
#            fn as_raw_fd(&self) -> RawFd {
#                self.inner.as_raw_fd()
#            }
#        }
#
#        mod ffi {
#            use std::io;
#            use std::os::raw::c_void;
#
#            pub const EPOLL_CTL_ADD: i32 = 1;
#            pub const EPOLL_CTL_DEL: i32 = 2;
#            pub const EPOLLIN: i32 = 0x1;
#            pub const EPOLLONESHOT: i32 = 0x40000000;
#
#            /// This is a new structure for us. The Union type in Rust is there mostly to work with C-type unions. A Union is
#            /// like an untyped enum, meaing that `Data` can be *either* a `uint32` or a `uint64` for exaple. It's easiest to think
#            /// of this like a very primitive Enum. The size of a Union is the size of its largest field.
#            /// You can read more about Unions in Rust here: https://doc.rust-lang.org/reference/items/unions.html
#            #[repr(C)]
#            pub union Data {
#                void: *const c_void,
#                fd: i32,
#                uint32: u32,
#                uint64: u64,
#            }
#
#            /// Modelling `Data` Union to a Enum is not needed since we know what value we pass in and therefore what
#            /// value to expect. We know that the size of `Data` is 8 bytes anyway.
#            pub enum EpollData {
#                Void(*const c_void),
#                Fd(i32),
#                Uint32(u32),
#                Uint64(u64),
#            }
#
#            impl EpollData {
#                pub fn as_usize(&self) -> usize {
#                    match self {
#                        EpollData::Void(n) => *n as usize,
#                        EpollData::Fd(n) => *n as usize,
#                        EpollData::Uint32(n) => *n as usize,
#                        EpollData::Uint64(n) => *n as usize,
#                    }
#                }
#            }
#
#            /// Since the same name is used multiple times, it can be confusing but we have an `Event` structure.
#            /// This structure ties a file descriptor and a field called `events` together. The field `events` holds information
#            /// about what events are ready for that file descriptor.
#            #[repr(C)]
#            pub struct Event {
#                /// This can be confusing, but this is the events that are ready on the file descriptor.
#                events: u32,
#                // TODO: Consider if we should just treat this as a usize instead...
#                epoll_data: Data,
#            }
#
#            impl Event {
#                pub fn new(events: i32, id: usize) -> Self {
#                    Event {
#                        events: events as u32,
#                        epoll_data: Data { uint64: id as u64 },
#                    }
#                }
#                pub fn data(&self) -> EpollData {
#                    unsafe {
#                        match self.epoll_data {
#                            Data { void } => EpollData::Void(void),
#                            Data { fd } => EpollData::Fd(fd),
#                            Data { uint32 } => EpollData::Uint32(uint32),
#                            Data { uint64 } => EpollData::Uint64(uint64),
#                        }
#                    }
#                }
#            }
#
#            #[link(name = "c")]
#            extern "C" {
#                /// http://man7.org/linux/man-pages/man2/epoll_create1.2.html
#                pub fn epoll_create(size: i32) -> i32;
#
#                /// http://man7.org/linux/man-pages/man2/close.2.html
#                pub fn close(fd: i32) -> i32;
#
#                /// http://man7.org/linux/man-pages/man2/epoll_ctl.2.html
#                pub fn epoll_ctl(epfd: i32, op: i32, fd: i32, event: *mut Event) -> i32;
#
#                /// http://man7.org/linux/man-pages/man2/epoll_wait.2.html
#                ///
#                /// - epoll_event is a pointer to an array of Events
#                /// - timeout of -1 means indefinite
#                pub fn epoll_wait(
#                    epfd: i32,
#                    events: *mut Event,
#                    maxevents: i32,
#                    timeout: i32,
#                ) -> i32;
#
#                /// http://man7.org/linux/man-pages/man2/timerfd_create.2.html
#                pub fn eventfd(initva: u32, flags: i32) -> i32;
#            }
#        }
#
#        fn epoll_create() -> io::Result<i32> {
#            // Size argument is ignored but must be greater than zero
#            let res = unsafe { ffi::epoll_create(1) };
#            if res < 0 {
#                Err(io::Error::last_os_error())
#            } else {
#                Ok(res)
#            }
#        }
#
#        fn close_fd(fd: i32) -> io::Result<()> {
#            let res = unsafe { ffi::close(fd) };
#            if res < 0 {
#                Err(io::Error::last_os_error())
#            } else {
#                Ok(())
#            }
#        }
#
#        fn epoll_ctl(epfd: i32, op: i32, fd: i32, event: &mut Event) -> io::Result<()> {
#            let res = unsafe { ffi::epoll_ctl(epfd, op, fd, event) };
#            if res < 0 {
#                Err(io::Error::last_os_error())
#            } else {
#                Ok(())
#            }
#        }
#
#        /// Waits for events on the epoll instance to occur. Returns the number file descriptors ready for the requested I/O.
#        /// When successful, epoll_wait() returns the number of file descriptors ready for the requested
#        /// I/O, or zero if no file descriptor became ready during the requested timeout milliseconds
#        fn epoll_wait(
#            epfd: i32,
#            events: &mut [Event],
#            maxevents: i32,
#            timeout: i32,
#        ) -> io::Result<i32> {
#            let res = unsafe { ffi::epoll_wait(epfd, events.as_mut_ptr(), maxevents, timeout) };
#            if res < 0 {
#                Err(io::Error::last_os_error())
#            } else {
#                Ok(res)
#            }
#        }
#
#        fn eventfd(initva: u32, flags: i32) -> io::Result<i32> {
#            let res = unsafe { ffi::eventfd(initva, flags) };
#            if res < 0 {
#                Err(io::Error::last_os_error())
#            } else {
#                Ok(res)
#            }
#        }
#    }
#}

```

And since this is just a runtime we write to learn we get this output showing us what happens when we run that code:


In the following chapters I'll walk you through what's going on here but right from the start we can notice some important things going on here:

The code we write has a lot of simiarities with Javascript and looks very little like idiomatic Rust code.

When you look through the events you'll see they happen in a different order that we wrote them. That tells us that there is some concurrency going on here.

The next chapters will explain everything going on here, so relax, get a cup of tea and stay focused as we explain everything.