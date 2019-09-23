# Shortcuts and improvements



## Shortcuts
This is the event loop. There are several things we could do here to make it a better implementation. One is to set a max backlog of callbacks to execute in a single tick, so we don't starve the threadpool or file handlers. Another is to dynamically decide if/and how long the thread could be allowed to be parked for example by looking at the backlog of events, and if there is any backlog disable it. Some of our Vec's will only grow, and not resize, so if we have a period of very high load,the memory will stay higher than we need until a restart. This could be dealt with or a differentdata structure could be used.