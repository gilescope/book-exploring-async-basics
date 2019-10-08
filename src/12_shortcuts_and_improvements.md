# Shortcuts and improvements

There are several things we could do here to make this a better implementation which I want to point out here. 

One is to set a max backlog of callbacks to execute in a single tick, so we don't starve the threadpool or file handlers. In other words, limit the amount of callbacks we execute in a single iteration.

Another is to dynamically decide if/and how long the thread could be allowed to be parked for example by looking at the backlog of events, and if there is any backlog disable any waiting at all. 

When it comes to data structures, there is lots we could have done.
If you look closely, you'll see that some of our Vec's will only grow, and not resize ever, so if we have a period of very high load,the memory will stay higher than it needs to be until a restart. This could be dealt with or a different data structure whitout this property could have been chosen (like a linked list).

Some of the shortcuts (like panicing when we have no available threads in the threadpool) is especially limiting. If you were to improve this example, that would be one of the first things to fix.

There is probably several other aspects that deserve to be mentioned here, but I'll leve it with this for now.

