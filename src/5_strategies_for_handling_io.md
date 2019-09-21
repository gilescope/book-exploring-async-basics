# Strategies for handling I/O

Before we dive into Writing some code we'll finish off this part of the book talking a bit about different strategies of handling I/O and concurrency.


## The perfect solution

Let's start off by picturing a perfect world, and how the most efficient way of handelig I/O in a concurrent manner could look like.

If we go back to this model and think it through based on the knowledge we now have. If we want to use every CPU cycle the best way possible how would we design this?

![overview](../book/images/AsyncBasicsSimplified.png)

The best way would be the following:

1. We give the Network Card a message that we want to be notified **immediately** when new data has arrived for us.
2. The network card is hyper optimized in the way it checks for data, it's firmware makes sure of that.
3. As soon as some data has arrived for us the Network Card let's us know
4. We either finish what we're doing or handle that data immediately

Now we'll look at some ways we normally handle this:

## Using OS threads

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


## Green threads

Another common way of handling this is green threads. Languages like GO uses this to great success. In many ways this is similar to what the OS does but the runtime can be better adjusted and suited to your specific needs.

Pros:
- Simple to use, your code will look like it does when using OS threas
- Reasonably performant
- Abundant memory usage is less of a problem

Cons:
- You need a runtime, and by having that you are duplicating part of the work the OS already does. The rundime will have a cost.
- 
- Can be difficult to implement in a flexible way
- 