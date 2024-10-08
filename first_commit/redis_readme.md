**README: Contents**\
  [Introduction](#Introduction)\
  [Beyond key-value databases](#Beyond-key-value-databases)\
  [What are the differences between Redis and Memcached?](#What-are-the-differences-between-Redis-and-Memcached)\
  [What are the differences between Redis and Tokyo Cabinet / Tyrant?](#What-are-the-differences-between-Redis-and-Tokyo-Cabinet-/-Tyrant)\
  [Does Redis support locking?](#Does-Redis-support-locking)\
  [Multiple databases support](#Multiple-databases-support)\
  [Redis Data Types](#Redis-Data-Types)\
    [Implementation Details](#Implementation-Details)\
  [Redis Tutorial](#Redis-Tutorial)\
  [License](#License)\
  [Credits](#Credits)

README
======

Introduction
============

Redis is a database. To be more specific redis is a very simple database implementing a dictionary where keys are associated with values. For example I can set the key "surname\_1992" to the string "Smith".  
  
Redis takes the whole dataset in memory, but the dataset is persistent since from time to time Redis writes a dump of the dataset on disk asynchronously. The dump is loaded every time the server is restarted. This means that if a system crash occurs the last few queries can get lost (that is acceptable in many applications), so we supported master-slave replication from the early days.

Beyond key-value databases
==========================

In most key-value databases keys and values are simple strings. In Redis keys are just strings too, but the associated values can be Strings, Lists and Sets, and there are commands to perform complex atomic operations against this data types, so you can think at Redis as a data structures server.  
  
For example you can append elements to a list stored at the key "mylist" using the LPUSH or RPUSH operation in O(1). Later you'll be able to get a range of elements with LRANGE or trim the list with LTRIM. Sets are very flexible too, it is possible to add and remove elements from Sets (unsorted collections of strings), and then ask for server-side intersection of Sets.  
  
All this features, the support for sorting Lists and Sets, allow to use Redis as the sole DB for your scalable application without the need of any relational database. [We wrote a simple Twitter clone in PHP + Redis](TwitterAlikeExample.html) to show a real world example, the link points to an article explaining the design and internals in very simple words.

What are the differences between Redis and Memcached?
=====================================================

In the following ways:  
  

*   Memcached is not persistent, it just holds everything in memory without saving since its main goal is to be used as a cache. Redis instead can be used as the main DB for the application. We [wrote a simple Twitter clone](TwitterAlikeExample.html) using only Redis as database.

*   Like memcached Redis uses a key-value model, but while keys can just be strings, values in Redis can be lists and sets, and complex operations like intersections, set/get n-th element of lists, pop/push of elements, can be performed against sets and lists. It is possible to use lists as message queues.

What are the differences between Redis and Tokyo Cabinet / Tyrant?
==================================================================

Redis and Tokyo can be used for the same applications, but actually they are **ery** different beasts:  
  

*   Tokyo is purely key-value, everything beyond key-value storing of strings is delegated to an embedded Lua interpreter. AFAIK there is no way to guarantee atomicity of operations like pushing into a list, and every time you want to have data structures inside a Tokyo key you have to perform some kind of object serialization/de-serialization.

*   Tokyo stores data on disk, synchronously, this means you can have datasets bigger than memory, but that under load, like every kind of process that relay on the disk I/O for speed, the performances may start to degrade. With Redis you don't have this problems but you have another problem: the dataset in every single server must fit in your memory.

*   Redis is generally an higher level beast in the operations supported. Things like SORTing, Server-side set-intersections, can't be done with Tokyo. But Redis is not an on-disk DB engine like Tokyo: the latter can be used as a fast DB engine in your C project without the networking overhead just linking to the library. Still remember that in many scalable applications you need multiple servers talking with multiple servers, so the server-client model is almost always needed.

Does Redis support locking?
===========================

No, the idea is to provide atomic primitives in order to make the programmer able to use redis with locking free algorithms. For example imagine you have 10 computers and 1 redis server. You want to count words in a very large text. This large text is split among the 10 computers, every computer will process its part and use Redis's INCR command to atomically increment a counter for every occurrence of the word found.  
  
INCR/DECR are not the only atomic primitives, there are others like PUSH/POP on lists, POP RANDOM KEY operations, UPDATE and so on. For example you can use Redis like a Tuple Space ([http://en.wikipedia.org/wiki/Tuple\_space](http://en.wikipedia.org/wiki/Tuple_space)) in order to implement distributed algorithms.  
  
(News: locking with key-granularity is now planned)

Multiple databases support
==========================

Another synchronization primitive is the support for multiple DBs. By default DB 0 is selected for every new connection, but using the SELECT command it is possible to select a different database. The MOVE operation can move an item from one DB to another atomically. This can be used as a base for locking free algorithms together with the 'RANDOMKEY' or 'POPRANDOMKEY' commands.

Redis Data Types
================

Redis supports the following three data types as values:  
  

*   Strings: just any sequence of bytes. Redis strings are binary safe so they can not just hold text, but images, compressed data and everything else.
*   Lists: lists of strings, with support for operations like append a new string on head, on tail, list length, obtain a range of elements, truncate the list to a given length, sort the list, and so on.
*   Sets: an unsorted set of strings. It is possible to add or delete elements from a set, to perform set intersection, union, subtraction, and so on.

Values can be Strings, Lists or Sets. Keys can be a subset of strings not containing newlines ("\\n") and spaces (" ").  
  
Note that sometimes strings may hold numeric vaules that must be parsed by Redis. An example is the INCR command that atomically increments the number stored at the specified key. In this case Redis is able to handle integers that can be stored inside a 'long long' type, that is a 64-bit signed integer.

Implementation Details
----------------------

Strings are implemented as dynamically allocated strings of characters. Lists are implemented as doubly linked lists with cached length. Sets are implemented using hash tables that use chaining to resolve collisions.

Redis Tutorial
==============

(note, you can skip this section if you are only interested in "formal" doc.)  
  
Later in this document you can find detailed information about Redis commands, the protocol specification, and so on. This kind of documentation is useful but... if you are new to Redis it is also BORING! The Redis protocol is designed so that is both pretty efficient to be parsed by computers, but simple enough to be used by humans just poking around with the 'telnet' command, so this section will show to the reader how to play a bit with Redis to get an initial feeling about it, and how it works.  
  
To start just compile redis with 'make' and start it with './redis-server'. The server will start and log stuff on the standard output, if you want it to log more edit redis.conf, set the loglevel to debug, and restart it.  
  
You can specify a configuration file as unique parameter:  
  

> ./redis-server /etc/redis.conf

This is NOT required. The server will start even without a configuration file using a default built-in configuration.  
  
Now let's try to set a key to a given value:  
  

$ telnet localhost 6379
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^\]'.
SET foo 3  
bar
+OK

The first line we sent to the server is "set foo 3". This means "set the key foo with the following three bytes I'll send you". The following line is the "bar" string, that is, the three bytes. So the effect is to set the key "foo" to the value "bar". Very simple!  
  
(note that you can send commands in lowercase and it will work anyway, commands are not case sensitive)  
  
Note that after the first and the second line we sent to the server there is a newline at the end. The server expects commands terminated by "\\r\\n" and sequence of bytes terminated by "\\r\\n". This is a minimal overhead from the point of view of both the server and client but allows us to play with Redis with the telnet command easily.  
  
The last line of the chat between server and client is "+OK". This means our key was added without problems. Actually SET can never fail but the "+OK" sent lets us know that the server received everything and the command was actually executed.  
  
Let's try to get the key content now:  
  

GET foo
3
bar

Ok that's very similar to 'set', just the other way around. We sent "get foo", the server replied with a first line that is just a number of bytes the value stored at key contained, followed by the actual bytes. Again "\\r\\n" are appended both to the bytes count and the actual data.  
  
What about requesting a non existing key?  
  

GET blabla
nil

When the key does not exist instead of the length just the "nil" string is sent. Another way to check if a given key exists or not is indeed the EXISTS command:  
  

EXISTS nokey
0
EXISTS foo
1

As you can see the server replied '0' the first time since 'nokey' does not exist, and '1' for 'foo', a key that actually exists.  
  
Ok... now you know the basics, read the [REDIS COMMAND REFERENCE](CommandReference.html) section to learn all the commands supported by Redis and the [PROTOCOL SPECIFICATION](ProtocolSpecification.html) section for more details about the protocol used if you plan to implement one for a language missing a decent client implementation.

License
=======

Redis is released under the BSD license. See the COPYING file for more information.

Credits
=======

Redis is written and maintained by Salvatore Sanfilippo, Aka 'antirez'.  
  
Enjoy, antirez
