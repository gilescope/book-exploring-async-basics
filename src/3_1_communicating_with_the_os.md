# Communicating with the operating system

Communication with the operating system is done through `System Calls` or 
"syscalls", this is a public API that the operating system provides for 
applications to work. Most of the time these calls are abstracted away for us as 
programmers by the language or the runtime we use. A language like Rust makes it 
trivial to make a syscall though which we will see below.

Now syscalls is an example of something that is absolutely non-portable, but 
often (not always) BSD-family operating systems uses the same syscalls as 
Linux-family of operating system. Often these are referred to as UNIX family of 
operating systems.

Windows on the other hand has nothing in common with the UNIX family and uses 
it's own api, often referred to as WinAPI.

##Syscall example

To get a bit more familiar with syscalls we'll implement a very basic one for 
the three arcitectures: BSD(macos), Linux and Windows.

The syscall we'll implement is the one used when we write something to `stdout`
since that is such a common operation it's interesting to se how it really works:

Fortunately for us in this specific example, the syscall is the same on Linux
and on Macos so we only need to worry if we're on Windows and therefore use the
`#[cfg(not(target_os = "windows"))]` conditional compilation flag. For the
Windows syscall we do the opposite.


## A cross platform Write syscall

You can run this code directly here in the window. However, the Rust playground 
runs on Linux, you'll need to copy the code over to a Windows machine if you 
want to try it out the code for Windows further down.

```rust
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

// and: http://man7.org/linux/man-pages/man2/write.2.html
#[cfg(not(target_os = "windows"))]
#[link(name = "c")]
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize) -> i32;
}

#[cfg(not(target_os = "windows"))]
fn syscall(message: String) -> io::Result<()> {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    let res = unsafe { write(1, msg_ptr, len) };

    if res == -1 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}

```

I'll explain what we just did here. I assume that the `main` method needs no 
comment.

#### Linux and Macos

```rust, no_run, noplaypen
#[link(name = "c")]
```

Every Linux installation comes with a version of `libc` which a C-library for 
communicating with the operating system. Having a `libc` with a consistent API 
means they can change the underlying implementation without braking everyones 
code. This flag tells the compiler to link to the "c" library on the system we'
re compiling for.

```rust, no_run, noplaypen
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize);
}
```

`extern "C"` or only `extern` (C is assumed if nothing is specified) means we're 
linking to specific functions in the "c" library using the "C" calling 
convention. As you'll see on Windows we'll need to change this since it uses a 
different calling convention than the UNIX family.

The function we're linking to needs to have the exact same name, in this case 
`write`. The parameters doesn't need to have the same name but they must be in 
the right order and it's good practice to name them the same as in the library 
you're linking to.

The write function takes a `file descriptor` which in this case is a handle to 
`stdout`, a pointer to a array of `u8` values, and a count of how many values we 
want to read from the buffer.

```rust, no_run, noplaypen
#[cfg(not(target_os = "windows"))]
fn syscall_libc(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    unsafe { write(1, msg_ptr, len) };
}
```
The first thing we do is to get the pointer to the underlying buffer for our 
string. That will be a pointer of type `*const u8` which matches our `buf` 
argument. The length of the message corresponds to the `count` argument.

You might ask how we know that `1` is the file handle to `stdout` and where we 
found that value. You'll notice this a lot when writing syscalls from Rust. 
Usually constants are defined in the C header files which we can't link to, so 
we need to search them up. 1 is always the file descriptor for `stdout` on UNIX 
systems.

A call to a FFI function is always unsafe so we need to use the `unsafe` keyword 
here.


### Syscall on Windows

```rust, no_run, noplaypen
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}

#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {

    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;
    
    let mut output: u32 = 0;
        let handle = unsafe { GetStdHandle(-11) };
        if handle  == -1 {
            return Err(io::Error::last_os_error())
        }

        let res = unsafe { 
            WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null()) 
            };
        if res  == 0 {
            return Err(io::Error::last_os_error());
        }

    assert_eq!(output as usize, len);
    Ok(())
}
``` 

```
The Rust playground, which we use to run our code, is a Linux machine. I 
disabled the possibility to run the windows code here (since it will 
essentially just be skipped on compilation). However, if you have a Windows 
machine, copy the code above and try for yourself.
```

Now, just by looking at the code above you see it starts to get a bit more 
complex, but let's spend som time to go through line by line what we do here as 
well.

```text
#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
```

The first line is just telling the compiler to only compile this if the 
`target_os` is Windows.

The second line is a linker directive, telling the linker we want to link to the 
library `kernel32` (if you ever see an example that links to `user32` that will 
also work).


```rust, no_run, noplaypen
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}
```

First of all, `extern "stdcall"`, tells the compiler that we won't use the `C` 
calling convention but use Windows calling convention called `stdcall`.

The next part is the functions we want to link to. On Windows, we need to link 
to two functions to get this to work: `GetStdHandle` and `WriteConsoleA`. 
`GetStdHandle` retrieves a reference to a standard device like `stdout`.

WriteConsole comes in two flavors, `WriteConsoleW` that takes in Unicode text 
and `WriteConsoleA` that takes ANSI encoded text. 

Now, ANSI encoded text works fine if you only write English text, but as soon as 
you write text in other languages you might need to use special characters that 
are not possible to represent in `ANSI` but is possible in `utf-8` and our code 
will break.

That's why we'll convert our `utf-8` encoded text to `utf-16` encoded Unicode 
text that can represent these characters and use the `WriteConsoleW` function.

```rust, no_run, noplaypen
#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {

    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;
    
    let mut output: u32 = 0;
        let handle = unsafe { GetStdHandle(-11) };
        if handle  == -1 {
            return Err(io::Error::last_os_error())
        }

        let res = unsafe { 
            WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null()) 
            };

        if res  == 0 {
            return Err(io::Error::last_os_error());
        }

    assert_eq!(output, len);
    Ok(())
}
```
The first thing we do is to convert the text to utf-16 encoded text which 
Windows uses. Fortunately Rust has a built in function to encode our `utf-8`
encoded text to `utf-16` code points. `encode_utf16` returns an iterator over 
`u16` code points that we can collect to a `Vec`.

```rust, no_run, noplaypen
let msg: Vec<u16> = message.encode_utf16().collect();
let msg_ptr = msg.as_ptr();
let len = msg.len() as u32;
```

We then get the pointer to the underlaying buffer of our `Vec` and calculate the 
length.

```rust, no_run, noplaypen
let handle = unsafe { GetStdHandle(-11) };
   if handle  == -1 {
       return Err(io::Error::last_os_error())
   }
```

The next is a call to `GetStdHandle`. We pass in the value `-11`. The values we 
need to pass in for the different standard devices is actually documented 
together with the `GetStdHandle` documentation:

- Stdin: -10
- Stdout: -11
- StdErr: -12

Now, we're lucky here, it's not that common that we find this information 
together with the documentation function we call but it's very convenient when 
we do.

What return codes to expect is also documented thoroughly so we handle potential 
errors here in the same way as we did for the Linux/Macos syscalls.

```rust, no_run, noplaypen
let res = unsafe { 
    WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null()) 
    };

if res  == 0 {
    return Err(io::Error::last_os_error());
}
```

Next up is the call to the `WriteConsoleW` function. Now that we have explained 
everything else there is nothing too fancy about this.

### A note about complexity
There is a lot of "hidden" complexity when writing cross platform code at this 
level. One hurdle is to get something working, which can prove to be quite a 
challenge. Getting it to work **correctly** and **safely** while covering edge 
cases is an additional challenge. 

Are we 100% sure that all valid `utf-8` code points which we use in Rust is valid 
`utf-16` encoded Unicode that Windows will display correctly?

I think so, but being 100 % sure about [this might not be as easy as one might think](https://en.wikipedia.org/wiki/Comparison_of_Unicode_encodings).


# Our finished cross platform syscall

```rust
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

// and: http://man7.org/linux/man-pages/man2/write.2.html
#[cfg(not(target_os = "windows"))]
#[link(name = "c")]
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize) -> i32;
}

#[cfg(not(target_os = "windows"))]
fn syscall(message: String) -> io::Result<()> {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    let res = unsafe { write(1, msg_ptr, len) };

    if res == -1 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}

#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}

#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {

    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;

    let mut output: u32 = 0;
        let handle = unsafe { GetStdHandle(-11) };
        if handle  == -1 {
            return Err(io::Error::last_os_error())
        }

        let res = unsafe { 
            WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null()) 
            };

        if res  == 0 {
            return Err(io::Error::last_os_error());
        }

    assert_eq!(output, len);
    Ok(())
}
```