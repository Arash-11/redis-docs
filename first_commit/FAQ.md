**FAQ: Contents**\
  [Why I need Redis if there is already memcachedb, Tokyo Cabinet, ...?](#Why-I-need-Redis-if-there-is-already-memcachedb-Tokyo-Cabinet)\
  [Isn't this key-value thing just hype?](#Isnt-this-key-value-thing-just-hype)\
  [Can I backup a Redis DB while the server is working?](#Can-I-backup-a-Redis-DB-while-the-server-is-working)\
  [What's the Redis memory footprint?](#Whats-the-Redis-memory-footprint)\
  [I like Redis high level operations and features, but I don't like it takes everything in memory and I can't have a dataset larger the memory. Plans to change this?](#I-like-Redis-high-level-operations-and-features-but-I-dont-like-it-takes-everything-in-memory-and-I-cant-have-a-dataset-larger-the-memory-Plans-to-change-this)\
  [Ok but I absolutely need to have a DB larger than memory, still I need the Redis features](#Ok-but-I-absolutely-need-to-have-a-DB-larger-than-memory-still-I-need-the-Redis-features)\
  [What happens if Redis runs out of memory?](#What-happens-if-Redis-runs-out-of-memory)\
  [What Redis means actually?](#What-Redis-means-actually)\
  [Why did you started the Redis project?](#Why-did-you-started-the-Redis-project)

FAQ
===

Why I need Redis if there is already memcachedb, Tokyo Cabinet, ...?
====================================================================

Memcachedb is basically memcached done persistent. Redis is a different evolution path in the key-value DBs, the idea is that the main advantages of key-value DBs are retained even without a so severe loss of comfort of plain key-value DBs. So Redis offers more features:  
  

*   Keys can store different data types, not just strings. Notably Lists and Sets. For example if you want to use Redis as a log storage system for different computers every computer can just `RPUSH data to the computer_ID key`. Don't want to save more than 1000 log lines per computer? Just issue a `LTRIM computer_ID 0 999` command to trim the list after every push.

*   Another example is about Sets. Imagine to build a social news site like [Reddit](http://reddit.com). Every time a user upvote a given news you can just add to the news\_ID\_upmods key holding a value of type SET the id of the user that did the upmodding. Sets can also be used to index things. Every key can be a tag holding a SET with the IDs of all the objects associated to this tag. Using Redis set intersection you obtain the list of IDs having all this tags at the same time.

*   We wrote a [simple Twitter Clone](http://retwis.antirez.com) using just Redis as database. Download the source code from the download section and imagine to write it with a plain key-value DB without support for lists and sets... it's **much** harder.

*   Multiple DBs. Using the SELECT command the client can select different datasets. This is useful because Redis provides a MOVE atomic primitive that moves a key form a DB to another one, if the target DB already contains such a key it returns an error: this basically means a way to perform locking in distributed processing.

*   **So what is Redis really about?** The User interface with the programmer. Redis aims to export to the programmer the right tools to model a wide range of problems. **Sets, Lists with O(1) push operation, lrange and ltrim, server-side fast intersection between sets, are primitives that allow to model complex problems with a key value database**.

Isn't this key-value thing just hype?
=====================================

I imagine key-value DBs, in the short term future, to be used like you use memory in a program, with lists, hashes, and so on. With Redis it's like this, but this special kind of memory containing your data structures is shared, atomic, persistent.  
  
When we write code it is obvious, when we take data in memory, to use the most sensible data structure for the work, right? Incredibly when data is put inside a relational DB this is no longer true, and we create an absurd data model even if our need is to put data and get this data back in the same order we put it inside (an ORDER BY is required when the data should be already sorted. Strange, dont' you think?).  
  
Key-value DBs bring this back at home, to create sensible data models and use the right data structures for the problem we are trying to solve.

Can I backup a Redis DB while the server is working?
====================================================

Yes you can. When Redis saves the DB it actually creates a temp file, then rename(2) that temp file name to the destination file name. So even while the server is working it is safe to save the database file just with the _cp_ unix command. Note that you can use master-slave replication in order to have redundancy of data, but if all you need is backups, cp or scp will do the work pretty well.

What's the Redis memory footprint?
==================================

Worst case scenario: 1 Million keys with the key being the natural numbers from 0 to 999999 and the string "Hello World" as value use 100MB on my Intel macbook (32bit). Note that the same data stored linearly in an unique string takes something like 16MB, this is the norm because with small keys and values there is a lot of overhead. Memcached will perform similarly.  
  
With large keys/values the ratio is much better of course.  
  
64 bit systems will use much more memory than 32 bit systems to store the same keys, especially if the keys and values are small, this is because pointers takes 8 bytes in 64 bit systems. But of course the advantage is that you can have a lot of memory in 64 bit systems, so to run large Redis servers a 64 bit system is more or less required.

I like Redis high level operations and features, but I don't like it takes everything in memory and I can't have a dataset larger the memory. Plans to change this?
===================================================================================================================================================================

The whole key-value hype started for a reason: performances. Redis takes the whole dataset in memory and writes asynchronously on disk in order to be very fast, you have the best of both worlds: hyper-speed and persistence of data, but the price to pay is exactly this, that the dataset must fit on your computers RAM.  
  
If the data is larger then memory, and this data is stored on disk, what happens is that the bottleneck of the disk I/O speed will start to ruin the performances. Maybe not in benchmarks, but once you have real load with distributed key accesses the data must come from disk, and the disk is damn slow. Not only, but Redis supports higher level data structures than the plain values. To implement this things on disk is even slower.  
  
Redis will always continue to hold the whole dataset in memory because this days scalability requires to use RAM as storage media, and RAM is getting cheaper and cheaper. Today it is common for an entry level server to have 16 GB of RAM! And in the 64-bit era there are no longer limits to the amount of RAM you can have in theory.

Ok but I absolutely need to have a DB larger than memory, still I need the Redis features
=========================================================================================

One possible solution is to use both MySQL and Redis at the same time, basically take the state on Redis, and all the things that get accessed very frequently: user auth tokens, Redis Lists with chronologically ordered IDs of the last N-comments, N-posts, and so on. Then use MySQL as a simple storage engine for larger data, that is just create a table with an auto-incrementing ID as primary key and a large BLOB field as data field. Access MySQL data only by primary key (the ID). The application will run the high traffic queries against Redis but when there is to take the big data will ask MySQL for specific resources IDs.

What happens if Redis runs out of memory?
=========================================

With modern operating systems malloc() returning NULL is not common, usually the server will start swapping and Redis performances will be disastrous so you'll know it's time to use more Redis servers or get more RAM.  
  
However it is planned to add a configuration directive to tell Redis to stop accepting queries but instead to SAVE the latest data and quit if it is using more than a given amount of memory. Also the new INFO command (work in progress in this days) will report the amount of memory Redis is using so you can write scripts that monitor your Redis servers checking for critical conditions.  
  
Update: redis SVN is able to know how much memory it is using and report it via the [INFO](InfoCommand.html) command.

What Redis means actually?
==========================

Redis means two things:

*   it's a joke on the word Redistribute (instead to use just a Relational DB redistribute your workload among Redis servers)
*   it means REmote DIctionary Server

Why did you started the Redis project?
======================================

In order to scale [LLOOGG](http://lloogg.com). But after I got the basic server working I liked the idea to share the work with other guys, and Redis was turned into an open source project.
