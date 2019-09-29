# Http module

```rust
struct Http;
impl Http {
    pub fn http_get_slow(url: &str, delay_ms: u32, cb: impl Fn(Js) + 'static + Clone) {
        let rt: &mut Runtime = unsafe { &mut *RUNTIME };
        // Don't worry, http://slowwly.robertomurray.co.uk is a site for simulating a delayed
        // response from a server. Perfect for our use case.
        let mut stream: minimio::TcpStream =
            minimio::TcpStream::connect("slowwly.robertomurray.co.uk:80").unwrap();
            
        let request = format!(
            "GET /delay/{}/url/http://{} HTTP/1.1\r\n\
             Host: slowwly.robertomurray.co.uk\r\n\
             Connection: close\r\n\
             \r\n",
            delay_ms, url
        );

        stream
            .write_all(request.as_bytes())
            .expect("Error writing to stream");

        let token = rt.generate_cb_identity();
        rt.epoll_registrator
            .register(&mut stream, token, minimio::Interests::readable())
            .unwrap();

        let wrapped = move |_n| {
            let mut stream = stream;
            let mut buffer = String::new();
            stream
                .read_to_string(&mut buffer)
                .expect("Stream read error");
            cb(Js::String(buffer));
        };

        rt.register_io(token, wrapped);
    }
}
```