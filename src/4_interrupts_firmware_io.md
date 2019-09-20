# Interrupts, Firmware and I/O

We're nearing an end of the general CS subjects in the book, and we'll start
to dig our way out of the rabbit hole soon.

This part tries to tie things together and look at how the whole computer works
as a system to handle I/O and concurrency. 

Let's get to it!

## A simplified overview 

Let's go through some of the steps where we imagine that we read from a
network card:

![Simplified Overview](./images/AsyncBasicsSimplified.png)

> **Disclaimer**
> We're making things simple here. This is a rather complex operation but we'll
> choose the operations that interest us and skip some along the way.

## 1. Our code

We register a socket. This happens by issuing a `syscall` to the OS. Depending
on the OS we either get a  `file descriptor` (Macos/Linux) or a `socket` (Windows).

The next step is that we register our interest in `read` events on that socket.

## 2. We register events with the OS
The operating systems we focus on handles this in one of three ways:

1. We tell the operating system that we're interested in `Read` events but we want
to wait for it to happen by `yielding` control over our thread to the OS. The OS
then suspends our thread by storing the register state and switch to some other 
thread. 

From our perspective this will be blocking our thread until we have data to read.

2. We tell the operating system that we're interested in `Read` events but we
just want a handle to a the task which we can `poll` to check if the event is
ready or not. The OS will not suspend our thread.

3. We tell the operating system that we are probably going to be interested in 
many events, but we want to subscribe to one event queue. When we `poll` this
queue it will block until one or more event occurs.  

> My next book will be about about alternative C since that is a very interesting
> model of handling I/O events that's going to be important later on to understand 
> why Rusts concurrency abstractions are modeled the way they are. Of that reason 
> we won't cover this in detail here.

## 3. The Network Card

> We're skipping some steps here but it's not very vital to our understanding. 

Meanwhile on the network card there is a small microcontroller running 
specialized firmware. We can imagine that this microcontroller is polling in a
busy loop checking if any data is incoming. 

> The exact way the Network Card handles it internals can be different from this
> (and most likely are). The important part is that there is a simple CPU running
> on the network card doing work to check if there is incoming events.

Once the firmware registers incoming data it issues a Harware Interrupt.

## 4. Hardware Interrupt

> This is a very simplified explanation. If you're interested in knowing more
> about how this works I can't recommend Robert Mustacchi's excellent article
[Turtles on the wire: understanding how the OS uses the modern NIC](https://www.joyent.com/blog/virtualizing-nics) enough.

Modern CPUs have a set of `Interrupt Request Lines` for it to handle events that occur from
external devices. A CPU has a fixed set of interrupt lines.

A hardware interrupt is an electrical signal that can occur at _any time_. The 
CPU immediately **interrupts** its normal workflow to handle the interrupt by
saving the state of it's registers and look up the interrupt handler to run in
the Interrupt Descriptor Table

## 5. Interrupt Handler

The [Interrupt Descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table) is a table that in essence points to a handler function for a specific interrupt. The handler function for a Network Card would typically be registered and handled by a `driver` for that card.

> The IDT is not stored on the CPU as it might seem in the diagram. It's located
> in a fixed and know location in main memory. The CPU only holds a pointer to the
> table in one of it's registers.

## 6. Writing the data

This is a step that might vary a lot depending on the CPU and the firmware on the
network card. If the Network Card and the CPU supports [Direct Memory Access](https://en.wikipedia.org/wiki/Direct_memory_access) (which 
I believe is the standard on a normal modern system) the Network Card might write
data directly to a set of buffers the OS already has set up in main memory. In such a
system the `firmware` on the Network Card might issue an `Interrupt` when the data
is written to memory.

The DMAC (Direct Memory Access Controller) is just added since in such a system,
it would control the access to memory. It's not part of the CPU per se as in the
diagram above. It's not too important for us right now so let's move on.

## 7. The driver

The `driver` would normally handle the communication between the OS and the Network Card.
At _some point_ the buffers are filled and the OS gets notified that the data is ready
to be read. 

## 8. Reading the data

Depending on whether we chose alternative A, B or C the OS will:

1. Wake our thread
2. Return `Ready` on the next `poll`
3. Wake the thread and return a `Read` event for the handler we registered.


## Interrupts

As I hinted at above there are two kinds of interrupts:

1. Hardware Interrupts
2. Software Interrupts

They are very different in nature.

### Hardware Interrupts

Hardware interrupts are created by sending an electrical signal through an [Interrupt Request Line (IRQ)](https://en.wikipedia.org/wiki/Interrupt_request_(PC_architecture)#x86_IRQs). These are hardware lines signalling the CPU directly. 

### Software Interrupts

This is interrupts issued from software instead of hardware. As in the case of a hardware interrupt the CPU jumps to the Interrupt Descriptor Table and runs the handler for the specified interrupt.

Let's see how that works in practice.

You remember from earlier when we used a syscall to write a message to `stdout`? Let's see how we can use a software interrupt to do the same:

> If you want a more torough introduction to inline assembly I can refer you to the [relevant chapter in my previous book](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/an-example-we-can-build-upon) if you haven't read it already.

On Linux this would look something like this:

```rust
#![feature(asm)]
fn main() {
    let message = String::from("Hello world from interrupt!\n");
    syscall(message);
}

#[cfg(target_os = "linux")]
fn syscall(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    unsafe {
        asm!("
        mov     $$1, %rax   # system call 1 is write on linux
        mov     $$1, %rdi   # file handle 1 is stdout
        mov     $0, %rsi    # address of string to output
        mov     $1, %rdx    # number of bytes
        syscall             # call kernel, syscall interrupt
    "
        :
        : "r"(msg_ptr), "r"(len)

        )
    }
}
```

> First of all this is not really a software interrupt. Creating a proper software interrupt on x84_64 (64 bit systems) is not easy. On the earlier x86 systems you could do this exact same call (only exchanging the 64 bit `r__` registers with `e__` registers) and calling `int $$0x80` instead of `syscall`. Syscall was added later since a software interrupt is a costly operation. The `syscall` instruction uses [VDSO](http://articles.manugarg.com/systemcallinlinux2_6.html) which is a memory page attached to each process so no context switch is necessary to execute the system call.

For this to work we need to write some inline assembly again. I'll skip the main function since that should be easy to understand now, and focus on the instructions we write to the CPU.

The code to initiate the `write` syscall on Linux is `1` so when we write `$$1` we're writing the literal value 1 to the `rax` register.

> `$$` in inline assembly using the AT&T syntax is how you write a literal value. A single `$` means you're referring to parameter so when we write `$0` we're referring to the first parameter `msg_ptr`.

Coincidentially, placing the value `1` in to the `rdi` register means that we're referring to `stdout` which is the file descriptor we want to write to.

Secondly we pass in the adress of our string buffer and the length of the buffer, and call the `syscall` instruction.


