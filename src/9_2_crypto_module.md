# Crypto module

As you'll soon understand, we won't actually cover cryptography here, but we'll
simulate a CPU intensive operation that would block if not run on the threadpool.

Without further ado: The Glorious Crypto Module

```rust, no_run
struct Crypto;
impl Crypto {
    fn encrypt(n: usize, cb: impl Fn(Js) + 'static + Clone) {
        let work = move || {
            fn fibonacchi(n: usize) -> usize {
                match n {
                    0 => 0,
                    1 => 1,
                    _ => fibonacchi(n - 1) + fibonacchi(n - 2),
                }
            }

            let fib = fibonacchi(n);
            Js::Int(fib)
        };

        let rt = unsafe { &mut *RUNTIME };
        rt.register_work(work, ThreadPoolEventKind::Encrypt, cb);
    }
}
```
Well, um, this is disappointing if you're into crypto. I'm sorry. The logic here is not very interesting, we take a number `n` as argument and calculate
the n'th fibonacchi number recursively. A famous and inefficient way of putting your
CPU to work.

The overall function will be familiar and look very similar to what we do in our `Fs` module since this
module also sends work to the `threadpool`.

We create a closure (remember that none of the code in the closure will be
run here, but it will get called in one of our worker threads).

Once that is finished we return the result as `Js::Int(fib)`.

Lastly we dereference our `RUNTIME` and we call `rt.register_work(work, ThreadPoolEventKind::Encrypt, cb)` to register our task with the `threadpool`.

Let's move on, we're very soon at the finish line.
