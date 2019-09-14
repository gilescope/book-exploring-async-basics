# The Operating System

The operating system stands in the center of everything we do as programmers, so there is no way for us to discuss any kind 
of fundamentals in programming without talking about operating systems in a bit of detail.

## Communicating with the operating system

Communication with the operating system is done through `System Calls` or "syscalls", this is a public API that the 
operating system provides for applications to work. Most of the time these calls are abstracted away for us as programmers 
by the language or the runtime we use. A language like Rust makes it trivial to make a syscall though which we will see below.

Now syscalls is an example of something that is absolutely non-portable, but often (not always) BSD-family operating systems 
uses the same syscalls as Linux-family of operating system. Often these are referred to as UNIX family of operating systems.

Windows on the other hand has nothing in common with the UNIX family and uses it's own api, often referred to as WinAPI.

### Syscall example

To get a bit more familiar with syscalls we'll implement a very basic one for the three arcitectures: BSD(macos), Linux and Windows.

The syscall we'll implement is the one used when we write something to `stdout` since that is such a common operation it's interesting to 
se how it really works:


Fortunately for us in this specific example, the syscall is the same on Linux and on Macos so we only need to worry if we're on 
Windows and therefore use the `#[cfg(not(target_os = "windows"))]` conditional compilation flag. For the Windows syscall we do the opposite.

#### A cross platform Write syscall

You can run this directly here and see it work, but since the Rust playground runs on Linux, you'll need to copy the code over to
a windows machine if you want to try it out.

```rust
fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall_libc(sys_message);
}

#[cfg(not(target_os = "windows"))]
#[link(name = "c")]
extern "C" {
    // http://man7.org/linux/man-pages/man2/write.2.html
    fn write(fd: u32, buf: *const u8, count: usize);
}

#[cfg(not(target_os = "windows"))]
fn syscall_libc(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    unsafe { write(1, msg_ptr, len) };
}

```

I'll explain what we just did here. I assume that the `main` method needs no comment.

```rust, no_run, noplaypen
#[link(name = "c")]
```

Every Linux installation comes with a version of `libc` which a C-library for communicating 
with the operating system. Having a `libc` with a consistent API means they can change the underlying implementation without braking
everyones code. This flag tells the compiler to link to the "c" library on the system we're compiling for.

```rust, no_run, noplaypen
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize);
}
```

`extern "C"` or only `extern` (C is assumed if nothing is specified) means we're linking to specific functions in the "c" library using the "C" calling convention. As you'll see
on Windows we'll need to change this since it uses a different calling convention than the UNIX family.

The function we're linking to needs to have the exact same name, in this case `write`. The parameters doesn't need to have the same 
name but they must be in the right order and it's good practice to name them the same as in the library you're linking to.

The write function takes a `file descriptor` which in this case is a handle to `stdout`, a pointer to a array of `u8` values,
and a count of how many values we want to read from the buffer.

```rust, no_run, noplaypen
#[cfg(not(target_os = "windows"))]
fn syscall_libc(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    unsafe { write(1, msg_ptr, len) };
}
```
The first thing we do is to get the pointer to the underlying buffer for our string. That will be a pointer of type `*const u8`
which matches our `buf` argument. The length of the message corresponds to the `count` argument.

You might ask how we know that `1` is the file handle to `stdout` and where we found that value. You'll notice this a lot when writing syscalls from 
Rust. Usually constants are defined in the C header files which we can't link to, so we need to search them up. 1 is always the file descriptor for
`stdout` on UNIX systems.

A call to a FFI function is always unsafe so we need to use the `unsafe` keyword here.


Now for this simple example we'll only focus on the Unix family syscall. Don't worry, in the next book you'll get more than enough time
with syscalls on all of the three big platforms.


# Threads

Threads are one of the baisc constructs for running code that we programmers create. Like the syscall for outputting text to the console, 
there is a syscall for asking the OS to create a thread. 

Now threads are a bit special, since they are the thing that actually makes us think that we write **synchronous** code at all.

Let's stop for a second and think a bit about the significance of threads for our the problem domain we're investigating here:

The "naive" way of thinking about your code is like this:




Now threads

memory

faking sync

I find it easy to forget is that everything I do goes through the operating system. Of course I know it's there, .

When we ask for memory, like if we call `let my_vec = vec![0_u8;1040]` we actually ask the OS to map some memory for us and hand it over. 

Have you ever wondered how you can write plain assembly, which pretty much is CPU instructions, and execute them, but still get an error if you try to access memory that is not mapped to you?

Let's say you execute the following code:

```rust
#![feature(asm)]
fn main() {
    let t = 100;
    let x = dereference(&t);
    
    println!("{}", x);
}

fn dereference(ptr: *const usize) -> usize {
    let res: usize;
    unsafe {
    // Note: the parenthesis around ($1) means that we want to get the value at 
    // the memory location which $1 is pointing at
    asm!("mov ($1), $0":"=r"(res): "r"(ptr));
    }
 
    res
}
```

As you see, this code will output `100` as expected. But let's instead create a pointer with the address `9999999` which
we know just points somewhere random and see what happens when we pass that into the same function:

```rust
#![feature(asm)]
fn main() {
    let t = 9999999 as *const usize;
    let x = dereference(t);
    
    println!("{}", x);
}
#
# fn dereference(ptr: *const usize) -> usize {
#     let res: usize;
#     unsafe {
#     asm!("mov ($1), $0":"=r"(res): "r"(ptr));
#     }
#  
#     res
# }
```
Now we get a segmentation fault. Not very surprising, and a complicated way of dereferencing a pointer but I wanted to stress 
that we're "talking" directly with the CPU here. There are no checks in Rust, no syscalls or anything happening behind the 
scenes that alerst the OS that we try to access memory we shouldn't access. So how does the CPU know that we can't access this memory location?

This is a sure hint that the CPU does indeed operate in tandem with the operating system, we ask the operating system to map some
memory for us, and somehow the CPU (MMU) know what memory we can access and not.


# Bonus section

So how does the CPU know that we don't have access to the memory referenced in the example above?

Most CPU's ca run in different memory protection levels.

