# What is asynchronous code execution and concurrency?

Concurrency is about **dealing** with a lot of things at the same time. Rob Pike
defines this as "the composition of independently executed processes."

Parallelism is about **doing** a lot of things at the same time.

We handle concurrency by executing tasks/processes in an asynchronous manner. 
Asynchronous code execution is therefore the way we handle concurrency in programming.

## Lets start off with some definitions

### Resource
Something we need to perform work on a task. Our resources is limited.

### Progressing a task
Perform work that requires some kind of resource, whether it's computational power 
from the CPU or it's your brain using its processing power to process something. 

### Parallel
Something happening independently at the **exact** same time.

### Concurrent
Tasks that are "in progress" at the same time, but not neccicarely progressing
simultaneously. In computer programming this most often implies that the 
tasks which run concurrently can be stopped and resumed. 

### Reference frame
The frame of reference when we use when we define what we operate concurrent relative
to. Most often it's the tasks we control in our process, but as you'll see, not
being aware that it's important that we use the same reference frame can cause 
some confusion.


## The mental model I use.

I firmly believe the main reason we find parallel and concurrent programming hard to reason about is that it's very easy to 
confuse parallel execution with concurrent execution. I think that most of this confusion stems from how we tend to 
model events in our everyday life. We tend to not define these terms very precise so our intuition 
is often wrong. 
> It doesn't help that **concurrent** is defined in the dictionary as: _operating or occurring at the same time_ which 
> doesn't really help us much when trying to describe how it differs from **parallel**

**Let's model our world with these simple rules:**
1. Everything you do requires resources
2. Resources are limited and/or timing is important
4. Our main purpose is to use resources as efficiently as possible
5. Waiting is performing work, however it's useless work
6. Resources are limited

Now, all of these need to be true for concurrency to even matter, but this is 
true from on perspective or another more often than you'd think. Even in the 
simple case that your server or computer has more than enough resources, you still
waste energy if you're inefficient or your users might wait longer then they should.

### Parallelism

Is increasing the resources we use to solve a task. It has nothing to do with efficiency.

### Concurrency

Has everything to do with efficiency and resource utilization. Concurrency can never make _one single task go faster_. 
It can only help us utilize our resources better and thereby _finish a set of tasks faster_.


### Let's draw some parallels to process economics

In businesses that manufacture goods, we often talk about LEAN processes. And this is pretty easy to compare with what concurrency
does for programmers. I'll let let this 3 minute video explain it for me:

<iframe width="560" height="315" src="https://www.youtube.com/embed/Oz8BR5Lflzg" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

This very modern video might not make you a LEAN expert, but it does explain some
of the gains we try to achieve when applying LEAN techniques, amongst those: 
eliminate waiting and non-value-adding tasks.

> In programming we could say that we want to avoid `blocking` and `polling` in a busy loop.

Now would adding more resources (more workers) help in this case? Yes, but we use double the resources to produce the 
same output as 1 person with a optimal process could do. That's not optimal utilization of our resources.

> To continue our metaphor, we could say that we could solve the problem of a freezing UI while waiting for an I/O event to occur 
> by using a new thread and `poll`in a loop or `block` there instead. However, that thread is either consuming resources doing
> nothing or worse, using one core to busy loop while checking if an event is ready. Either way it's not optimal, especially
> if you run a server you want to utilize fully.

If you consider the coffee machine as some I/O resource, we would like to start that process, then move on to preparing the 
next job, or do other work that needs to be done instead of waiting.



But that means there are things happening in parallel here? Yes, the coffee machine is doing work while the "worker" is doing
maintenance and filling water. But this is the crux: Our reference frame is the worker, not the whole system. The worker
is doing things concurrently. The guy making coffee (the worker) is your code. 

**Concurrency is about working smarter and harder. Parallelism is throwing more resources at the problem**


## Concurrency and its relation to I/O

As you might understand from what I've written so far, writing async code mostly
makes sense when you need to be smart to make optimal use of your resources.

Now if you're code is working hard to solve a problem, there often is no help
in concurrency, this is where parallelism comes in to play since it gives you
a way to throw more resources at the problem if you can split it into parts that
you can work on in parallel.

**I can see two major use cases for concurrency:**

1. When performing I/O and you need to wait for some external event to occur
2. When you need to divide your attention and prevent one task from waiting too 
long

The first is the classic I/O example, where you will have to wait for a network
call, a file operation or something else to happen before you can progress one 
task, but you have many tasks to do so instead of waiting you continue work 
elsewhere and either check in regularly to see if the task is ready to progress
or make sure you are notified when that task is ready to progress.

The second is an example that is often the case when having an UI. Let's pretend
you only have one core. How do you prevent the whole UI from becoming unresponsive
while performing other CPU intensive tasks?

Well, you can stop what ever task you're doing every 16ms, and run the "update UI"
task, and then resume whatever you were doing afterwards. This way, you will have
to stop/resume your task 60 times a second, but you will also have a fully 
responsive UI which has roughly a 60 Hz refresh rate.

## What about threads

We'll cover threads a bit more when we talk about operating systems, but I'll mention them here as well. The problem with
threads provided by the operating system is that they appear to be mapped to cores. But that is not neccicarely the truth even 
though most operating systems will try to map one thread to a core up to the number of threads is equal to  the number of cores.

Once we create more threads than there are cores, the OS will switch between our threads and progress each of them concurrently
using the scheduler to give each thread some time to run. So in this way, threads can be a means to achieve parallelism, but 
they can also be a means to achieve concurrency. Now if you find that confusing, I understand. There is a reason why this is 
hard to understand. 

But it's a good time to talk about changing the reference frame.

## Changing the reference frame

When you write code that is perfectly synchronous from your perspective, let's take a look at how that looks from the operating
system perspective.

The Operating System might not run your code from start to end at all. It might stop and resume your process many times. 
The CPU might get interrupted and handle som inputs while you think it's only focused on your task. So synchronous execution is
only an illusion. But from the perspective of you as a programmer it's not, and that is the important takeaway:

When we talk about concurrency without providing any other context we are using you as a programmer and your code 
(your process) as the reference frame.

The reason I spend so much time on this is that once you realize that, you'll start to see that some of the things you hear and
learn that might seem contradicting really is not. You'll just have to consider the reference frame first.

If this sounds complicated, I promise that we'll get to see and know this better as we go on. So if you found this confusing, relax
this will become clearer as we go on.

