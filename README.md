说明：
-------
添加redis中文注释，本文件是原始redis说明的描述以及部分翻译注释，具体的代码阅读文档见[code_analysis.md](https://github.com/firejh/redis7.0_code_analysis/edit/main/README.md)说明，这里会整体概括阅读总结。另外，源码内部也会部分添加注释。


This README is just a fast *quick start* document. You can find more detailed documentation at [redis.io](https://redis.io).

What is Redis?
--------------
Redis是何物？

Redis is often referred to as a *data structures* server. What this means is that Redis provides access to mutable data structures via a set of commands, which are sent using a *server-client* model with TCP sockets and a simple protocol. So different processes can query and modify the same data structures in a shared way.
Redis通常被称为一个*数据结构*服务器。意味着Redis通过一组命令提供了对可变数据结构的访问，这些命令是使用一个带有TCP套接字和简单协议的*server-client*模型发送的。

Data structures implemented into Redis have a few special properties:
在Redis中实现的数据结构有几个特殊的属性:

* Redis cares to store them on disk, even if they are always served and modified into the server memory. This means that Redis is fast, but that it is also non-volatile.
Redis关心磁盘存储，即使他们总是操作内存。这意味着Redis是快速的，但它也是非易失性的。（总结是可以磁盘存储，数据会不丢失。但是不丢失的代价是什么，这是要探究的。）
* The implementation of data structures emphasizes memory efficiency, so data structures inside Redis will likely use less memory compared to the same data structure modelled using a high-level programming language.
数据结构的实现强调了内存效率，因此与使用高级编程语言建模的相同数据结构相比，Redis内部的数据结构可能会使用更少的内存。（使用自己的数据结构，比c++的STL容器更适合redis，性能会更优。）
* Redis offers a number of features that are natural to find in a database, like replication, tunable levels of durability, clustering, and high availability.
Redis提供了许多在数据库中很容易找到的特性，比如复制、可调级别的持久性、集群和高可用性。（一些支持的特性）

Another good example is to think of Redis as a more complex version of memcached, where the operations are not just SETs and GETs, but operations that work with complex data types like Lists, Sets, ordered data structures, and so forth.
类似memecached但是功能更多，直接忽略吧~

If you want to know more, this is a list of selected starting points:

* Introduction to Redis data types. https://redis.io/topics/data-types-intro
介绍Redis数据类型
* Try Redis directly inside your browser. https://try.redis.io
直接在浏览器中尝试Redis，网页版客户端
* The full list of Redis commands. https://redis.io/commands
Redis命令的完整列表，会经常用的
* There is much more inside the official Redis documentation. https://redis.io/documentation
官方文档

Building Redis
--------------
编译Redis

Redis can be compiled and used on Linux, OSX, OpenBSD, NetBSD, FreeBSD.
We support big endian and little endian architectures, and both 32 bit
and 64 bit systems.
支持各类系统、大小端架构、32和64位等（后面只会针对linux代码研究、大小端、位数牵扯一些性能考虑会适当研究）

It may compile on Solaris derived systems (for instance SmartOS) but our
support for this platform is *best effort* and Redis is not guaranteed to
work as well as in Linux, OSX, and \*BSD.
Solaris支持的不够好

It is as simple as:
最简单编译（通常默认的一般是符合多数应用场景）
    % make

To build with TLS support, you'll need OpenSSL development libraries (e.g.
libssl-dev on Debian/Ubuntu) and run:
TSL：安全传输层协议
想加密传输要以来openssl（这个感觉没有必要，肯定影响性能，数据操一般是内网通信，阅读也可忽略此部分，至于openssl的安全传输可以单独研究）

    % make BUILD_TLS=yes

To build with systemd support, you'll need systemd development libraries (such 
as libsystemd-dev on Debian/Ubuntu or systemd-devel on CentOS) and run:
systemd：是 Linux 系统工具,用来启动守护进程。（这个linux应该经常看到，了解一下不是坏事）
忽略~
    % make USE_SYSTEMD=yes

To append a suffix to Redis program names, use:
redis可执行文件添加后缀（编译有configure文件的程序时候常见--program-prefix就是加前缀，这里是后缀，应该都是来自configure命令，configure来自哪里就不查了~）

    % make PROG_SUFFIX="-alt"

You can build a 32 bit Redis binary using:
32位编译

    % make 32bit

After building Redis, it is a good idea to test it using:
编译好redis后可以编译测试程序（一些测试代码）
    % make test

If TLS is built, running the tests with TLS enabled (you will need `tcl-tls`
installed):
如果是使用TLS版本redis（要先执行脚本生成证书，测试程序也要加tls参数，可以试试）
    % ./utils/gen-test-certs.sh
    % ./runtest --tls


Fixing build problems with dependencies or cached build options
---------
修复依赖项或缓存构建选项的构建问题（deps下有一些依赖，如果有改动需要make clean，这个我们自己也经常遇到，可以单独研究一下g++的编译依赖）

Redis has some dependencies which are included in the `deps` directory.
`make` does not automatically rebuild dependencies even if something in
the source code of dependencies changes.

When you update the source code with `git pull` or when code inside the
dependencies tree is modified in any other way, make sure to use the following
command in order to really clean everything and rebuild from scratch:

    make distclean

This will clean: jemalloc, lua, hiredis, linenoise.

Also if you force certain build options like 32bit target, no C compiler
optimizations (for debugging purposes), and other similar build time options,
those options are cached indefinitely until you issue a `make distclean`
command.

Fixing problems building 32 bit binaries
---------
要 make distclean

If after building Redis with a 32 bit target you need to rebuild it
with a 64 bit target, or the other way around, you need to perform a
`make distclean` in the root directory of the Redis distribution.

In case of build errors when trying to build a 32 bit binary of Redis, try
the following steps:

* Install the package libc6-dev-i386 (also try g++-multilib).
* Try using the following command line instead of `make 32bit`:
  `make CFLAGS="-m32 -march=native" LDFLAGS="-m32"`

Allocator
---------
内存分配器，默认使用jemalloc，可以减少内存碎片

Selecting a non-default memory allocator when building Redis is done by setting
the `MALLOC` environment variable. Redis is compiled and linked against libc
malloc by default, with the exception of jemalloc being the default on Linux
systems. This default was picked because jemalloc has proven to have fewer
fragmentation problems than libc malloc.

To force compiling against libc malloc, use:

    % make MALLOC=libc

To compile against jemalloc on Mac OS X systems, use:

    % make MALLOC=jemalloc

Monotonic clock
---------------
单调的时钟，clock_gettime经过linux和x86的优化不需要用户和内核的上线文切换，性能得到优化，但是性能代价依然搞，无法满足一些性能敏感的应用程序的延迟测量需求。
可以使用处理器时钟。

By default, Redis will build using the POSIX clock_gettime function as the
monotonic clock source.  On most modern systems, the internal processor clock
can be used to improve performance.  Cautions can be found here: 
    http://oliveryang.net/2015/09/pitfalls-of-TSC-usage/

To build with support for the processor's internal instruction clock, use:

    % make CFLAGS="-DUSE_PROCESSOR_CLOCK"

Verbose build
-------------
构建，可以看详细的输出

Redis will build with a user-friendly colorized output by default.
If you want to see a more verbose output, use the following:

    % make V=1

Running Redis
-------------
启动命令

To run Redis with the default configuration, just type:

    % cd src
    % ./redis-server

If you want to provide your redis.conf, you have to run it using an additional
parameter (the path of the configuration file):

    % cd src
    % ./redis-server /path/to/redis.conf

It is possible to alter the Redis configuration by passing parameters directly
as options using the command line. Examples:

    % ./redis-server --port 9999 --replicaof 127.0.0.1 6379
    % ./redis-server /etc/redis/6379.conf --loglevel debug

All the options in redis.conf are also supported as options using the command
line, with exactly the same name.

Running Redis with TLS:
------------------
TLS（Transport Layer Security，安全传输层)，TLS是建立在传输层TCP协议之上的协议，服务于应用层，它的前身是SSL（Secure Socket Layer，安全套接字层），它实现了将应用层的报文进行加密后再交由TCP进行传输的功能。加密一般是不需要的。

Please consult the [TLS.md](TLS.md) file for more information on
how to use Redis with TLS.

Playing with Redis
------------------
客户端操作

You can use redis-cli to play with Redis. Start a redis-server instance,
then in another terminal try the following:

    % cd src
    % ./redis-cli
    redis> ping
    PONG
    redis> set foo bar
    OK
    redis> get foo
    "bar"
    redis> incr mycounter
    (integer) 1
    redis> incr mycounter
    (integer) 2
    redis>

You can find the list of all the available commands at https://redis.io/commands.

Installing Redis
-----------------
安装，可以编译时指定安装目录

In order to install Redis binaries into /usr/local/bin, just use:

    % make install

You can use `make PREFIX=/some/other/directory install` if you wish to use a
different destination.

Make install will just install binaries in your system, but will not configure
init scripts and configuration files in the appropriate place. This is not
needed if you just want to play a bit with Redis, but if you are installing
it the proper way for a production system, we have a script that does this
for Ubuntu and Debian systems:

    % cd utils
    % ./install_server.sh

_Note_: `install_server.sh` will not work on Mac OSX; it is built for Linux only.

The script will ask you a few questions and will setup everything you need
to run Redis properly as a background daemon that will start again on
system reboots.

You'll be able to stop and start Redis using the script named
`/etc/init.d/redis_<portnumber>`, for instance `/etc/init.d/redis_6379`.

Code contributions
-----------------
代码共享方式

Note: By contributing code to the Redis project in any form, including sending
a pull request via Github, a code fragment or patch via private email or
public discussion groups, you agree to release your code under the terms
of the BSD license that you can find in the [COPYING][1] file included in the Redis
source distribution.

Please see the [CONTRIBUTING][2] file in this source distribution for more
information. For security bugs and vulnerabilities, please see [SECURITY.md][3].

[1]: https://github.com/redis/redis/blob/unstable/COPYING
[2]: https://github.com/redis/redis/blob/unstable/CONTRIBUTING
[3]: https://github.com/redis/redis/blob/unstable/SECURITY.md

Redis internals
===
下面将对代码说明，但是只是一个高层面的说明，不会有代码细节

If you are reading this README you are likely in front of a Github page
or you just untarred the Redis distribution tar ball. In both the cases
you are basically one step away from the source code, so here we explain
the Redis source code layout, what is in each file as a general idea, the
most important functions and structures inside the Redis server and so forth.
We keep all the discussion at a high level without digging into the details
since this document would be huge otherwise and our code base changes
continuously, but a general idea should be a good starting point to
understand more. Moreover most of the code is heavily commented and easy
to follow.

Source code layout
---
源代码布局，根据路介绍

The Redis root directory just contains this README, the Makefile which
calls the real Makefile inside the `src` directory and an example
configuration for Redis and Sentinel. You can find a few shell
scripts that are used in order to execute the Redis, Redis Cluster and
Redis Sentinel unit tests, which are implemented inside the `tests`
directory.

Inside the root are the following important directories:

* `src`: contains the Redis implementation, written in C. 代码实现目录
* `tests`: contains the unit tests, implemented in Tcl.测试
* `deps`: contains libraries Redis uses. Everything needed to compile Redis is inside this directory; your system just needs to provide `libc`, a POSIX compatible interface and a C compiler. Notably `deps` contains a copy of `jemalloc`, which is the default allocator of Redis under Linux. Note that under `deps` there are also things which started with the Redis project, but for which the main repository is not `redis/redis`. 一些库

There are a few more directories but they are not very important for our goals
here. We'll focus mostly on `src`, where the Redis implementation is contained,
exploring what there is inside each file. The order in which files are
exposed is the logical one to follow in order to disclose different layers
of complexity incrementally.

Note: lately Redis was refactored quite a bit. Function names and file
names have been changed, so you may find that this documentation reflects the
`unstable` branch more closely. For instance, in Redis 3.0 the `server.c`
and `server.h` files were named `redis.c` and `redis.h`. However the overall
structure is the same. Keep in mind that all the new developments and pull
requests should be performed against the `unstable` branch.

server.h
---
The simplest way to understand how a program works is to understand the
data structures it uses. So we'll start from the main header file of
Redis, which is `server.h`.
宏定义、数据结构、全局变量、全局函数等等，是阅读代码首先要看的

All the server configuration and in general all the shared state is
defined in a global structure called `server`, of type `struct redisServer`.
A few important fields in this structure are:
所有的服务器配置和共享状态通常都是
定义在一个名为“server”的全局结构中，类型为“struct redisServer”。
这个结构中的几个重要领域是:

* `server.db` is an array of Redis databases, where data is stored.是一个Redis数据库数组，数据存储在那里。
* `server.commands` is the command table.是命令表。
* `server.clients` is a linked list of clients connected to the server.连接到服务器的客户端的链表。
* `server.master` is a special client, the master, if the instance is a replica.主从用，如果是备份示例，这个是主示例的链接


There are tons of other fields. Most fields are commented directly inside
the structure definition. 其他的看代码注释

Another important Redis data structure is the one defining a client.
In the past it was called `redisClient`, now just `client`. The structure
has many fields, here we'll just show the main ones:
另一个重要的Redis数据结构是定义客户端。
在过去它被称为“redisClient”，现在只是“client”。结构
有很多字段，这里我们只展示主要的
```c
struct client {
    int fd;
    sds querybuf;
    int argc;
    robj **argv;
    redisDb *db;
    int flags;
    list *reply;
    // ... many other fields ...
    char buf[PROTO_REPLY_CHUNK_BYTES];
}
```
The client structure defines a *connected client*:


* The `fd` field is the client socket file descriptor.客户端套接字文件描述符执
* `argc` and `argv` are populated with the command the client is executing, so that functions implementing a given Redis command can read the arguments.启动参数
* `querybuf` accumulates the requests from the client, which are parsed by the Redis server according to the Redis protocol and executed by calling the implementations of the commands the client is executing.客户端的请求
* `reply` and `buf` are dynamic and static buffers that accumulate the replies the server sends to the client. These buffers are incrementally written to the socket as soon as the file descriptor is writeable.收发缓冲区

As you can see in the client structure above, arguments in a command
are described as `robj` structures. The following is the full `robj`
structure, which defines a *Redis object*:

    typedef struct redisObject {
        unsigned type:4;
        unsigned encoding:4;
        unsigned lru:LRU_BITS; /* lru time (relative to server.lruclock) */
        int refcount;
        void *ptr;
    } robj;

Basically this structure can represent all the basic Redis data types like
strings, lists, sets, sorted sets and so forth. The interesting thing is that
it has a `type` field, so that it is possible to know what type a given
object has, and a `refcount`, so that the same object can be referenced
in multiple places without allocating it multiple times. Finally the `ptr`
field points to the actual representation of the object, which might vary
even for the same type, depending on the `encoding` used.
基本上这个结构可以表示所有基本的Redis数据类型字符串、列表、集合、排序集合等等。
有趣的是它有一个'type'字段，这样就可以知道给定的类型是什么对象，
有并且有一个'refcount'，这样同一个对象可以被引用在多个地方，而不是多次分配它。
最后,'ptr'字段指向对象，这可能会有所不同即使是相同的类型，也取决于所使用的'encoding(编码)'。

Redis objects are used extensively in the Redis internals, however in order
to avoid the overhead of indirect accesses, recently in many places
we just use plain dynamic strings not wrapped inside a Redis object.
Redis对象在Redis内部被广泛使用，但是为了避免间接访问的开销，最近在很多地方我们只是使用普通的动态字符串，没有包装在Redis对象中

server.c
---

This is the entry point of the Redis server, where the `main()` function
is defined. The following are the most important steps in order to startup
the Redis server.
这是Redis服务器的入口点，这里定义了' main() '函数。以下是启动Redis服务器最重要的步骤。

* `initServerConfig()` sets up the default values of the `server` structure.设置' server '结构的默认值
* `initServer()` allocates the data structures needed to operate, setup the listening socket, and so forth.分配操作所需的数据结构，设置侦听套接字，等等。
* `aeMain()` starts the event loop which listens for new connections.启动监听新连接的事件循环。


There are two special functions called periodically by the event loop:
事件循环定期调用两个特殊函数:

1. `serverCron()` is called periodically (according to `server.hz` frequency), and performs tasks that must be performed from time to time, like checking for timed out clients.定期被调用(根据'server.hz'的频率)，并执行必须时刻执行的任务，如检查超时的客户端。
2. `beforeSleep()` is called every time the event loop fired, Redis served a few requests, and is returning back into the event loop.在每次事件循环触发时被调用，一些请求结果也会在事件循环中返回

Inside server.c you can find code that handles other vital things of the Redis server:
在server.c中，你可以找到处理Redis服务器其他重要事情的代码:

* `call()` is used in order to call a given command in the context of a given client.调用客户端给的命令
* `activeExpireCycle()` handles eviction of keys with a time to live set via the `EXPIRE` command.处理有expire设置的key
* `performEvictions()` is called when a new write command should be performed but Redis is out of memory according to the `maxmemory` directive.内存不足时处理
* The global variable `redisCommandTable` defines all the Redis commands, specifying the name of the command, the function implementing the command, the number of arguments required, and other properties of each command.定义了所有Redis命令，包括命令名、实现命令的函数、所需参数的数量以及每个命令的其他属性。

commands.c
---
This file is auto generated by utils/generate-command-code.py, the content is based on the JSON files in the src/commands folder.
These are meant to be the single source of truth about the Redis commands, and all the metadata about them.
These JSON files are not meant to be used directly by anyone directly, instead that metadata can be obtained via the COMMAND command.
该文件由utils/generate-command-code.py自动生成，内容基于src/commands文件夹中的JSON文件。
这些是关于Redis命令的唯一真实来源，以及所有关于它们的元数据。
这些JSON文件不打算由任何人直接使用，而是可以通过COMMAND命令获得元数据。

networking.c
---

This file defines all the I/O functions with clients, masters and replicas(which in Redis are just special clients):
这个文件定义了所有的I/O功能与客户端、主从(这在Redis只是特殊的客户端):

* `createClient()` allocates and initializes a new client.分配并初始化一个新客户端
* the `addReply*()` family of functions are used by command implementations in order to append data to the client structure, that will be transmitted to the client as a reply for a given command executed.系列函数被命令实现用于向客户端结构添加数据，这些数据将作为对执行的给定命令的回复传输给客户端。
* `writeToClient()` transmits the data pending in the output buffers to the client and is called by the *writable event handler* `sendReplyToClient()`.将输出缓冲区中挂起的数据传输给客户端，并由*writable event handeler* 'sendReplyToClient()'调用
* `readQueryFromClient()` is the *readable event handler* and accumulates data read from the client into the query buffer.一个读事件处理程序，它将从客户端读取的数据积累到query buffer.
* `processInputBuffer()` is the entry point in order to parse the client query buffer according to the Redis protocol. Once commands are ready to be processed, it calls `processCommand()` which is defined inside `server.c` in order to actually execute the command.根据Redis协议解析客户端查询缓冲区的入口。一旦解析到一个命令，会调用' processCommand() '（在' server.c '中定义的）。
* `freeClient()` deallocates, disconnects and removes a client.释放、断开连、移除客户端

aof.c and rdb.c
---

As you can guess from the names, these files implement the RDB and AOF
persistence for Redis. Redis uses a persistence model based on the `fork()`
system call in order to create a process with the same (shared) memory
content of the main Redis process. This secondary process dumps the content
of the memory on disk. This is used by `rdb.c` to create the snapshots
on disk and by `aof.c` in order to perform the AOF rewrite when the
append only file gets too big.
从名字中你可以猜到，这些文件实现了Redis的RDB和AOF持久性。Redis使用一个基于' fork() '系统调用的持久化模型来创建一个与主Redis进程相同(共享)内存内容的进程。
这个辅助进程将内存的内容转储到磁盘上。
'rdb.c'使用该函数在磁盘上创建快照，'aof.c'使用该函数在追加文件太大时执行AOF重写。

The implementation inside `aof.c` has additional functions in order to
implement an API that allows commands to append new commands into the AOF
file as clients execute them.内部的实现有额外的功能，以实现一个API，允许命令在客户端执行AOF文件时附加新命令

The `call()` function defined inside `server.c` is responsible for calling
the functions that in turn will write the commands into the AOF.
在' server.c '中定义的' call() '函数负责调用函数，然后将命令写入AOF

db.c
---

Certain Redis commands operate on specific data types; others are general.
Examples of generic commands are `DEL` and `EXPIRE`. They operate on keys
and not on their values specifically. All those generic commands are
defined inside `db.c`.
某些Redis命令操作特定的数据类型。
通用命令的例子是' DEL '和' EXPIRE '。它们作用于键，而不是它们的值。
所有这些通用命令都定义在' db.c '中。

Moreover `db.c` implements an API in order to perform certain operations
on the Redis dataset without directly accessing the internal data structures.
此外，' db.c '实现了一个API，以便在不直接访问内部数据结构的情况下在Redis数据集上执行某些操作

The most important functions inside `db.c` which are used in many command
implementations are the following:
' db.c '中最重要的函数在许多命令实现中使用如下:

* `lookupKeyRead()` and `lookupKeyWrite()` are used in order to get a pointer to the value associated to a given key, or `NULL` if the key does not exist.获得一个指向给定键关联值的指针，或者' NULL '如果该键不存在。
* `dbAdd()` and its higher level counterpart `setKey()` create a new key in a Redis database.创建新库
* `dbDelete()` removes a key and its associated value.删除一个键及关联值
* `emptyDb()` removes an entire single database or all the databases defined.删除数据库或所有已定义的数据库

The rest of the file implements the generic commands exposed to the client.
文件的其余部分实现向客户机公开的通用命令。

object.c
---

The `robj` structure defining Redis objects was already described. Inside
`object.c` there are all the functions that operate with Redis objects at
a basic level, like functions to allocate new objects, handle the reference
counting and so forth. Notable functions inside this file:
定义Redis对象的' robj '结构已经描述过了。在' object.c '里面有所有在基本层面上操作Redis对象的函数，比如分配新对象的函数，处理引用计数等等。该文件中值得注意的函数:

* `incrRefCount()` and `decrRefCount()` are used in order to increment or decrement an object reference count. When it drops to 0 the object is finally freed.
增加或减少对象引用计数。当它降为0时，对象最终被释放。
* `createObject()` allocates a new object. There are also specialized functions to allocate string objects having a specific content, like `createStringObjectFromLongLong()` and similar functions.
* 分配一个新对象。还有一些专门的函数来分配具有特定内容的字符串对象，比如' createStringObjectFromLongLong() '和类似的函数

This file also implements the `OBJECT` command.
该文件还实现了“OBJECT”命令。

replication.c
---

This is one of the most complex files inside Redis, it is recommended to
approach it only after getting a bit familiar with the rest of the code base.
In this file there is the implementation of both the master and replica role
of Redis.
这是Redis中最复杂的文件之一，建议在熟悉了其余的代码基础后再使用它。
在这个文件中有Redis的主副本角色的实现。

One of the most important functions inside this file is `replicationFeedSlaves()` that writes commands to the clients representing replica instances connected
to our master, so that the replicas can get the writes performed by the clients:
this way their data set will remain synchronized with the one in the master.
这个文件中最重要的一个函数是replicationFeedSlaves()，它把命令写给连接到主服务器的副本实例的客户端，这样副本就可以得到客户端执行的写操作:
通过这种方式，它们的数据集将与主服务器中的数据集保持同步。

This file also implements both the `SYNC` and `PSYNC` commands that are
used in order to perform the first synchronization between masters and
replicas, or to continue the replication after a disconnection.
该文件还实现了“SYNC”和“PSYNC”命令，用于在主副本和副本之间执行第一次同步，或在断开连接后继续复制。

Script
脚本实现，暂时忽略
---
The script unit is compose of 3 units
* `script.c` - integration of scripts with Redis (commands execution, set replication/resp, ..)
* `script_lua.c` - responsible to execute Lua code, uses script.c to interact with Redis from within the Lua code.
* `function_lua.c` - contains the Lua engine implementation, uses script_lua.c to execute the Lua code.
* `functions.c` - Contains Redis Functions implementation (FUNCTION command), uses functions_lua.c if the function it wants to invoke needs the Lua engine.
* `eval.c` - Contains the `eval` implementation using `script_lua.c` to invoke the Lua code.


Other C files
---

* `t_hash.c`, `t_list.c`, `t_set.c`, `t_string.c`, `t_zset.c` and `t_stream.c` contains the implementation of the Redis data types. They implement both an API to access a given data type, and the client command implementations for these data types.
redis数据类型的实现
* `ae.c` implements the Redis event loop, it's a self contained library which is simple to read and understand.
实现了Redis事件循环，它是一个自包含的库，易于阅读和理解
* `sds.c` is the Redis string library, check https://github.com/antirez/sds for more information.
字符串库
* `anet.c` is a library to use POSIX networking in a simpler way compared to the raw interface exposed by the kernel.
与内核公开的原始接口相比，它以更简单的方式使用POSIX网络。
* `dict.c` is an implementation of a non-blocking hash table which rehashes incrementally.
非阻塞哈希表的实现，它会递增地重新哈希。
* `cluster.c` implements the Redis Cluster. Probably a good read only after being very familiar with the rest of the Redis code base. If you want to read `cluster.c` make sure to read the [Redis Cluster specification][4].
* 先了解用法和其他功能后再阅读

[4]: https://redis.io/topics/cluster-spec

Anatomy of a Redis command
---

All the Redis commands are defined in the following way:

    void foobarCommand(client *c) {
        printf("%s",c->argv[1]->ptr); /* Do something with the argument. */
        addReply(c,shared.ok); /* Reply something to the client. */
    }

The command is then referenced inside `server.c` in the command table:

    {"foobar",foobarCommand,2,"rtF",0,NULL,0,0,0,0,0},

In the above example `2` is the number of arguments the command takes,
while `"rtF"` are the command flags, as documented in the command table
top comment inside `server.c`.

After the command operates in some way, it returns a reply to the client,
usually using `addReply()` or a similar function defined inside `networking.c`.

There are tons of command implementations inside the Redis source code
that can serve as examples of actual commands implementations. Writing
a few toy commands can be a good exercise to get familiar with the code base.

There are also many other files not described here, but it is useless to
cover everything. We just want to help you with the first steps.
Eventually you'll find your way inside the Redis code base :-)

Enjoy!
