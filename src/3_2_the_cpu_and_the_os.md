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