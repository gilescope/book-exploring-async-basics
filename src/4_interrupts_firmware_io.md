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

Once the firmware registers incoming data it issues a Hardware Interrupt.

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
network card. If the Network Card and the CPU supports [Direct Memory Access](https://en.wikipedia.org/wiki/Direct_memory_access) (which should be the standard on all modern systems today) the Network Card will write data directly to a set of buffers the OS already has set up in main memory. In such a system the `firmware` on the Network Card might issue an `Interrupt` when the data is written to memory. `DMA` is very efficient
since the CPU is only notified when the data is already in memory. On older systems the
CPU needed to devote often substantial time to handle the data transfer from the
network card.

The DMAC (Direct Memory Access Controller) is just added since in such a system,
it would control the access to memory. It's not part of the CPU per se as in the
diagram above. It's we're deep enough in the rabbit hole now and this is not really important for us right now so let's move on.

## 7. The driver

The `driver` would normally handle the communication between the OS and the Network Card.
At _some point_ the buffers are filled, and the network card issues an `Interrupt`. The CPU then jumps to the handler of that interrupt. The interrupt handler for this exact type
of interrupt is registered by the driver, so it's actually the driver that handles this event and in turn informs the kernel that the data is ready to be read. 

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

Hardware interrupts are created by sending an electrical signal through an [Interrupt Request Line (IRQ)](https://en.wikipedia.org/wiki/Interrupt_request_(PC_architecture)#x86_IRQs). These are hardware lines signaling the CPU directly. 

### Software Interrupts

This is interrupts issued from software instead of hardware. As in the case of a hardware interrupt the CPU jumps to the Interrupt Descriptor Table and runs the handler for the specified interrupt.


### Firmware

Firmware doesn't get much attention from most of us, however, they're a crucial part of the world we live in. They run on all kinds of hardware, and has all kinds of strange and peculiar ways to make the computer we program on work.

When I think about firmware, I think about the scenes from Star Wars where they walk into a bar with all kinds of strange and obscure creatures. I imagine the world of firmware is much like this, few us of know what they do or how they work on a particular system.

Now, firmware needs a microcontroller or similar to be able to work. Even the CPU has firmware which makes it work. That means there are many more small "CPUs" on our system than the cores we program against.

Why is this important? Well, you remember that concurrency is all about efficiency right? We'll since we have many CPU's already doing work for us on our system, one of our concerns is to not replicate or duplicate that work when we write code.

If a network card has firmware that continually checks if new data has arrived, it's pretty wasteful if we duplicate that by letting our CPU continually check if new data arrives as well. It's much better if we either check once in a while or even better, gets notified when data has arrived for us.

