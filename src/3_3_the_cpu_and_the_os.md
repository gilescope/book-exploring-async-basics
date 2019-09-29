## 

# Does the CPU and the OS cooperate?

If you would have asked me this question a year ago I would most likely answer
no. The OS operates on top of the CPU and is the "gateway" all other application
uses. Now, first of all, I wouldn't have thought this through, but unless you
learn how this work from the bottom up, it's not easy to know for sure.

What started to make me think I was very wrong was some code looking like this:

> We'll write instructions directly to the CPU so there know there are no 
> abstractions, syscalls or runtime of some sort that might do some magic for us

```rust
#![feature(asm)]
fn main() {
    let t = 100;
    let t_ptr: *const usize = &t;
    let x = dereference(t_ptr);
    
    println!("{}", x);
}

fn dereference(ptr: *const usize) -> usize {
    let res: usize;
    unsafe { 
        asm!("mov ($1), $0":"=r"(res): "r"(ptr)) 
        };
    res
}
```

Here we write a `dereference` function using assembly instructions. We know there
is no way the OS is involved here.

As you see, this code will output `100` as expected. But let's now instead create a 
pointer with the address `99999999999999` which we know is invalid and see what 
happens when we pass that into the same function:

```rust
#![feature(asm)]
fn main() {
    let t = 99999999999999 as *const usize;
    let x = dereference(t);
    
    println!("{}", x);
}
# fn dereference(ptr: *const usize) -> usize {
#     let res: usize;
#     unsafe {
#     asm!("mov ($1), $0":"=r"(res): "r"(ptr));
#     }
# 
#     res
# }
```
Now we get a segmentation fault. Not surprising really, but how does the CPU
know that we're not allowed to dereference this memory?

- Does the CPU ask the OS if this process is allowed to access this memory location?
- Won't that be very slow? 
- How does the CPU know that it has an OS running on top of it at all? D
- Does all CPUs know what a segmentation fault is? 
- Does the CPU have any concept of memory segmentation at all? 
- Why do we get an error message at all and not just a crash?

## Down the rabbit hole

Yes, this is a small rabbit hole. It turns out that there
os some kind of corporation, but maybe not the way you naively would think.

Many modern CPUs provide some infrastructure that Operating Systems need to 
implement and give us the security and stability we expect. Actually, most 
advanced CPUs provide a lot more options than operating systems like Linux, BSD and
Windows actually uses.

There is especially two that I want to address here:

1. How the CPU prevents us from accessing memory we're not supposed to access
2. How the CPU handles asynchronous events like I/O

> If you want to know more about how this works in detail I will absolutely
> recommend that you give [Philipp Oppermann excelent series](https://os.phil-opp.com/)
> a read. It's extremely well written and will answer all these questions and many more.


## How does the CPU prevent us from accessing memory we're not supposed to?

As I mentioned, modern CPUs have already some definition of basic concepts. Some
concepts the CPU is aware of are (there are more but we can't cover everything here):

- Virtual memory 
- Page table
- Page fault
- Exceptions
- [Privelege level](https://en.wikipedia.org/wiki/Protection_ring)

Exactly how this works will differ depending on the exact CPU so we'll treat them 
in general terms here.

Most modern CPUs however has a MMU (Memory Management Unit). This is a part of the
CPU (often etched on the same dye even). The MMUs job is to translate between
the virtual address we use in our programs, to a physical address.

When the OS starts a process (like our program) it sets up a page table for our
process, and makes sure a special register on the CPU points to this page table.

Now, when we try to dereference `t_ptr` in the code above, the address is at some point
sent to the MMU for translation, which looks it up in the page table which translates
it to the physical address.

In this case it will point to a memory address on our stack that holds the value `100`.

When we pass in `99999999999999` and ask it to fetch what's stored at that address 
(which is what dereferencing does) it looks for the translation in the page table but
can't find it.

The CPU then treats this as a `page fault`. 

At boot, the OS at provided the CPU with a Interrupt Descriptor Table. This table
has a predefined format where the OS has to provide handlers for the predefined 
exceptions the CPU can encounter.

Since the OS provided ha pointer to a function that handles `Page Fault` the CPU 
jumps to that function and thereby hands over control to the Operating System. 

The OS then prints a nice message for us letting us know that we encountered 
what it calls a `segmentation fault`. This message will therefore wary depending on the OS you 
run the code on.

## But can't we just change the page table in the CPU?

Now this is where `Privelege Level` comes in. Most modern operating systems operate with two `Ring Levels`. Ring 0, the kernel space, and Ring 3, user space.

![Privelege rings](./images/priv_rings.png)

Most CPUs has a concept of more rings that what most modern operating systems use. This has historical reasons, which is also why `Ring 0` and `Ring 3` are used (and not 1, 2).

Now every entry in the page table has additional information about it, amongst that information is the information about what ring it belongs to. This is set up when your OS boots up,

Code executed in `Ring 0` has almost unrestricted access to external devices, memory and is free to change registers that provide security at the hardware level.

Now, the code you write in `Ring 3` will typically have extremely restricted access to I/O and certain CPU registers (and instructions). Trying to issue an instruction or setting a register from `Ring 3` to change the `page table` will be prevented already at the CPU level, which then treats this as an exception and jumps to the handler provided by the OS.

This is also the reason why you have no other choice than to cooperate with the OS and handle I/O tasks through syscalls. The system wouldn't be very secure if this wasn't the case.
