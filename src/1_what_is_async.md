# What is async and concurrency?

Composition of independently executed processes.

Concurrency is about **dealing** with a lot of things at the same time.

Parralellism is about **doing** a lot of things at the same time.

We handle concurrency by executing tasks/processes in an asynchronous manner.

## The mental model I use

It's tempting to confuse parallel execution with concurrent execution, and I think that most of this confusion stems from how
we tend to view this stuff in everyday life, we tend to not define these terms very precises so our intuition about this stuff is
kind of random.

First of all, we have to understand that these definitions depend on some kind of reference frame. And when we talk about 
concurrent, asynchronous and parallel in programming we do that from the reference frame from you as the programmer and the 
program you're writing.

As we'll see if we change this reference frame things might look different, but first, let's make a mental model we can build upon.

### Some definitions

#### Resource:
Something needed to perform work, or a task, of which we have limited amounts of.

#### Performing a task:
For this to have any meaning we'll have to define this as work that requires some kind of resource, whether it's computational power 
from the CPU or it's your brain processing something. 

For example, from in the context of your brain, talking on the phone is not a task when:
1. You hold the phone to your ear and there is silence or in between ring tones when waiting for someone to pick up
2. When there are silences or breaks in between sentences or for some other reason
3. If you're not actively listening or talking

Now a very efficient person can do some other work while waiting on the phone for someone to pick up, as long as that task is
quick to stop and resume later on short notice. There might even be reason to do some tasks while on the phone if you can context 
switch good enough (I've witnessed people watching TV and talk on the phone at the same time). But even though we tend to say we
do this i parallel we don't, we just pause and resume tasks very quickly.

So already here we see that our daily definitions are way to imprecise for us to make good mental models about this subject.

#### Parallel:
Something happening at the **exact** same time that requires some limited resource to happen.

#### Concurrent:
The ability to stop and resume a task. This means that we have more options when deciding how our limited resources are spent over time. 
This enable us to respond to external events, change priorities or share resources more efficiently (or fairly).


### We humans

Let's take a look at how we humans work. Now the brain is the natural thing to compare with a CPU we have in a processor, and it's not
a bad mental model to use.

Now, when we say we can do something "at the same time" we're really both talking about what we actually can do in parralel and
what we do concurrently.

#### We breathe and write at the same time
Now some of the things we do are truly parallel tasks. Our brain has a part that manages our vital functions like breathing, having our
heart pump and so on. TWe could compare it with the FPU (Floating Point Unit) of a CPu or maybe even a separate "core" tasked with making
sure that we do these tasks whether we sleep or are awake.

This is a truly parallel task. But beyond that, if we want to have more things done at the **exact** same time we probably 
need two persons or more to do it. Note the emphasis on **exact same time**, in our daily lives we are usually very 
imprecise about this part and that causes a lof of confusion when you want to reason about the formal definition of doing
something in parallel.

#### Wa can't actually perform two cognitive tasks in parallel

So I've been amazed sometimes, how some people seem to be able to do things in parallel. Part of the reason is that I myself is 
extremely bad at multitasking. It's even a source of laughs for my colleagues and friends.

The main problem here is that [we have trouble processing two inputs at the same time](https://bmcpsychology.biomedcentral.com/articles/10.1186/2050-7283-1-18), 
what we really do is to switch rapidly between two tasks, and some people do that better than others, but most often it has a cost 
in that there is some time wasted just switching between the tasks.


### Changing the reference frame

Now, if we instead of looking at tasks from the context of the brain, and instead look at them in the context of our body. Things change.

Physically, when you hold the phone to your ear, you're on the phone, no matter whats happening or not in the phone. The point
is that it limits what you can do, one of your hands is tied up in that task and so on.

Physically, there is no problem looking at the TV and talking on the phone at the same time, you can probably even jump on 
one leg while you look at the TV and talk on the phone. You'll look a bit silly though.

So changing the reference frame changes what we conceive as parallel or concurrent and the same thing happens on your computer.

The Operating System might not run your code from start to stop and might stop and resume it many times. The CPU might get interrupted and
handle som inputs while you think it's only focused on your task. Concurrency when we talk about it is from the perspective of you as a 
programmer and your code.

The reason I spend so much time on this is that once you realize that, you'll start to see that some of the things you hear and
learn that might seem contradictionary is not. You'll have to consider the reference frame first.

If this sounds complicated, I promise that we'll get to see and know this better as we go on. So if you found this confusing, relax
this will become clearer as we go on.