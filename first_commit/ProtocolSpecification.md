**ProtocolSpecification: Contents**\
  [Protocol Specification](#Protocol-Specification)\
    [Networking layer](#Networking-layer)\
    [Simple INLINE commands](#Simple-INLINE-commands)\
    [Bulk commands](#Bulk-commands)\
    [Bulk replies](#Bulk-replies)\
    [Bulk reply error reporting](#Bulk-reply-error-reporting)\
    [Multi-Bulk replies](#Multi-Bulk-replies)\
    [Nil elements in Multi-Bulk replies](#Nil-elements-in-Multi-Bulk-replies)\
    [Multi-Bulk replies errors](#Multi-Bulk-replies-errors)\
    [Status code reply](#Status-code-reply)\
    [Integer reply](#Integer-reply)\
    [Single line reply](#Single-line-reply)\
    [Multiple commands and pipelining](#Multiple-commands-and-pipelining)

ProtocolSpecification
=====================

Protocol Specification
======================

The Redis protocol is a compromise between being easy to parse by a computer and being easy to parse by an human. Before reading this section you are strongly encouraged to read the "REDIS TUTORIAL" section of this README in order to get a first feeling of the protocol playing with it by TELNET.

Networking layer
----------------

A client connects to a Redis server creating a TCP connection to the port 6973. Every redis command or data transmitted by the client and the server is terminated by "\\r\\n" (CRLF).

Simple INLINE commands
----------------------

The simplest commands are the inline commands. This is an example of a server/client chat (the server chat starts with S:, the client chat with C:)  
  

C: PING
S: +PONG

An inline command is a CRLF-terminated string sent to the client. The server usually replies to inline commands with a single line that can be a number or a return code.  
  
When the server replies with a status code (that is a one line reply just indicating if the operation succeeded or not), if the first character of the reply is a "+" then the command succeeded, if it is a "-" then the following part of the string is an error.  
  
The following is another example of an INLINE command returning an integer:  
  

C: EXISTS somekey
S: 0

Since 'somekey' does not exist the server returned '0'.  
  
Note that the EXISTS command takes one argument. Arguments are separated simply by spaces.

Bulk commands
-------------

A bulk command is exactly like an inline command, but the last argument of the command must be a stream of bytes in order to send data to the server. the "SET" command is a bulk command, see the following example:  
  

C: SET mykey 6
C: foobar
S: +OK

The last argument of the commnad is '6'. This specify the number of DATA bytes that will follow (note that even this bytes are terminated by two additional bytes of CRLF).  
  
All the bulk commands are in this exact form: instead of the last argument the number of bytes that will follow is specified, followed by the bytes, and CRLF. In order to be more clear for the programmer this is the string sent by the client in the above sample:  
  

> "SET mykey 6\\r\\nfoobar\\r\\n"

Bulk replies
------------

The server may reply to an inline or bulk command with a bulk reply. See the following example:  
  

C: GET mykey
S: 6
S: foobar

A bulk reply is very similar to the last argument of a bulk command. The server sends as the first line the number of bytes of the actual reply followed by CRLF, then the bytes are sent followed by additional two bytes for the final CRLF. The exact sequence sent by the server is:  
  

> "6\\r\\nfoobar\\r\\n"

If the requested value does not exist the bulk reply will use the special value 'nil' instead to send the line containing the number of bytes to read. This is an example:  
  

C: GET nonexistingkey
S: nil

The client library API should not return an empty string, but a nil object. For example a Ruby library should return 'nil' while a C library should return NULL.

Bulk reply error reporting
--------------------------

Bulk replies can signal errors, for example trying to use GET against a list value is not permitted. Bulk replies use a negative bytes count in order to signal an error. An error string of ABS(bytes\_count) bytes will follow. See the following example:  
  

S: GET alistkey
S: -38
S: -ERR Requested element is not a string

\-38 means: sorry your operation resulted in an error, but a 38 bytes string that explains this error will follow. Client APIs should abort on this kind of errors, for example a PHP client should call the die() function.  
  
The following commands reply with a bulk reply: GET, KEYS, LINDEX, LPOP, RPOP

Multi-Bulk replies
------------------

Commands similar to LRANGE needs to return multiple values (every element of the list is a value, and LRANGE needs to return more than a single element). This is accomplished using multiple bulk writes, prefixed by an initial line indicating how many bulk writes will follow. Example:  
  

C: LRANGE mylist 0 3
S: 4
S: 3
S: foo
S: 3
S: bar
S: 5
S: Hello
S: 5
S: World

The first line the server sent is "4\\r\\n" in order to specify that four bulk write will follow. Then every bulk write is transmitted.  
  
If the specified key does not exist instead of the number of elements in the list, the special value 'nil' is sent. Example:  
  

C: LRANGE nokey 0 1
S: nil

A client library API SHOULD return a nil object and not an empty list when this happens. This makes possible to distinguish between empty list and non existing ones.

Nil elements in Multi-Bulk replies
----------------------------------

Single elements of a multi bulk reply may have -1 length, in order to signal that this elements are missing and not empty strings. This can happen with the SORT command when used with the GET _pattern_ option when the specified key is missing. Example of a multi bulk reply containing an empty element:  
  

S: 3
S: 3
S: foo
S: -1
S: 3
S: bar

The second element is nul. The client library should return something like this:  
  

\["foo",nil,"bar"\]

Multi-Bulk replies errors
-------------------------

Like bulk reply errors Multi-bulk reply errors are reported using a negative count. Example:  
  

C: LRANGE stringkey 0 1
S: -38
S: -ERR Requested element is not a string

The following commands reply with a multi-bulk reply: LRANGE, LINTER  
  
Check the Bulk replies errors section for more information.

Status code reply
-----------------

As already seen a status code reply is in the form of a single line string terminated by "\\r\\n". For example:  
  

+OK

and  
  

\-ERR no suck key

are two examples of status code replies. The first character of a status code reply is always "+" or "-".  
  
The following commands reply with a status code reply: PING, SET, SELECT, SAVE, BGSAVE, SHUTDOWN, RENAME, LPUSH, RPUSH, LSET, LTRIM

Integer reply
-------------

This type of reply is just a CRLF terminated string representing an integer. For example "0\\r\\n", or "1000\\r\\n" are integer replies.  
  
With commands like INCR or LASTSAVE using the integer reply to actually return a value there is no special meaning for the returned integer. It is just an incremental number for INCR, a UNIX time for LASTSAVE and so on.  
  
Some commands like EXISTS will return 1 for true and 0 for false.  
  
Other commands like SADD, SREM and SETNX will return 1 if the operation was actually done, 0 otherwise, and **a negative value** if the operation is invalid (for example SADD against a non-set value), accordingly to this table:

\-1 no such key
-2 operation against the a key holding a value of the wrong type
-3 source and destiantion objects/dbs are the same
-4 argument out of range

In all this cases it is mandatory that the client raises an error instead to pass the negative value to the caller. Please check the commands documentation for the exact behaviour.  
  
The following commands will reply with an integer reply: SETNX, DEL, EXISTS, INCR, INCRBY, DECR, DECRBY, DBSIZE, LASTSAVE, RENAMENX, MOVE, LLEN, SADD, SREM, SISMEMBER, SCARD  
  
The commands that will never return a negative integer (commands that can't fail) are: INCR, DECR, INCRBY, DECRBY, LASTSAVE, EXISTS, SETNX, DEL, DBSIZE.

Single line reply
-----------------

This replies are just single line strings terminated by CRLF. Only two commands reply in this way currently, RANDOMKEY and TYPE.

Multiple commands and pipelining
--------------------------------

A client can use the same connection in order to issue multiple commands. Pipelining is supported so multiple commands can be sent with a single write operation by the client, it is not needed to read the server reply in order to issue the next command. All the replies can be read at the end.  
  
Usually Redis server and client will have a very fast link so this is not very important to support this feature in a client implementation, still if an application needs to issue a very large number of commands in short time to use pipelining can be much faster.
