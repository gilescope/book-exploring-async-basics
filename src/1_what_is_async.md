# What is asynchronous code execution and concurrency?

Composition of independently executed processes.

Concurrency is about **dealing** with a lot of things at the same time. 

Parralellism is about **doing** a lot of things at the same time.

We handle concurrency by executing tasks/processes in an asynchronous manner. Asynchronous code execution is therefore the 
way we handle concurrency in programming.

## The mental model I use.

I firmly believe the main reason we find parallel and concurrent programming hard to reason about is that it's very easy to 
confuse parallel execution with concurrent execution. I think that most of this confusion stems from how we tend to 
model events in our everyday life. We tend to not define these terms very precise so our intuition 
is often wrong.

**Let's model our world with these simple rules:**
1. Everything you do requires resources
2. Resources are limited
3. Timing is important
4. Our main purpose is to use resources as efficiently as possible
5. Waiting is wasting resources
6. We don't have enough resources

Now, all of these need to be true for concurrency to even matter, but this is true from on perspective or another more often than you'd think.

### Parallelism

Is increasing the resources we use to solve a task. It has nothing to do with efficiency.

### Concurrency

Has everything to do with efficiency and resource utilization. Concurrency can never make _one single task go faster_. 
It can only help us utilize our resources better and thereby _finish a set of tasks faster_.


### Let's draw some parallels to process economics

In businesses that manufacture goods, we often talk about LEAN processes. And this is pretty easy to compare with what concurrency
does for programmers. I'll let let this 3 minute video explain it for me:

<iframe width="560" height="315" src="https://www.youtube.com/embed/Oz8BR5Lflzg" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

Now would adding more resources (more workers) help in this case? Yes, but we use double the resources to produce the 
same output as 1 person with a optimal process could do. That's not optimal utilization of our resources.

If you consider the coffee machine as some I/O resource we would like to start that process, and move on to preparing the 
next job, or do other work that needs to be done instead of waiting.

But that means there are things happening in parallel here? Yes, the coffee machine is doing work while the "worker" is doing
maintenance and filling water. But this is the crux: Our reference frame is the worker, not the whole system. The worker
is doing things concurrently. The guy making coffee (the worker) is your code. 

**Concurrency is about working smarter and harder. Parallelism is throwing more resources at the problem**

### What about threads

We'll cover threads a bit more when we talk about operating systems, but I'll mention them here as well. The problem with
threads provided by the operating system is that they appear to be mapped to cores. But that is not neccicarely the truth even 
though most operating systems will try to map one thread to a core up to the number of threads is equal to  the number of cores.

Once we create more threads than there are cores, the OS will switch between our threads and progress each of them concurrently
using the scheduler to give each thread some time to run. So in this way, threads can be a means to achieve parallelism, but 
they can also be a means to achieve concurrency. Now if you find that confusing, I understand. There is a reason why this is 
hard to understand. 

But it's a good time to talk about changing the reference frame.

### Changing the reference frame

When you write code that is perfectly synchronous from your perspective, let's take a look at how that looks from the operating
system perspective.

The Operating System might not run your code from start to end at all. It might stop and resume your process many times. 
The CPU might get interrupted and handle som inputs while you think it's only focused on your task. So synchronous execution is
only an illusion. But from the perspective of you as a programmer it's not, and that is the important takeaway:

When we talk about concurrency without providing any other context we are using you as a programmer and your code 
(your process) as the reference frame.

The reason I spend so much time on this is that once you realize that, you'll start to see that some of the things you hear and
learn that might seem contradictionary is not. You'll have to consider the reference frame first.

If this sounds complicated, I promise that we'll get to see and know this better as we go on. So if you found this confusing, relax
this will become clearer as we go on.

## Lets finish up this chapter with some definitions

We can derive some definitions from all above paragraphs that will help us going forward:

### Resource
Something needed to perform work on a task. We have limited amounts of resources.

### Performing a task
For this to have any meaning we'll have to define this as work that requires some kind of resource, whether it's computational power 
from the CPU or it's your brain processing something. 

### Parallel
Something happening at the **exact** same time that requires some limited resource to happen.

### Concurrent
The ability to stop and resume a task. This means that we have more options when deciding how our limited resources are spent over time. 
This enable us to respond to external events, change priorities or share resources more efficiently (or fairly).

### Reference frame
You need to know what reference frame you use when talking about concurrency.

