# Strategies for handling I/O

Before we dive into Writing some code we'll finish off this part of the book talking a bit about different strategies of handling I/O and concurrency.


## The perfect solution

Let's start off by picturing a perfect world, and how the most efficient way of handelig I/O in a concurrent manner could look like.

If we go back to this model and think it through based on the knowledge we now have. If we want to use every CPU cycle the best way possible how would we design this?

![overview](./images/AsyncBasicsSimplified.png)

The best way would be the following:

1. We give the Network Card a message that we want to be notified **immediately** when new data has arrived for us.
2. The network card is hyper optimized in the way it checks for data, it's firmware makes sure of that.
3. As soon as some data has arrived for us the Network Card let's us know
4. We either finish what we're doing or handle that data immediately

Now we'll look at some ways we normally handle this:

## 1. Using OS threads

Now one way of accomplishing this is letting the OS take care of everything for us. We do this by simply spawning a new OS thread for each task we want to accomplish and write code like we normally would.

Pros:
- Simple
- Easy to code
- Reasonably performant
- You get paralellism for free

Cons:
- OS level threads come with a rather large stack. If you have many tasks happening simultaniously (like in a webserver under heavy load) you'll run out of memory pretty soon.
- There are a lot of syscalls involved this can be pretty costly
- The OS has many things it needs to handle. It might not switch back to your thread as fast as you'd wish
- The OS doesn't know which tasks to prioritize, and you might want to give som tasks a higher priority than others.


## 2. Green threads

Another common way of handling this is green threads. Languages like GO uses this to great success. In many ways this is similar to what the OS does but the runtime can be better adjusted and suited to your specific needs.

Pros:
- Simple to use, your code will look like it does when using OS threas
- Reasonably performant
- Abundant memory usage is less of a problem

Cons:
- You need a runtime, and by having that you are duplicating part of the work the OS already does. The rundime will have a cost.
- Can be difficult to implement in a flexible way to handle a wide set of tasks

## 3. Poll based event loops supported by the OS

The third way we're covering today is the one that most closely matches our _ideal_ solution. In this solution the we register an interest in an event, and then let the OS tell us when it's ready. 

The way this works is that we tell the OS that we're interested in knowing when data is arriving for us on the network card. The network card issues an interrupt when something has happened in which the driver let's the OS know that the data is ready. The OS let's us know that data is ready for us to read.

**Pros:**

- Very little work is duplicated which makes it very performant
- It's very efficient
- Gives us the maximum amount of flexibility to decide how to handle the events that occurs

**Cons:**

- Different operating systems have different ways of handle these kind of queues. Some of them are difficult to reconcile with each other. Some operating systems has limitations on what I/O operations support this method.
- Great flexibility comes with a good deal of complexity
- Difficult to write an abstraction layer that accounts for the differences between the operating systems without introducing unwanted costs, and at the same time provide a ergonomic API.


## Final note

The Node runtime uses a combination of both 1 and 3, but tries to force all I/O to use alternative 3. This is also part of the reason why Node is so good at handle many connections concurrently.

Rusts async story is modeled around option 3, and some of the reason it has taken a long time is related to the _cons_ of this method. Most notably the last point.
