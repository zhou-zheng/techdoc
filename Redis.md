# **Redis 学习文档**
REDIS = REmote DIctionary Server

## **Redis 安装**
实操环境使用的是 Vagrant Centos/7，安装主要步骤如下：
```
$ wget http://download.redis.io/releases/redis-4.0.6.tar.gz
$ tar xzf redis-4.0.6.tar.gz
$ cd redis-4.0.6
$ make
$ make test
$ sudo make install
$ sudo ./utils/install_server.sh
```
在实际操作过程中，可能遇到如下问题：<br />
1）` /bin/sh: cc: command not found `<br />
问题原因：虚拟机系统中缺少gcc<br />
解决方法：` sudo yum -y install gcc automake autoconf libtool make `<br />
2）` zmalloc.h:50:31: fatal error: jemalloc/jemalloc.h: No such file or directory `<br />
问题原因：在 README 有这么一段话：
> Allocator<br />
Selecting a non-default memory allocator when building Redis is done by setting  
the \`MALLOC\` environment variable. Redis is compiled and linked against libc  
malloc by default, with the exception of jemalloc being the default on Linux  
systems. This default was picked because jemalloc has proven to have fewer  
fragmentation problems than libc malloc.<br />
To force compiling against libc malloc, use:<br /> 
&emsp;&emsp;% make MALLOC=libc<br />
To compile against jemalloc on Mac OS X systems, use:<br />
&emsp;&emsp;% make MALLOC=jemalloc

意思是说关于分配器 allocator， 如果有 MALLOC 这个环境变量，会用这个环境变量去建立Redis。
而且 libc 并不是默认的分配器，默认的是 jemalloc, 因为 jemalloc 被证明比 libc 有更少的 fragmentation problems。但是如果你没有 jemalloc 而只有 libc，当然 make 出错。<br />
解决方法：` make MALLOC=libc `<br />
3）make test 时出错 ` You need tcl 8.5 or newer in order to run the Redis test `<br />
解决方法：` sudo yum -y install tcl `<br />
4）make install 后，redis 相关命令均被安装到 /usr/local/bin/ 下。<br />
5）通过 Redis 自带的 install_server.sh 脚本配置 Redis 能随系统自启动。<br />
注意：make命令执行完成编译后，会在src目录下生成6个可执行文件，分别是redis-server、redis-cli、redis-benchmark、redis-check-aof、redis-check-rdb、redis-sentinel。

## **Redis 支持的数据结构**
- strings
- hashes
- lists
- sets
- sorted sets
- bitmaps
- hyperloglogs
- geospatial indexes

## **Redis 杂谈**
### **Redis-cli**
以下内容摘抄翻译自[redis-cli, the Redis command line interface](https://redis.io/topics/rediscli)<br />
- 从其他程序获取对 redis-cli 的输入<br />
`redis-cli -x set foo < /etc/services`<br />
`cat /tmp/commands.txt | redis-cli`<br />
- 反复运行同一命令<br />
`redis-cli -r 5 incr foo`
`redis-cli -r -1 -i 10 INFO | grep rss_human`
- 批量插入数据
参考 [mass insertion guide](https://redis.io/topics/mass-insert)<br />
- 导出 csv 数据<br />
`redis-cli --csv lrange mylist 0 -1`
- 运行 Lua 脚本<br />
参考 [Redis Lua debugger documentation](https://redis.io/topics/ldb)<br />
- 命令帮助<br />
`help @list`<br />
`help lpush`<br />
- 监控统计<br />
`redis-cli --stat`<br />
- 查找大键值<br />
`redis-cli --bigkeys`<br />
- 获取键列表<br />
`redis-cli --scan --pattern 'user:*'`<br />
- Pub/Sub 模式<br />
`redis-cli psubscribe '*'`<br />
- 监控 Redis 中执行的命令<br />
`redis-cli monitor`<br />
- 监控 Redis 实例的延迟<br />
`redis-cli --latency`<br />
`redis-cli --latency-history`<br />
`redis-cli --latency-dist`<br />
`redis-cli --intrinsic-latency <test-time>`<br />
- 远程备份 RDB 文件<br />
`redis-cli --rdb /tmp/dump.rdb`<br />
- Slave 模式<br />
`redis-cli --slave`
- LRU 模拟<br />
`redis-cli --lru-test 10000000`<br />

## **Redis 配置**
- [redis.conf for Redis 4.0](https://raw.githubusercontent.com/antirez/redis/4.0/redis.conf)<br />
- [redis.conf for Redis 3.2](https://raw.githubusercontent.com/antirez/redis/3.2/redis.conf)<br />
- 通过命令行传递参数<br />
`./redis-server --port 6380 --slaveof 127.0.0.1 6379`<br />
- 配置 Redis 作缓存<br />
在 redis.conf 中作如下配置，这样就很接近 memcached，参考 [using Redis as an LRU cache](https://redis.io/topics/lru-cache)：<br />
`maxmemory 2mb`<br />
`maxmemory-policy allkeys-lru`<br />

- 用 redis-benchmark + SET 随机填充一百万个 key：<br />
`redis-benchmark -t set -n 1000000 -r 100000000 `

以下内容摘自 [Redis The full list of commands](https://redis.io/topics/data-types-intro)
- > **Redis Lists**<br />
Redis Lists are implemented with linked lists because for a database system it is crucial to be able to add elements to a very long list in a very fast way. Another strong advantage, as you'll see in a moment, is that Redis Lists can be taken at constant length in constant time.<br />
When fast access to the middle of a large collection of elements is important, there is a different data structure that can be used, called sorted sets.
- > **Common use cases for lists**<br />
Lists are useful for a number of tasks, two very representative use cases are the following:<br />
&emsp;&emsp; · Remember the latest updates posted by users into a social network.<br />
&emsp;&emsp; · Communication between processes, using a consumer-producer pattern where the producer pushes items into a list, and a consumer (usually a worker) consumes those items and executed actions. Redis has special list commands to make this use case both more reliable and efficient.<br />
To describe a common use case step by step, imagine your home page shows the latest photos published in a photo sharing social network and you want to speedup access.<br />
&emsp;&emsp; · Every time a user posts a new photo, we add its ID into a list with LPUSH.<br />
&emsp;&emsp; · When users visit the home page, we use LRANGE 0 9 in order to get the latest 10 posted items.
- > **Capped lists**<br />
In many use cases we just want to use lists to store the latest items, whatever they are: social network updates, logs, or anything else.<br />
Redis allows us to use lists as a capped collection, only remembering the latest N items and discarding all the oldest items using the LTRIM command.<br />
The LTRIM command is similar to LRANGE, but instead of displaying the specified range of elements it sets this range as the new list value. All the elements outside the given range are removed.<br />
This allows for a very simple but useful pattern: doing a List push operation + a List trim operation together in order to add a new element and discard elements exceeding a limit:<br />
`LPUSH mylist <some element>`<br />
`LTRIM mylist 0 999`<br />
- > **Blocking operations on lists**<br />
Lists have a special feature that make them suitable to implement queues, and in general as a building block for inter process communication systems: blocking operations.<br />
Imagine you want to push items into a list with one process, and use a different process in order to actually do some kind of work with those items. This is the usual producer / consumer setup, and can be implemented in the following simple way:<br />
&emsp;&emsp; · To push items into the list, producers call LPUSH.<br />
&emsp;&emsp; · To extract / process items from the list, consumers call RPOP.<br />
However it is possible that sometimes the list is empty and there is nothing to process, so RPOP just returns NULL. In this case a consumer is forced to wait some time and retry again with RPOP. This is called polling, and is not a good idea in this context because it has several drawbacks:<br />
&emsp;&emsp; · Forces Redis and clients to process useless commands (all the requests when the list is empty will get no actual work done, they'll just return NULL).<br />
&emsp;&emsp; · Adds a delay to the processing of items, since after a worker receives a NULL, it waits some time. To make the delay smaller, we could wait less between calls to RPOP, with the effect of amplifying problem number 1, i.e. more useless calls to Redis.<br />
So Redis implements commands called BRPOP and BLPOP which are versions of RPOP and LPOP able to block if the list is empty: they'll return to the caller only when a new element is added to the list, or when a user-specified timeout is reached.<br />
This is an example of a BRPOP call we could use in the worker:<br />
`> brpop tasks 5`<br />
`1) "tasks"`<br />
`2) "do_something"`<br />
`(4.32s)`<br />
It means: "wait for elements in the list tasks, but return if after 5 seconds no element is available".<br />
Note that you can use 0 as timeout to wait for elements forever, and you can also specify multiple lists and not just one, in order to wait on multiple lists at the same time, and get notified when the first list receives an element.<br />
A few things to note about BRPOP:<br />
&emsp;&emsp; · Clients are served in an ordered way: the first client that blocked waiting for a list, is served first when an element is pushed by some other client, and so forth.<br />
&emsp;&emsp; · The return value is different compared to RPOP: it is a two-element array since it also includes the name of the key, because BRPOP and BLPOP are able to block waiting for elements from multiple lists.<br />
&emsp;&emsp; · If the timeout is reached, NULL is returned.<br />
- > **Redis Hashes**<br />
Redis hashes look exactly how one might expect a "hash" to look, with field-value pairs<br />
While hashes are handy to represent objects, actually the number of fields you can put inside a hash has no practical limits (other than available memory), so you can use hashes in many different ways inside your application.<br />
It is worth noting that small hashes (i.e., a few elements with small values) are encoded in special way in memory that make them very memory efficient.<br />
- > **Redis Sets**<br />
Redis Sets are **unordered** collections of strings.<br />
Sets are good for expressing relations between objects. <br />
The `SADD` command adds new elements to a set.<br />
The `SINTER` command performs the intersection between different sets.<br />
In addition to intersection you can also perform unions, difference, extract a random element, and so forth.<br />
The command to extract an element is called `SPOP`, and is handy to model certain problems. For example in order to implement a web-based poker game, you may want to represent your deck with a set. Imagine we use a one-char prefix for (C)lubs, (D)iamonds, (H)earts, (S)pades:<br />
`>  sadd deck C1 C2 C3 C4 C5 C6 C7 C8 C9 C10 CJ CQ CK`<br />
&emsp; `D1 D2 D3 D4 D5 D6 D7 D8 D9 D10 DJ DQ DK H1 H2 H3`<br />
&emsp; `H4 H5 H6 H7 H8 H9 H10 HJ HQ HK S1 S2 S3 S4 S5 S6`<br />
&emsp; `S7 S8 S9 S10 SJ SQ SK`<br />
&emsp; `(integer) 52`<br />
Now we want to provide each player with 5 cards. The `SPOP` command removes a random element, returning it to the client, so it is the perfect operation in this case.<br />
The `SUNIONSTORE` command normally performs the union between multiple sets, and stores the result into another set.<br />
The `SCARD` command provides the number of elements inside a set.<br />
When you need to just get random elements without removing them from the set, there is the `SRANDMEMBER` command suitable for the task. It also features the ability to return both repeating and non-repeating elements.<br />
- > **Redis Sorted sets**<br />
Sorted sets are a data type which is similar to a mix between a Set and a Hash. Like sets, sorted sets are composed of unique, non-repeating string elements, so in some sense a sorted set is a set as well.<br />
However while elements inside sets are not ordered, every element in a sorted set is associated with a floating point value, called the score (this is why the type is also similar to a hash, since every element is mapped to a value).<br />
Moreover, elements in a sorted sets are taken in order (so they are not ordered on request, order is a peculiarity of the data structure used to represent sorted sets). They are ordered according to the following rule:<br />
&emsp;&emsp; · If A and B are two elements with a different score, then A > B if A.score is > B.score.<br />
&emsp;&emsp; · If A and B have exactly the same score, then A > B if the A string is lexicographically greater than the B string. A and B strings can't be equal since sorted sets only have unique elements.<br />
The `ZADD` command takes one additional argument (placed before the element to be added) which is the score. <br />
Use `ZREVRANGE` instead of `ZRANGE` to order sorted set the opposite way.<br />
It is possible to return scores as well, using the `WITHSCORES` argument in `ZRANGE`.<br />
Sorted sets are more powerful than this. They can operate on ranges. <br />
The `ZRANGEBYSCORE` command ...<br />
The `ZREMRANGEBYSCORE` command ...<br />
The `ZRANK` and `ZREVRANK` commands ...<br />
The main commands to operate with lexicographical ranges are `ZRANGEBYLEX`, `ZREVRANGEBYLEX`, `ZREMRANGEBYLEX` and `ZLEXCOUNT`.<br />
Using `ZRANGEBYLEX` we can ask for lexicographical ranges.<br />
This feature is important because it allows us to use sorted sets as a generic index. For example, if you want to index elements by a 128-bit unsigned integer argument, all you need to do is to add elements into a sorted set with the same score (for example 0) but with an 16 byte prefix consisting of the 128 bit number in big endian. Since numbers in big endian, when ordered lexicographically (in raw bytes order) are actually ordered numerically as well, you can ask for ranges in the 128 bit space, and get the element's value discarding the prefix.<br />
If you want to see the feature in the context of a more serious demo, check the [Redis autocomplete demo](http://autocomplete.redis.io/).<br />
Sorted Sets are probably **the most advanced** Redis data types, so take some time to check the [full list of Sorted Set commands](https://redis.io/commands#sorted_set) to discover what you can do with Redis! <br />

ALEX ZHOU 注：<br />
In `ZRANGEBYLEX`, valid start and stop must start with ( or [, in order to specify if the range item is respectively exclusive or inclusive. The special values of + or - for start and stop have the special meaning or positively infinite and negatively infinite strings, so for instance the command `ZRANGEBYLEX myzset - +` is guaranteed to return all the elements in the sorted set, if all the elements have the same score.<br />
( 和 [ 代表不包含和包含，后面跟的字符串将被用来进行二进制字符串比较。所以下例中 (London 以写完整的形式在比较，不可理解为类似通配符的样式。
```
127.0.0.1:6379> ZADD mycity 1 Delhi 2 London 3 Paris 4 Tokyo 5 NewYork 6 Seoul
(integer) 6
127.0.0.1:6379> ZRANGE mycity 0 -1
1) "Delhi"
2) "London"
3) "Paris"
4) "Tokyo"
5) "NewYork"
6) "Seoul"
127.0.0.1:6379> ZRANGEBYLEX mycity - +
1) "Delhi"
2) "London"
3) "Paris"
4) "Tokyo"
5) "NewYork"
6) "Seoul"
127.0.0.1:6379> ZRANGEBYLEX mycity "[London" +
1) "London"
2) "Paris"
3) "Tokyo"
4) "NewYork"
5) "Seoul"
127.0.0.1:6379> ZRANGEBYLEX mycity "(London" +
1) "Paris"
2) "Tokyo"
3) "NewYork"
4) "Seoul"
127.0.0.1:6379> ZRANGEBYLEX mycity "(London" "(Seoul"
1) "Paris"
```
```
127.0.0.1:6379> ZADD mycity 1 Delhi 2 London 3 Paris 4 Tokyo 5 NewYork 6 Seoul
(integer) 6
127.0.0.1:6379> ZRANGE mycity 0 -1
1) "Delhi"
2) "London"
3) "Paris"
4) "Tokyo"
5) "NewYork"
6) "Seoul"
127.0.0.1:6379> ZRANGEBYLEX mycity - + LIMIT 0 2
1) "Delhi"
2) "London"
127.0.0.1:6379> ZRANGEBYLEX mycity - + LIMIT 2 3
1) "Paris"
2) "Tokyo"
3) "NewYork"
```
- > **Updating the score: leader boards**<br />
Just a final note about sorted sets before switching to the next topic. Sorted sets' scores can be updated at any time. Just calling ZADD against an element already included in the sorted set will update its score (and position) with O(log(N)) time complexity. As such, sorted sets are suitable when there are tons of updates.<br />
Because of this characteristic a common use case is leader boards. The typical application is a Facebook game where you combine the ability to take users sorted by their high score, plus the get-rank operation, in order to show the top-N users, and the user rank in the leader board (e.g., "you are the #4932 best score here").<br />
- > **Bitmaps**<br />
Bitmaps are not an actual data type, but a set of bit-oriented operations defined on the String type. Since strings are binary safe blobs and their maximum length is 512 MB, they are suitable to set up to 2<sup>32</sup> different bits.
Bit operations are divided into two groups: constant-time single bit operations, like setting a bit to 1 or 0, or getting its value, and operations on groups of bits, for example counting the number of set bits in a given range of bits (e.g., population counting).<br />
One of the biggest advantages of bitmaps is that they often provide extreme space savings when storing information.<br />
Bits are set and retrieved using the `SETBIT` and `GETBIT` command.<br />
The `SETBIT` command takes as its first argument the bit number, and as its second argument the value to set the bit to, which is 1 or 0. The command automatically enlarges the string if the addressed bit is outside the current string length.<br />
`GETBIT` just returns the value of the bit at the specified index. Out of range bits (addressing a bit that is outside the length of the string stored into the target key) are always considered to be zero.<br />
There are three commands operating on group of bits:<br />
`BITOP` performs bit-wise operations between different strings. The provided operations are AND, OR, XOR and NOT.<br />
`BITCOUNT` performs population counting, reporting the number of bits set to 1.<br />
`BITPOS` finds the first bit having the specified value of 0 or 1.<br />
Both `BITPOS` and `BITCOUNT` are able to operate with byte ranges of the string, instead of running for the whole length of the string.<br />
Common use cases for bitmaps are:<br />
&emsp;&emsp; · Real time analytics of all kinds.<br />
&emsp;&emsp; · Storing space efficient but high performance boolean information associated with object IDs.<br />
- > **HyperLogLogs**<br />
A HyperLogLog is a probabilistic data structure used in order to count unique things (technically this is referred to estimating the cardinality of a set). Usually counting unique items requires using an amount of memory proportional to the number of items you want to count, because you need to remember the elements you have already seen in the past in order to avoid counting them multiple times. However there is a set of algorithms that trade memory for precision: you end with an estimated measure with a standard error, which in the case of the Redis implementation is less than 1%. The magic of this algorithm is that you no longer need to use an amount of memory proportional to the number of items counted, and instead can use a constant amount of memory! 12k bytes in the worst case, or a lot less if your HyperLogLog (We'll just call them HLL from now) has seen very few elements.<br />
HLLs in Redis, while technically a different data structure, are encoded as a Redis string, so you can call GET to serialize a HLL, and SET to deserialize it back to the server.<br />
Conceptually the HLL API is like using Sets to do the same task. You would SADD every observed element into a set, and would use SCARD to check the number of elements inside the set, which are unique since SADD will not re-add an existing element.<br />
While you don't really add items into an HLL, because the data structure only contains a state that does not include actual elements, the API is the same:<br />
&emsp;&emsp; · Every time you see a new element, you add it to the count with PFADD.<br />
&emsp;&emsp; · Every time you want to retrieve the current approximation of the unique elements added with PFADD so far, you use the PFCOUNT.<br />
An example of use case for this data structure is counting unique queries performed by users in a search form every day.<br />
- > **SCAN cursor**<br />
The `SCAN` command and the closely related commands `SSCAN`, `HSCAN` and `ZSCAN` are used in order to incrementally iterate over a collection of elements.<br />
&emsp;&emsp; · `SCAN` iterates the set of keys in the currently selected Redis database.<br />
&emsp;&emsp; · `SSCAN` iterates elements of Sets types.<br />
&emsp;&emsp; · `HSCAN` iterates fields of Hash types and their associated values.<br />
&emsp;&emsp; · `ZSCAN` iterates elements of Sorted Set types and their associated scores.<br />
Since these commands allow for incremental iteration, returning only a small number of elements per call, they can be used in production without the downside of commands like `KEYS` or `SMEMBERS` that may block the server for a long time (even several seconds) when called against big collections of keys or elements.<br />
However while blocking commands like `SMEMBERS` are able to provide all the elements that are part of a Set in a given moment, The `SCAN` family of commands only offer limited guarantees about the returned elements since the collection that we incrementally iterate can change during the iteration process.<br />
- > **SCAN basic usage**<br />
`SCAN` is a cursor based iterator. This means that at every call of the command, the server returns an updated cursor that the user needs to use as the cursor argument in the next call.<br />
An iteration starts when the cursor is set to 0, and terminates when the cursor returned by the server is 0.<br />
As you can see the `SCAN` return value is an array of two values: the first value is the new cursor to use in the next call, the second value is an array of elements.<br />
Starting an iteration with a cursor value of 0, and calling `SCAN` until the returned cursor is 0 again is called a full iteration.<br />
- > **Scan guarantees**<br />
The `SCAN` command, and the other commands in the `SCAN` family, are able to provide to the user a set of guarantees associated to full iterations.<br />
&emsp;&emsp; · A full iteration always retrieves all the elements that were present in the collection from the start to the end of a full iteration. This means that if a given element is inside the collection when an iteration is started, and is still there when an iteration terminates, then at some point `SCAN` returned it to the user.<br />
&emsp;&emsp; · A full iteration never returns any element that was NOT present in the collection from the start to the end of a full iteration. So if an element was removed before the start of an iteration, and is never added back to the collection for all the time an iteration lasts, `SCAN` ensures that this element will never be returned.<br />
However because SCAN has very little state associated (just the cursor) it has the following drawbacks:<br />
&emsp;&emsp; · A given element may be returned multiple times. It is up to the application to handle the case of duplicated elements, for example only using the returned elements in order to perform operations that are safe when re-applied multiple times.<br />
&emsp;&emsp; · Elements that were not constantly present in the collection during a full iteration, may be returned or not: it is undefined.<br />
- > **Number of elements returned at every SCAN call**<br />
`SCAN` family functions do not guarantee that the number of elements returned per call are in a given range. The commands are also allowed to return zero elements, and the client should not consider the iteration complete as long as the returned cursor is not zero.<br />
However there is a way for the user to tune the order of magnitude of the number of returned elements per call using the `COUNT` option.<br />
It is important to note that the `MATCH` filter is applied after elements are retrieved from the collection, just before returning data to the client. This means that if the pattern matches very little elements inside the collection, `SCAN` will likely return no elements in most iterations.<br />
- > **Multiple parallel iterations**<br />
It is possible for an infinite number of clients to iterate the same collection at the same time, as the full state of the iterator is in the cursor, that is obtained and returned to the client at every call. Server side no state is taken at all.<br />
- > **Terminating iterations in the middle**<br />
Since there is no state server side, but the full state is captured by the cursor, the caller is free to terminate an iteration half-way without signaling this to the server in any way. An infinite number of iterations can be started and never terminated without any issue.<br />
-> **Calling SCAN with a corrupted cursor**<br />
Calling `SCAN` with a broken, negative, out of range, or otherwise invalid cursor, will result into undefined behavior but never into a crash.<br />

以下内容摘自 [Redis FAQ](https://redis.io/topics/faq)
- > **What's the Redis memory footprint?**<br />
To give you a few examples (all obtained using 64-bit instances):<br />
&emsp;&emsp; · An empty instance uses ~ 3MB of memory.<br />
&emsp;&emsp; · 1 Million small Keys -> String Value pairs use ~ 85MB of memory.<br />
&emsp;&emsp; · 1 Million Keys -> Hash value, representing an object with 5 fields, use ~ 160 MB of memory.<br />
To test your use case is trivial using the `redis-benchmark` utility to generate random data sets and check with the `INFO memory` command the space used.<br />
64-bit systems will use considerably more memory than 32-bit systems to store the same keys, especially if the keys and values are small. This is because pointers take 8 bytes in 64-bit systems. But of course the advantage is that you can have a lot of memory in 64-bit systems, so in order to run large Redis servers a 64-bit system is more or less required. The alternative is sharding.
- > **Is using Redis together with an on-disk database a good idea?**<br />
Yes, a common design pattern involves taking very write-heavy small data in Redis (and data you need the Redis data structures to model your problem in an efficient way), and big blobs of data into an SQL or eventually consistent on-disk database. Similarly sometimes Redis is used in order to take in memory another copy of a subset of the same data stored in the on-disk database. This may look similar to caching, but actually is a more advanced model since normally the Redis dataset is updated together with the on-disk DB dataset, and not refreshed on cache misses.
- > **What happens if Redis runs out of memory?**<br />
Redis will either be killed by the Linux kernel OOM killer, crash with an error, or will start to slow down. With modern operating systems malloc() returning NULL is not common, usually the server will start swapping (if some swap space is configured), and Redis performance will start to degrade, so you'll probably notice there is something wrong.<br />
Redis has built-in protections allowing the user to set a max limit to memory usage, using the maxmemory option in the configuration file to put a limit to the memory Redis can use. If this limit is reached Redis will start to reply with an error to write commands (but will continue to accept read-only commands), or you can configure it to evict keys when the max memory limit is reached in the case you are using Redis for caching.<br />
We have detailed documentation in case you plan to use Redis as an LRU cache.<br />
The INFO command will report the amount of memory Redis is using so you can write scripts that monitor your Redis servers checking for critical conditions before they are reached.<br />
- > **What is the maximum number of keys a single Redis instance can hold? and what the max number of elements in a Hash, List, Set, Sorted Set?**<br />
Redis can handle up to 2<sup>32</sup> keys, and was tested in practice to handle at least 250 million keys per instance.<br />
Every hash, list, set, and sorted set, can hold 2<sup>32</sup> elements.<br />
In other words your limit is likely the available memory in your system.<br />

## **Redis 命令**
### **APPEND**
APPEND key value<br />
O(1)<br />
Pattern: Time series<br />
### **AUTH**
Note: because of the high performance nature of Redis, it is possible to try a lot of passwords in parallel in very short time, so make sure to generate a strong and very long password so that this attack is infeasible.<br />
### **BGREWRITEAOF**
### **BGSAVE**
Save the DB in background. A client may be able to check if the operation succeeded using the `LASTSAVE` command.<br />
### **BITCOUNT**
BITCOUNT key [start end]<br />
O(N)<br />
Pattern: real-time metrics using bitmaps<br />
[REDIS BITMAPS – FAST, EASY, REALTIME METRICS](http://blog.getspool.com/2011/11/29/fast-easy-realtime-metrics-using-redis-bitmaps/) 非常值得作为指标统计方案的参考。<br />
### **BITFIELD**
BITFIELD key [GET type offset] [SET type offset value] [INCRBY type offset increment] [OVERFLOW WRAP|SAT|FAIL]<br />
O(1)<br />
比较复杂琐碎，建议有时间重新研读。<br />
### **BITOP**
BITOP operation destkey key [key ...]<br />
O(N)<br />
The BITOP command supports four bitwise operations: AND, OR, XOR and NOT.<br />
Pattern: real time metrics using bitmaps<br />
BITOP is a potentially slow command as it runs in O(N) time. Care should be taken when running it against long input strings.<br />
For real-time metrics and statistics involving large inputs a good approach is to use a slave (with read-only option disabled) where the bit-wise operations are performed to avoid blocking the master instance.<br />
### **BITPOS**
BITPOS key bit [start][end]<br />
O(N)<br />
The range is interpreted as a range of bytes and not a range of bits, so start=0 and end=2 means to look at the first three bytes.<br />
Note that bit positions are returned always as absolute values starting from bit zero even when start and end are used to specify a range.<br />
### **BLPOP**
BLPOP key [key ...] timeout<br />
O(1)<br />
An element is popped from the head of **the first list that is non-empty**, with the given keys being checked in the order that they are given.<br />
Pattern: Event notification<br />
### **BRPOP**
BRPOP key [key ...] timeout<br />
O(1)<br />
See `BLPOP` for the exact semantics.<br />
### **BRPOPLPUSH**
BRPOPLPUSH source destination timeout<br />
O(1)<br />
See `RPOPLPUSH` for more information.<br />
Pattern: Reliable queue<br />
Pattern: Circular list<br />
### **CLIENT KILL**
CLIENT KILL [ip:port] [ID client-id] [TYPE normal|master|slave|pubsub] [ADDR ip:port] [SKIPME yes/no]<br />
O(N)<br />
### **CLIENT LIST**
CLIENT LIST<br />
O(N)<br />
### **CLIENT GETNAME**
CLIENT GETNAME<br />
O(1)<br />
### **CLIENT PAUSE**
CLIENT PAUSE timeout<br />
O(1)<br />
This command is useful as it makes able to switch clients from a Redis instance to another one in a controlled way. For example during an instance upgrade the system administrator could do the following:<br />
&emsp;&emsp; · Pause the clients using CLIENT PAUSE<br />
&emsp;&emsp; · Wait a few seconds to make sure the slaves processed the latest replication stream from the master.<br />
&emsp;&emsp; · Turn one of the slaves into a master.<br />
&emsp;&emsp; · Reconfigure clients to connect with the new master.<br />
### **CLIENT REPLY**
CLIENT REPLY ON|OFF|SKIP<br />
O(1)<br />
### **CLIENT SETNAME**
CLIENT SETNAME connection-name<br />
O(1)<br />
### **CLUSTER ...**
### **COMMAND**
COMMAND<br />
O(N)<br />
### **COMMAND COUNT**
COMMAND COUNT<br />
O(1)<br />
### **COMMAND GETKEYS**
COMMAND GETKEYS<br />
O(N)<br />
### **COMMAND INFO**
COMMAND INFO command-name [command-name ...]<br />
O(N)<br />
### **CONFIG GET**
CONFIG GET parameter<br />
### **CONFIG REWRITE**
CONFIG REWRITE<br />
### **CONFIG SET**
CONFIG SET parameter value<br />
### **CONFIG RESETSTAT**
CONFIG RESETSTAT<br />
O(1)<br />
### **DBSIZE**
DBSIZE<br />
### **DEBUG OBJECT**
DEBUG OBJECT key<br />
DEBUG OBJECT is a debugging command that should not be used by clients. Check the `OBJECT` command instead.<br />
### **DEBUG SEGFAULT**
DEBUG SEGFAULT<br />
`DEBUG SEGFAULT` performs an invalid memory access that crashes Redis. It is used to simulate bugs during the development.<br />
### **DECR**
DECR key<br />
O(1)<br />
### **DECRBY**
DECRBY key decrement<br />
O(1)<br />
### **DEL**
DEL key [key ...]<br />
O(N)<br />
### **DISCARD**
DISCARD<br />
### **DUMP**
DUMP key<br />
O(1)+O(N*M)
### **ECHO**
ECHO message<br />
### **EVAL**
EVAL script numkeys key [key ...] arg [arg ...]<br />
### **EVALSHA**
EVALSHA sha1 numkeys key [key ...] arg [arg ...]<br />
`SCRIPT LOAD`<br />
### **EXEC**
EXEC<br />
### **EXISTS**
EXISTS key [key ...]<br />
O(1)<br />
### **EXPIRE**
EXPIRE key seconds<br />
O(1)<br />
Pattern: Navigation session
### **EXPIREAT**
EXPIREAT key timestamp<br />
O(1)<br />
### **FLUSHALL**
FLUSHALL [ASYNC]<br />
O(N)<br />
An ASYNC option was added to `FLUSHALL` and `FLUSHDB` of Redis 4.0.0 or greater  in order to let the entire dataset or a single database to be freed asynchronously.<br />
### **FLUSHDB**
FLUSHDB [ASYNC]<br />
O(N)<br />
### **GEOADD**
GEOADD key longitude latitude member [longitude latitude member ...]<br />
O(log(N))<br />
### **GEOHASH**
GEOHASH key member [member ...]<br />
O(log(N))<br />
### **GEODIST**
GEODIST key member1 member2 [unit]<br />
O(log(N))<br />
### **GEORADIUS**
GEORADIUS key longitude latitude radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]<br />
O(N+log(M))<br />
### **GEORADIUSBYMEMBER**
GEORADIUSBYMEMBER key member radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]<br />
O(N+log(M))<br />
### **GET**
GET key<br />
O(1)<br />
`GET` only handles string values.<br />
### **GETBIT**
GETBIT key offset<br />
O(1)<br />
### **GETRANGE**
GETRANGE key start end<br />
O(N)<br />
### **GETSET**
GETSET key value<br />
O(1)<br />
Design Pattern: `GETSET` can be used together with INCR for counting with atomic reset.<br />
### **HDEL**
HDEL key field [field ...]<br />
O(N)<br />
### **HEXISTS**
HEXISTS key field<br />
O(1)<br />
### **HGET**
HGET key field<br />
O(1)<br />
### **HGETALL**
HGETALL key<br />
O(N)<br />
### **HINCRBY**
HINCRBY key field increment<br />
O(1)<br />
### **HINCRBYFLOAT**
HINCRBYFLOAT key field increment<br />
O(1)<br />
### **HKEYS**
HKEYS key<br />
O(N)<br />
### **HLEN**
HLEN key<br />
O(1)<br />
### **HMGET**
HMGET key field [field ...]<br />
O(N)<br />
### **HMSET**
HMSET key field value [field value ...]<br />
O(N)<br />
### **HSET**
HSET key field value<br />
O(1)<br />
### **HSETNX**
HSETNX key field value<br />
O(1)<br />
### **HSTRLEN**
HSTRLEN key field<br />
O(1)<br />
### **HVALS**
HVALS key<br />
O(N)<br />
### **INCR**
INCR key<br />
O(1)<br />
Pattern: Counter<br />
Pattern: **Rate limiter**<br />
### **INCRBY**
ICNRBY key increment<br />
O(1)<br />
### **INCRBYFLOAT**
INCRBYFLOAT key increment<br />
O(1)<br />
### **INFO**
INFO [section]<br />
### **KEYS**
KEYS pattern<br />
O(N) <br />
Warning: consider `KEYS` as a command that should only be used in production environments with extreme care.<br />
### **LASTSAVE**
LASTSAVE<br />
### **LINDEX**
LINDEX key index<br />
O(N)<br />
### **LINSERT**
LINSERT key BEFORE|AFTER pivot value<br />
O(N)<br />
### **LLEN**
LLEN key<br />
O(1)<br />
### **LPOP**
LPOP key<br />
O(1)<br />
### **LPUSH**
LPUSH key value [value ...]<br />
O(1)<br />
### **LPUSHX**
LPUSHX key value [value ...]<br />
O(1)<br />
### **LRANGE**
LRANGE key start stop<br />
O(S+N)<br />
### **LREM**
LREM key count value<br />
O(N)<br />
### **LSET**
LSET key index value<br />
O(N)<br />
### **LTRIM**
LTRIM key start stop<br />
O(N)<br />
### **MGET**
MGET key [key ...]<br />
O(N)<br />
### **MIGRATE**
MIGRATE host port key|"" destination-db timeout [COPY] [REPLACE] [KEYS key [key ...]]<br />
O(N)<br />
### **MONITOR**
MONITOR<br />
### **MOVE**
MOVE key db<br />
O(1)<br />
### **MSET**
MSET key value [key value ...]<br />
O(N)<br />
### **MSETNX**
MSETNX key value [key value ...]<br />
O(N)<br />
### **MULTI**
MULTI<br />
### **OBJECT**
OBJECT subcommand [arguments [arguments ...]]<br />
O(1)<br />
### **PERSIST**
PERSIST key<br />
O(1)<br />
### **PEXPIRE**
PEXPIRE key milliseconds<br />
O(1)<br />
### **PEXPIREAT**
PEXPIREAT key milliseconds-timestamp<br />
O(1)<br />
### **PFADD**
PFADD key element [element ...]<br />
O(1)<br />
### **PFCOUNT**
PFCOUNT key [key ...]<br />
O(N)<br />
### **PFMERGE**
PFMERGE destkey sourcekey [sourcekey ...]<br />
O(N)<br />
### **PING**
PING [message]<br />
### **PSETEX**
PSETEX key milliseconds value<br />
O(1)<br />
### **PSUBSCRIBE**
PSUBSCRIBE pattern [pattern ...]<br />
O(N)<br />
### **PUBSUB**
PUBSUB subcommand [argument [argument ...]]<br />
O(N)<br />
### **PTTL**
PTTL key<br />
O(1)<br />
### **PUBLISH**
PUBLISH channel message
O(N+M)<br />
### **PUNSUBSCRIBE**
PUNSUBSCRIBE [pattern [pattern ...]]<br />
O(N+M)<br />
### **QUIT**
QUIT<br />
### **RANDOMKEY**
RANDOMKEY<br />
O(1)<br />
### **READONLY**
READONLY<br />
O(1)<br />
### **READWRITE**
READWRITE<br />
O(1)<br />
### **RENAME**
RENAME key newkey<br />
O(1)<br />
### **RENAMENX**
RENAMENX key newkey<br />
O(1)<br />
### **RESTORE**
RESTORE key ttl serialized-value [REPLACE]<br />
O(1)+O(N*M)<br />
### **ROLE**
ROLE<br />
### **RPOP**
RPOP key<br />
O(1)<br />
### **RPOPLPUSH**
RPOPLPUSH source destination<br />
O(1)<br />
Pattern: Reliable queue<br />
Pattern: **Circular list**<br />
### **RPUSH**
RPUSH key value [value ...]<br />
O(1)<br />
### **RPUSHX**
RPUSHX key value<br />
O(1)<br />
### **SADD**
SADD key member [member ...]<br />
O(1)<br />
### **SAVE**
SAVE<br />
### **SCARD**
SCARD key<br />
O(1)<br />
### **SCRIPT DEBUG**
SCRIPT DEBUG YES|SYNC|NO<br />
O(1)<br />
[Redis Lua debugger](https://redis.io/topics/ldb)
### **SCRIPT EXISTS**
SCRIPT EXISTS sha1 [sha1 ...]<br />
O(N)<br />
### **SCRIPT FLUSH**
SCRIPT FLUSH<br />
O(N)<br />
### **SCRIPT KILL**
SCRIPT KILL<br />
O(1)<br />
If the script already performed write operations it **can not** be killed in this way because it would violate Lua script atomicity contract. In such a case only `SHUTDOWN NOSAVE` is able to kill the script.<br />
### **SCRIPT LOAD**
SCRIPT LOAD script<br />
O(N)<br />
### **SDIFF**
SDIFF key [key ...]<br />
O(N)<br />
### **SDIFFSTORE**
SDIFFSTORE destination key [key ...]<br />
O(N)<br />
### **SELECT**
SELECT index<br />
When using Redis Cluster, the `SELECT` command **cannot** be used, since Redis Cluster only supports database zero.<br />
### **SET**
SET key value [EX seconds] [PX milliseconds] [NX|XX]<br />
O(1)<br />
Patterns: The command `SET resource-name anystring NX EX max-lock-time` is a simple way to implement a locking system with Redis.<br />
### **SETBIT**
SETBIT key offset value<br />
O(1)<br />
### **SETEX**
SETEX key seconds value<br />
O(1)<br />
### **SETNX**
SETNX key value<br />
O(1)<br />
Design pattern: Locking with `SETNX`<br />
### **SETRANGE**
SETRANGE key offset value<br />
O(1)<br />
Patterns: Use redis as a linear array.<br />
### **SHUTDOWN**
SHUTDOWN [NOSAVE|SAVE]<br />
`SHUTDOWN SAVE` will force a DB saving operation **even if** no save points are configured.<br />
There are conditions when we want just to terminate a Redis instance **ASAP**, regardless of what its content is. In such a case, the right combination of commands is to send a `CONFIG appendonly no` followed by a `SHUTDOWN NOSAVE`.<br />
### **SINTER**
SINTER key [key ...]<br />
O(N*M)<br />
### **SINTERSTORE**
SINTERSTORE destination key [key ...]<br />
O(N*M)<br />
### **SISMEMBER**
SISMEMBER key member<br />
O(1)<br />
### **SLAVEOF**
SLAVEOF host port<br />
### **SLOWLOG**
SLOWLOG subcommand [argument]<br />
Two parameters: `slowlog-log-slower-than`, `slowlog-max-len`<br />
subcommand: GET LEN RESET<br />
### **SMEMBERS**
SMEMBERS key<br />
O(N)<br />
### **SMOVE**
SMOVE source destination member<br />
O(1)<br />
### **SORT**
SORT key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC|DESC] [ALPHA] [STORE destination]<br />
O(N+M*log(M))<br />
An interesting pattern using `SORT` ... `STORE` consists in associating an `EXPIRE` timeout to the resulting key so that in applications where the result of a `SORT` operation can be cached for some time. Other clients will use the cached list instead of calling `SORT` for every request. When the key will timeout, an updated version of the cache can be created by calling `SORT` ... `STORE` again.<br />
It is possible to use BY and GET options against hash fields with the following syntax:<br />
`SORT mylist BY weight_*->fieldname GET object_*->fieldname`<br />
### **SPOP**
SPOP key [count]<br />
O(1)<br />
### **SRANDMEMBER**
SRANDMEMBER key [count]<br />
O(1)/O(N)<br />
### **SREM**
SREM key member [member ...]<br />
O(N)<br />
### **STRLEN**
STRLEN key<br />
O(1)<br />
### **SUBSCRIBE**
SUBSCRIBE channel [channel ...]<br />
O(N)<br />
### **SUNION**
SUNION key [key ...]<br />
O(N)<br />
### **SUNIONSTORE**
SUNIONSTORE destination key [key ...]<br />
O(N)<br />
### **SWAPDB**
SWAPDB index index<br />
### **SYNC**
SYNC<br />
### **TIME**
TIME<br />
O(1)<br />
### **TOUCH**
TOUCH key [key ...]<br />
O(N)<br />
### **TTL**
TTL key<br />
O(1)<br />
### **TYPE**
TYPE key<br />
O(1)<br />
### **UNSUBSCRIBE**
UNSUBSCRIBE [channel [channel ...]]<br />
O(N)<br />
### **UNLINK**
UNLINK key [key ...]<br />
O(1)/O(N)<br />
### **UNWATCH**
UNWATCH<br />
O(1)<br />
### **WAIT**
WAIT numslaves timeout<br />
O(1)<br />
### **WATCH**
WATCH key [key ...]<br />
O(1)<br />
### **ZADD**
ZADD key [NX|XX] [CH] [INCR] score member [score member ...]<br />
O(log(N))<br />
### **ZCARD**
ZCARD key<br />
O(1)<br />
### **ZCOUNT**
ZCOUNT key min max<br />
O(log(N))<br />
### **ZINCRBY**
ZINCRBY key increment member<br />
O(log(N))<br />
### **ZINTERSTORE**
ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]<br />
O(N*K)+O(M*log(M))<br />
### **ZLEXCOUNT**
ZLEXCOUNT key min max<br />
O(log(N))<br />
### **ZRANGE**
ZRANGE key start stop [WITHSCORES]<br />
O(log(N)+M)<br />
### **ZRANGEBYLEX**
ZRANGEBYLEX key min max [LIMIT offset count]<br />
O(log(N)+M)<br />
### **ZREVRANGEBYLEX**
ZREVRANGEBYLEX key max min [LIMIT offset count]<br />
O(log(N)+M)<br />
### **ZRANGEBYSCORE**
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]<br />
O(log(N)+M)<br />
Pattern: weighted random selection of an element<br />
### **ZRANK**
ZRANK key member<br />
O(log(N))<br />
### **ZREM**
ZREM key member [member ...]<br />
O(M*log(N))<br />
### **ZREMRANGEBYLEX**
ZREMRANGEBYLEX key min max<br />
O(log(N)+M)<br />
### **ZREMRANGEBYRANK**
ZREMRANGEBYRANK key start stop<br />
O(log(N)+M)<br />
### **ZREVRANK**
ZREVRANK key member<br />
O(log(N))<br />
### **ZSCORE**
ZSCORE key member<br />
O(1)<br />
### **ZUNIONSTORE**
ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]<br />
O(N)+O(M log(M))<br />
### **SCAN**
SCAN cursor [MATCH pattern] [COUNT count]<br />
O(1)/O(N)<br />
### **SSCAN**
SSCAN key cursor [MATCH pattern] [COUNT count]<br />
O(1)/O(N)<br />
### **HSCAN**
HSCAN key cursor [MATCH pattern] [COUNT count]<br />
O(1)/O(N)<br />
### **ZSCAN**
ZSCAN key cursor [MATCH pattern] [COUNT count]<br />
O(1)/O(N)<br />




## **Redis 分区**
以下内容摘自 [Redis Partitioning](https://redis.io/topics/partitioning)<br />
- > **Why partitioning is useful**<br />
Partitioning in Redis serves two main goals:<br />
&emsp;&emsp; · It allows for much larger databases, using the sum of the memory of many computers. Without partitioning you are limited to the amount of memory a single computer can support.<br />
&emsp;&emsp; · It allows scaling the computational power to multiple cores and multiple computers, and the network bandwidth to multiple computers and network adapters.
- > **Partitioning basics**<br />
There are different partitioning criteria.<br />
One of the simplest ways to perform partitioning is with range partitioning, and is accomplished by mapping ranges of objects into specific Redis instances.<br />
This system works and is actually used in practice, however, it has the disadvantage of requiring a table that maps ranges to instances. This table needs to be managed and a table is needed for every kind of object, so therefore range partitioning in Redis is often undesirable because it is much more inefficient than other alternative partitioning approaches.<br />
An alternative to range partitioning is hash partitioning. This scheme works with any key, without requiring a key in the form object_name:\<id\>, and is as simple as:<br />
&emsp;&emsp; · Take the key name and use a hash function (e.g., the crc32 hash function) to turn it into a number.<br />
&emsp;&emsp; · Use a modulo operation with this number in order to turn it into a number between 0 and 3, so that this number can be mapped to one of my four Redis instances.<br />
One advanced form of hash partitioning is called consistent hashing and is implemented by a few Redis clients and proxies.
- > **Different implementations of partitioning**<br />
Partitioning can be the responsibility of different parts of a software stack.<br />
&emsp;&emsp; · **Client side partitioning** means that the clients directly select the right node where to write or read a given key. Many Redis clients implement client side partitioning.<br />
&emsp;&emsp; · **Proxy assisted partitioning** means that our clients send requests to a proxy that is able to speak the Redis protocol, instead of sending requests directly to the right Redis instance. The proxy will make sure to forward our request to the right Redis instance accordingly to the configured partitioning schema, and will send the replies back to the client. The Redis and Memcached proxy Twemproxy implements proxy assisted partitioning.<br />
&emsp;&emsp; · **Query routing** means that you can send your query to a random instance, and the instance will make sure to forward your query to the right node. Redis Cluster implements an hybrid form of query routing, with the help of the client (the request is not directly forwarded from a Redis instance to another, but the client gets redirected to the right node).
- > **Disadvantages of partitioning**<br />
Some features of Redis don't play very well with partitioning:<br />
&emsp;&emsp; · Operations involving multiple keys are usually not supported. For instance you can't perform the intersection between two sets if they are stored in keys that are mapped to different Redis instances (actually there are ways to do this, but not directly).<br />
&emsp;&emsp; · Redis transactions involving multiple keys can not be used.<br />
&emsp;&emsp; · The partitioning granularity is the key, so it is not possible to shard a dataset with a single huge key like a very big sorted set.<br />
&emsp;&emsp; · When partitioning is used, data handling is more complex, for instance you have to handle multiple RDB / AOF files, and to make a backup of your data you need to aggregate the persistence files from multiple instances and hosts.<br />
&emsp;&emsp; · Adding and removing capacity can be complex. For instance Redis Cluster supports mostly transparent rebalancing of data with the ability to add and remove nodes at runtime, but other systems like client side partitioning and proxies don't support this feature. However a technique called Pre-sharding helps in this regard.

## **Redis 内存优化**
以下内容摘自 [Redis Memory Optimization page](https://redis.io/topics/memory-optimization)<br />
- > **Special encoding of small aggregate data types**<br />
Since Redis 2.2 many data types are optimized to use less space up to a certain size. Hashes, Lists, Sets composed of just integers, and Sorted Sets, when smaller than a given number of elements, and up to a maximum element size, are encoded in a very memory efficient way that uses up to 10 times less memory (with 5 time less memory used being the average saving).<br />
This is completely transparent from the point of view of the user and API. Since this is a CPU / memory trade off it is possible to tune the maximum number of elements and maximum element size for special encoded types using the following redis.conf directives.<br />
&emsp;&emsp; hash-max-zipmap-entries 512 (hash-max-ziplist-entries for Redis >= 2.6)<br />
&emsp;&emsp; hash-max-zipmap-value 64  (hash-max-ziplist-value for Redis >= 2.6)<br />
&emsp;&emsp; list-max-ziplist-entries 512<br />
&emsp;&emsp; list-max-ziplist-value 64<br />
&emsp;&emsp; zset-max-ziplist-entries 128<br />
&emsp;&emsp; zset-max-ziplist-value 64<br />
&emsp;&emsp; set-max-intset-entries 512<br />
If a specially encoded value will overflow the configured max size, Redis will automatically convert it into normal encoding. This operation is very fast for small values, but if you change the setting in order to use specially encoded values for much larger aggregate types the suggestion is to run some benchmark and test to check the conversion time.<br />

**ALEX ZHOU 注：**<br />
有关 "Special encoding of small aggregate data types" 的部分建议参考 [这篇文章](https://redis4you.com/articles.php?id=008&name=Understanding+hash-max-zipmap-entries+and+)<br />
&emsp;&emsp; CPU / memory trade off - When using this method, Redis may do huge savings of memory, however when access hash elements, it will need to iterate all hash field, e.g. Redis trades CPU for memory.<br />
&emsp;&emsp; hash_max_zipmap_entries - if hash fields are less this value the optimized method is used.<br />
&emsp;&emsp; hash_max_zipmap_value - if hash values sizes are less this value the optimized method is used.
- > **Using 32 bit instances**<br />
Redis compiled with 32 bit target uses a lot less memory per key, since pointers are small, but such an instance will be limited to 4 GB of maximum memory usage. To compile Redis as 32 bit binary use make 32bit. RDB and AOF files are compatible between 32 bit and 64 bit instances (and between little and big endian of course) so you can switch from 32 to 64 bit, or the contrary, without problems.
- > **Bit and byte level operations**<br />
Redis 2.2 introduced new bit and byte level operations: GETRANGE, SETRANGE, GETBIT and SETBIT. Using these commands you can treat the Redis string type as a random access array. For instance if you have an application where users are identified by a unique progressive integer number, you can use a bitmap in order to save information about the sex of users, setting the bit for females and clearing it for males, or the other way around. With 100 million users this data will take just 12 megabytes of RAM in a Redis instance. You can do the same using GETRANGE and SETRANGE in order to store one byte of information for each user. This is just an example but it is actually possible to model a number of problems in very little space with these new primitives.

**ALEX ZHOU 注：**<br />
有关 "Bit and byte level operations" 请参考 [这篇文章](https://stackoverflow.com/questions/20673131/can-someone-explain-redis-setbit-command)<br />
0x7f 解析如下<br />
位置：0 1 2 3 4 5 6 7<br />
取位：0 1 1 1 1 1 1 1<br />
&emsp;&emsp;&emsp;MSB&emsp;&emsp;&emsp;&emsp;LSB
- > **Use hashes when possible**<br />
Small hashes are encoded in a very small space, so you should try representing your data using hashes every time it is possible. For instance if you have objects representing users in a web application, instead of using different keys for name, surname, email, password, use a single hash with all the required fields.

## **Redis 缓存**
具体内容摘自 [Redis as an LRU cache](https://redis.io/topics/lru-cache)<br />
> When Redis is used as a cache, often it is handy to let it automatically evict old data as you add new one. This behavior is very well known in the community of developers, since it is the default behavior of the popular memcached system.<br />
LRU is actually only one of the supported eviction methods. This page covers the more general topic of the Redis maxmemory directive that is used in order to limit the memory usage to a fixed amount, and it also covers in depth the LRU algorithm used by Redis, that is actually an approximation of the exact LRU.<br />
Starting with Redis version 4.0, a new LFU (Least Frequently Used) eviction policy was introduced. This is covered in a separated section of this documentation.<br />
- > **Maxmemory configuration directive**<br />
The maxmemory configuration directive is used in order to configure Redis to use a specified amount of memory for the data set. It is possible to set the configuration directive using the redis.conf file, or later using the `CONFIG SET` command at runtime.<br />
Setting maxmemory to zero results into no memory limits. This is the default behavior for 64 bit systems, while 32 bit systems use an implicit memory limit of 3GB.<br />
When the specified amount of memory is reached, it is possible to select among different behaviors, called policies. Redis can just return errors for commands that could result in more memory being used, or it can evict some old data in order to return back to the specified limit every time new data is added.
- > **Eviction policies**<br />
The exact behavior Redis follows when the maxmemory limit is reached is configured using the maxmemory-policy configuration directive.<br />
The following policies are available:<br />
&emsp;&emsp; **· noeviction**: return errors when the memory limit was reached and the client is trying to execute commands that could result in more memory to be used (most write commands, but DEL and a few more exceptions).<br />
&emsp;&emsp; **· allkeys-lru**: evict keys by trying to remove the less recently used (LRU) keys first, in order to make space for the new data added.<br />
&emsp;&emsp; **· volatile-lru**: evict keys by trying to remove the less recently used (LRU) keys first, but only among keys that have an expire set, in order to make space for the new data added.<br />
&emsp;&emsp; **· allkeys-random**: evict keys randomly in order to make space for the new data added.<br />
&emsp;&emsp; **· volatile-random**: evict keys randomly in order to make space for the new data added, but only evict keys with an expire set.<br />
&emsp;&emsp; **· volatile-ttl**: evict keys with an expire set, and try to evict keys with a shorter time to live (TTL) first, in order to make space for the new data added.<br />
The policies volatile-lru, volatile-random and volatile-ttl behave like noeviction if there are no keys to evict matching the prerequisites.<br />
To pick the right eviction policy is important depending on the access pattern of your application, however you can reconfigure the policy at runtime while the application is running, and monitor the number of cache misses and hits using the Redis `INFO` output in order to tune your setup.<br />
In general as a rule of thumb:<br />
&emsp;&emsp; · Use the allkeys-lru policy when you expect a power-law distribution in the popularity of your requests, that is, you expect that a subset of elements will be accessed far more often than the rest. This is a good pick if you are unsure.<br />
&emsp;&emsp; · Use the allkeys-random if you have a cyclic access where all the keys are scanned continuously, or when you expect the distribution to be uniform (all elements likely accessed with the same probability).<br />
&emsp;&emsp; · Use the volatile-ttl if you want to be able to provide hints to Redis about what are good candidate for expiration by using different TTL values when you create your cache objects.<br />
The volatile-lru and volatile-random policies are mainly useful when you want to use a single instance for both caching and to have a set of persistent keys. However it is usually a better idea to run two Redis instances to solve such a problem.<br />
- > **How the eviction process works**<br />
It is important to understand that the eviction process works like this:<br />
&emsp;&emsp; · A client runs a new command, resulting in more data added.<br />
&emsp;&emsp; · Redis checks the memory usage, and if it is greater than the maxmemory limit, it evicts keys according to the policy.<br />
&emsp;&emsp; · A new command is executed, and so forth.<br />
- > **Approximated LRU algorithm**<br />
Redis LRU algorithm is not an exact implementation. This means that Redis is not able to pick the best candidate for eviction, that is, the access that was accessed the most in the past. Instead it will try to run an approximation of the LRU algorithm, by sampling a small number of keys, and evicting the one that is the best (with the oldest access time) among the sampled keys.<br />
What is important about the Redis LRU algorithm is that you are able to tune the precision of the algorithm by changing the number of samples to check for every eviction. This parameter is controlled by the following configuration directive:<br />
`maxmemory-samples 5`<br />
- > **The new LFU mode**<br />
Starting with Redis 4.0, a new Least Frequently Used eviction mode is available. This mode may work better (provide a better hits/misses ratio) in certain cases, since using LFU Redis will try to track the frequency of access of items, so that the ones used rarely are evicted while the one used often have an higher chance of remaining in memory.<br />
If you think at LRU, an item that was recently accessed but is actually almost never requested, will not get expired, so the risk is to evict a key that has an higher chance to be requested in the future. LFU does not have this problem, and in general should adapt better to different access patterns.<br />
To configure the LFU mode, the following policies are available:<br />
&emsp;&emsp; **· volatile-lfu** Evict using approximated LFU among the keys with an expire set.<br />
&emsp;&emsp; **· allkeys-lfu** Evict any key using approximated LFU.<br />
By default Redis 4.0 is configured to:<br />
&emsp;&emsp; · Saturate the counter at, around, one million requests.<br />
&emsp;&emsp; · Decay the counter every one minute.<br />
Instructions about how to tune these parameters can be found inside the example redis.conf file in the source distribution, but briefly, they are:<br />
`lfu-log-factor 10`<br />
`lfu-decay-time 1`<br />

## **Redis 通讯协议**
以下内容摘自 [Redis Protocol specification](https://redis.io/topics/protocol)<br />
Redis clients communicate with the Redis server using a protocol called RESP (REdis Serialization Protocol). While the protocol was designed specifically for Redis, it can be used for other client-server software projects.<br />
RESP can serialize different data types like integers, strings, arrays. There is also a specific type for errors. Requests are sent from the client to the Redis server as arrays of strings representing the arguments of the command to execute. Redis replies with a command-specific data type.<br />
RESP is binary-safe and does not require processing of bulk data transferred from one process to another, because it uses prefixed-length to transfer bulk data.<br />
Note: the protocol outlined here is only used for client-server communication. Redis Cluster uses a different binary protocol in order to exchange messages between nodes.<br />
- > **Networking layer**<br />
A client connects to a Redis server creating a TCP connection to the port 6379.<br />
While RESP is technically non-TCP specific, in the context of Redis the protocol is only used with TCP connections (or equivalent stream oriented connections like Unix sockets).<br />
- > **Request-Response model**<br />
Redis accepts commands composed of different arguments. Once a command is received, it is processed and a reply is sent back to the client.<br />
This is the simplest model possible, however there are two exceptions:<br />
&emsp;&emsp; · Redis supports pipelining (covered later in this document). So it is possible for clients to send multiple commands at once, and wait for replies later.<br />
&emsp;&emsp; · When a Redis client subscribes to a Pub/Sub channel, the protocol changes semantics and becomes a push protocol, that is, the client no longer requires to send commands, because the server will automatically send to the client new messages (for the channels the client is subscribed to) as soon as they are received.<br />
Excluding the above two exceptions, the Redis protocol is a simple request-response protocol.<br />
- > **RESP protocol description**<br />
The RESP protocol was introduced in Redis 1.2, but it became the standard way for talking with the Redis server in Redis 2.0. This is the protocol you should implement in your Redis client.<br />
RESP is actually a serialization protocol that supports the following data types: Simple Strings, Errors, Integers, Bulk Strings and Arrays.<br />
The way RESP is used in Redis as a request-response protocol is the following:<br />
&emsp;&emsp; · Clients send commands to a Redis server as a RESP Array of Bulk Strings.<br />
&emsp;&emsp; · The server replies with one of the RESP types according to the command implementation.<br />
In RESP, the type of some data depends on the first byte:<br />
&emsp;&emsp; · For Simple Strings the first byte of the reply is "+"<br />
&emsp;&emsp; · For Errors the first byte of the reply is "-"<br />
&emsp;&emsp; · For Integers the first byte of the reply is ":"<br />
&emsp;&emsp; · For Bulk Strings the first byte of the reply is "$"<br />
&emsp;&emsp; · For Arrays the first byte of the reply is "*"<br />
Additionally RESP is able to represent a Null value using a special variation of Bulk Strings or Array as specified later.<br />
In RESP different parts of the protocol are always terminated with "\r\n" (CRLF).<br />
- > **RESP Simple Strings**<br />
Simple Strings are encoded in the following way: a plus character, followed by a string that cannot contain a CR or LF character (no newlines are allowed), terminated by CRLF (that is "\r\n").<br />
Simple Strings are used to transmit non binary safe strings with minimal overhead. For example many Redis commands reply with just "OK" on success, that as a RESP Simple String is encoded with the following 5 bytes:<br />
`"+OK\r\n"`<br />
In order to send binary-safe strings, RESP Bulk Strings are used instead.<br />
When Redis replies with a Simple String, a client library should return to the caller a string composed of the first character after the '+' up to the end of the string, excluding the final CRLF bytes.<br />
- > **RESP Errors**<br />
RESP has a specific data type for errors. Actually errors are exactly like RESP Simple Strings, but the first character is a minus '-' character instead of a plus. The real difference between Simple Strings and Errors in RESP is that errors are treated by clients as exceptions, and the string that composes the Error type is the error message itself.<br />
The basic format is:<br />
`"-Error message\r\n"`<br />
Error replies are only sent when something wrong happens, for instance if you try to perform an operation against the wrong data type, or if the command does not exist and so forth. An exception should be raised by the library client when an Error Reply is received.<br />
The following are examples of error replies:<br />
`-ERR unknown command 'foobar'`<br />
`-WRONGTYPE Operation against a key holding the wrong kind of value`<br />
The first word after the "-", up to the first space or newline, represents the kind of error returned. This is just a convention used by Redis and is not part of the RESP Error format.<br />
A client implementation may return different kind of exceptions for different errors, or may provide a generic way to trap errors by directly providing the error name to the caller as a string.<br />
However, such a feature should not be considered vital as it is rarely useful, and a limited client implementation may simply return a generic error condition, such as false.<br />
- > **RESP Integers**<br />
This type is just a CRLF terminated string representing an integer, prefixed by a ":" byte. For example ":0\r\n", or ":1000\r\n" are integer replies.<br />
Many Redis commands return RESP Integers, like `INCR`, `LLEN` and `LASTSAVE`.<br />
There is no special meaning for the returned integer, it is just an incremental number for INCR, a UNIX time for LASTSAVE and so forth. However, the returned integer is guaranteed to be in the range of a signed 64 bit integer.<br />
Integer replies are also extensively used in order to return true or false. For instance commands like `EXISTS` or `SISMEMBER` will return 1 for true and 0 for false.<br />
Other commands like `SADD`, `SREM` and `SETNX` will return 1 if the operation was actually performed, 0 otherwise.<br />
The following commands will reply with an integer reply: `SETNX`, `DEL`, `EXISTS`, `INCR`, `INCRBY`, `DECR`, `DECRBY`, `DBSIZE`, `LASTSAVE`, `RENAMENX`, `MOVE`, `LLEN`, `SADD`, `SREM`, `SISMEMBER`, `SCARD`.<br />
- > **RESP Bulk Strings**<br />
Bulk Strings are used in order to represent a single binary safe string up to 512 MB in length.<br />
Bulk Strings are encoded in the following way:<br />
&emsp;&emsp; · A "$" byte followed by the number of bytes composing the string (a prefixed length), terminated by CRLF.<br />
&emsp;&emsp; · The actual string data.<br />
&emsp;&emsp; · A final CRLF.<br />
So the string "foobar" is encoded as follows:<br />
`"$6\r\nfoobar\r\n"`<br />
When an empty string is just:<br />
`"$0\r\n\r\n"`<br />
RESP Bulk Strings can also be used in order to signal non-existence of a value using a special format that is used to represent a `Null` value. In this special format the length is -1, and there is no data, so a `Null` is represented as:<br />
`"$-1\r\n"`<br />
This is called a Null Bulk String.<br />
The client library API should not return an empty string, but a nil object, when the server replies with a Null Bulk String. For example a Ruby library should return `'nil'` while a C library should return `NULL` (or set a special flag in the reply object), and so forth.<br />
- > **RESP Arrays**<br />
Clients send commands to the Redis server using RESP Arrays. Similarly certain Redis commands returning collections of elements to the client use RESP Arrays are reply type. An example is the LRANGE command that returns elements of a list.<br />
RESP Arrays are sent using the following format:<br />
&emsp;&emsp; · A * character as the first byte, followed by the number of elements in the array as a decimal number, followed by CRLF.<br />
&emsp;&emsp; · An additional RESP type for every element of the Array.<br />
So an empty Array is just the following:<br />
`"*0\r\n"`<br />
While an array of two RESP Bulk Strings "foo" and "bar" is encoded as:<br />
`"*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n"`<br />
Arrays can contain mixed types, it's not necessary for the elements to be of the same type. <br />
The concept of Null Array exists as well, and is an alternative way to specify a Null value (usually the Null Bulk String is used, but for historical reasons we have two formats).<br />
`"*-1\r\n"`<br />
A client library API should return a null object and not an empty Array when Redis replies with a Null Array. This is necessary to distinguish between an empty list and a different condition (for instance the timeout condition of the BLPOP command).<br />
- > **Null elements in Arrays**<br />
Single elements of an Array may be Null. This is used in Redis replies in order to signal that this elements are missing and not empty strings. This can happen with the SORT command when used with the GET pattern option when the specified key is missing. <br />
- > **Sending commands to a Redis Server**<br />
Now that you are familiar with the RESP serialization format, writing an implementation of a Redis client library will be easy. We can further specify how the interaction between the client and the server works:<br />
&emsp;&emsp; · A client sends to the Redis server a RESP Array consisting of just Bulk Strings.<br />
&emsp;&emsp; · A Redis server replies to clients sending any valid RESP data type as reply.<br />
- > **Multiple commands and pipelining**<br />
A client can use the same connection in order to issue multiple commands. Pipelining is supported so multiple commands can be sent with a single write operation by the client, without the need to read the server reply of the previous command before issuing the next one. All the replies can be read at the end.<br />
For more information please check our [page about Pipelining](https://redis.io/topics/pipelining).<br />
- > **Inline Commands**<br />
Sometimes you have only telnet in your hands and you need to send a command to the Redis server. While the Redis protocol is simple to implement it is not ideal to use in interactive sessions, and redis-cli may not always be available. For this reason Redis also accepts commands in a special way that is designed for humans, and is called the inline command format.<br />
The following is an example of a server/client chat using an inline command (the server chat starts with S:, the client chat with C:)<br />
`C: PING`<br />
`S: +PONG`<br />
The following is another example of an inline command returning an integer:<br />
`C: EXISTS somekey`<br />
`S: :0`<br />
Basically you simply write space-separated arguments in a telnet session. Since no command starts with * that is instead used in the unified request protocol, Redis is able to detect this condition and parse your command.<br />
- > **High performance parser for the Redis protocol**<br />
While the Redis protocol is very human readable and easy to implement it can be implemented with a performance similar to that of a binary protocol.<br />
RESP uses prefixed lengths to transfer bulk data, so there is never need to scan the payload for special characters like it happens for instance with JSON, nor to quote the payload that needs to be sent to the server.<br />
The Bulk and Multi Bulk lengths can be processed with code that performs a single operation per character while at the same time scanning for the CR character, like the following C code:<br />
After the first CR is identified, it can be skipped along with the following LF without any processing. Then the bulk data can be read using a single read operation that does not inspect the payload in any way. Finally the remaining the CR and LF character are discarded without any processing.<br />
``` C
#include <stdio.h>

int main(void) {
    unsigned char *p = "$123\r\n";
    int len = 0;

    p++;
    while(*p != '\r') {
        len = (len*10)+(*p - '0');
        p++;
    }

    /* Now p points at '\r', and the len is in bulk_len. */
    printf("%d\n", len);
    return 0;
}
```

## **Redis Pipeline**
以下内容摘自 [Using pipelining to speedup Redis queries](https://redis.io/topics/pipelining)<br />
- > **Request/Response protocols and RTT**<br />
Redis is a TCP server using the client-server model and what is called a Request/Response protocol.<br />
This means that usually a request is accomplished with the following steps:<br />
&emsp;&emsp; · The client sends a query to the server, and reads from the socket, usually in a blocking way, for the server response.<br />
&emsp;&emsp; · The server processes the command and sends the response back to the client.<br />
So for instance a four commands sequence is something like this:<br />
`Client: INCR X`<br />
`Server: 1`<br />
`Client: INCR X`<br />
`Server: 2`<br />
`Client: INCR X`<br />
`Server: 3`<br />
`Client: INCR X`<br />
`Server: 4`<br />
Clients and Servers are connected via a networking link. Such a link can be very fast (a loopback interface) or very slow (a connection established over the Internet with many hops between the two hosts). Whatever the network latency is, there is a time for the packets to travel from the client to the server, and back from the server to the client to carry the reply.<br />
This time is called RTT (Round Trip Time). It is very easy to see how this can affect the performances when a client needs to perform many requests in a row (for instance adding many elements to the same list, or populating a database with many keys). For instance if the RTT time is 250 milliseconds (in the case of a very slow link over the Internet), even if the server is able to process 100k requests per second, we'll be able to process at max four requests per second.<br />
If the interface used is a loopback interface, the RTT is much shorter (for instance my host reports 0,044 milliseconds pinging 127.0.0.1), but it is still a lot if you need to perform many writes in a row.<br />
Fortunately there is a way to improve this use case.<br />
- > **Redis Pipelining**<br />
A Request/Response server can be implemented so that it is able to process new requests even if the client didn't already read the old responses. This way it is possible to send multiple commands to the server without waiting for the replies at all, and finally read the replies in a single step.<br />
This is called pipelining, and is a technique widely in use since many decades.<br />
**IMPORTANT NOTE**: While the client sends commands using pipelining, the server will be forced to queue the replies, using memory. So if you need to send a lot of commands with pipelining, it is better to send them as batches having a reasonable number, for instance 10k commands, read the replies, and then send another 10k commands again, and so forth. The speed will be nearly the same, but the additional memory used will be at max the amount needed to queue the replies for this 10k commands.<br />
- > **It's not just a matter of RTT**<br />
Pipelining is not just a way in order to reduce the latency cost due to the round trip time, it actually improves by a huge amount the total operations you can perform per second in a given Redis server. This is the result of the fact that, without using pipelining, serving each command is very cheap from the point of view of accessing the data structures and producing the reply, but it is very costly from the point of view of doing the socket I/O. This involves calling the read() and write() syscall, that means going from user land to kernel land. The context switch is a huge speed penalty.<br />
When pipelining is used, many commands are usually read with a single read() system call, and multiple replies are delivered with a single write() system call. Because of this, the number of total queries performed per second initially increases almost linearly with longer pipelines, and eventually reaches 10 times the baseline obtained not using pipelining, as you can see from the following graph:<br />
![](images/pipeline_iops.png)
- > **Some real world code example**<br />
In the following benchmark we'll use the Redis Ruby client, supporting pipelining, to test the speed improvement due to pipelining:<br />
``` Ruby
require 'rubygems'
require 'redis'

def bench(descr)
    start = Time.now
    yield
    puts "#{descr} #{Time.now-start} seconds"
end

def without_pipelining
    r = Redis.new
    10000.times {
        r.ping
    }
end

def with_pipelining
    r = Redis.new
    r.pipelined {
        10000.times {
            r.ping
        }
    }
end

bench("without pipelining") {
    without_pipelining
}
bench("with pipelining") {
    with_pipelining
}
```
ALEX ZHOU 注：<br />
centos7 环境需安装版本 >2.2.2 的 ruby<br />
`sudo yum -y install zlib-devel`<br />
`sudo yum -y install openssl`<br />
`sudo yum -y install openssl-devel`<br />
`wget https://cache.ruby-lang.org/pub/ruby/2.5/ruby-2.5.0.tar.gz`<br />
`tar zxf ruby-2.5.0.tar.gz`<br />
`cd ruby-2.5.0`<br />
`./configure`<br />
`make`<br />
`sudo make install`<br />
ruby 需安装 redis 支持<br />
`sudo /usr/local/bin/gem install redis`<br />
> Running the above simple script will provide the following figures in my Mac OS X system, running over the loopback interface, where pipelining will provide the smallest improvement as the RTT is already pretty low:<br />
`without pipelining 1.185238 seconds`<br />
`with pipelining 0.250783 seconds`<br />
As you can see, using pipelining, we improved the transfer by a factor of five.<br />
- > **Pipelining VS Scripting**<br />
Using [Redis scripting](https://redis.io/commands/eval) (available in Redis version 2.6 or greater) a number of use cases for pipelining can be addressed more efficiently using scripts that perform a lot of the work needed at the server side. A big advantage of scripting is that it is able to both read and write data with minimal latency, making operations like read, compute, write very fast (pipelining can't help in this scenario since the client needs the reply of the read command before it can call the write command).<br />
Sometimes the application may also want to send EVAL or EVALSHA commands in a pipeline. This is entirely possible and Redis explicitly supports it with the SCRIPT LOAD command (it guarantees that EVALSHA can be called without the risk of failing).<br />
- > **Appendix: why a busy loops are slow even on the loopback interface?**<br />
Even with all the background covered in this page, you may still wonder why a Redis benchmark like the following (in pseudo code), is slow even when executed in the loopback interface, when the server and the client are running in the same physical machine:<br />
`FOR-ONE-SECOND:`<br />
&emsp;&emsp; `Redis.SET("foo","bar")`<br />
`END`<br />
After all if both the Redis process and the benchmark are running in the same box, isn't this just messages copied via memory from one place to another without any actual latency and actual networking involved?<br />
The reason is that processes in a system are not always running, actually it is the kernel scheduler that let the process run, so what happens is that, for instance, the benchmark is allowed to run, reads the reply from the Redis server (related to the last command executed), and writes a new command. The command is now in the loopback interface buffer, but in order to be read by the server, the kernel should schedule the server process (currently blocked in a system call) to run, and so forth. So in practical terms the loopback interface still involves network-alike latency, because of how the kernel scheduler works.<br />
Basically a busy loop benchmark is the silliest thing that can be done when metering performances in a networked server. The wise thing is just avoiding benchmarking in this way.<br />

## **Redis 服务端脚本**
以下内容摘自[EVAL script numkeys key [key ...] arg [arg ...]](https://redis.io/commands/eval)<br />
- > **Introduction to EVAL**<br />
`EVAL` and `EVALSHA` are used to evaluate scripts using the Lua interpreter built into Redis starting from version 2.6.0.<br />
The first argument of `EVAL` is a Lua 5.1 script. The script does not need to define a Lua function (and should not). It is just a Lua program that will run in the context of the Redis server.<br />
The second argument of `EVAL` is the number of arguments that follows the script (starting from the third argument) that represent Redis key names. The arguments can be accessed by Lua using the `KEYS` global variable in the form of a one-based array (so KEYS[1], KEYS[2], ...).<br />
All the additional arguments should not represent key names and can be accessed by Lua using the ARGV global variable, very similarly to what happens with keys (so ARGV[1], ARGV[2], ...).<br />
`> eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2]}" 2 key1 key2 first second`<br />
`1) "key1"`<br />
`2) "key2"`<br />
`3) "first"`<br />
`4) "second"`<br />
Note: as you can see Lua arrays are returned as Redis multi bulk replies, that is a Redis return type that your client library will likely convert into an Array type in your programming language.<br />
It is possible to call Redis commands from a Lua script using two different Lua functions:<br />
&emsp;&emsp; · `redis.call()`<br />
&emsp;&emsp; · `redis.pcall()`<br />
`redis.call()` is similar to `redis.pcall()`, the only difference is that if a Redis command call will result in an error, `redis.call()` will raise a Lua error that in turn will force `EVAL` to return an error to the command caller, while `redis.pcall` will trap the error and return a Lua table representing the error.<br />
The arguments of the redis.call() and redis.pcall() functions are all the arguments of a well formed Redis command:<br />
`> eval "return redis.call('set','foo','bar')" 0`<br />
`OK`<br />
`> eval "return redis.call('incr', 'foo')" 0`<br />
`(error) ERR Error running script (call to f_74e373dc4f7de1bf8574bfa77b3873b474d0c056): @user_script:1: ERR value is not an integer or out of range`<br />
`> eval "return redis.pcall('incr', 'foo')" 0`<br />
`(error) ERR value is not an integer or out of range`<br />
The above script sets the key foo to the string bar. However it violates the EVAL command semantics as all the keys that the script uses should be passed using the KEYS array:<br />
`> eval "return redis.call('set',KEYS[1],'bar')" 1 foo`<br />
`OK`<br />
All Redis commands must be analyzed before execution to determine which keys the command will operate on. In order for this to be true for EVAL, keys must be passed explicitly. This is useful in many ways, but especially to make sure Redis Cluster can forward your request to the appropriate cluster node.<br />
- > **Conversion between Lua and Redis data types**<br />
Redis return values are converted into Lua data types when Lua calls a Redis command using `call()` or `pcall()`. Similarly, Lua data types are converted into the Redis protocol when calling a Redis command and when a Lua script returns a value, so that scripts can control what `EVAL` will return to the client.<br />
This conversion between data types is designed in a way that if a Redis type is converted into a Lua type, and then the result is converted back into a Redis type, the result is the same as the initial value.<br />
In other words there is a one-to-one conversion between Lua and Redis types. The following table shows you all the conversions rules:<br />
Redis to Lua conversion table.<br />
&emsp;&emsp; · Redis integer reply -> Lua number<br />
&emsp;&emsp; · Redis bulk reply -> Lua string<br />
&emsp;&emsp; · Redis multi bulk reply -> Lua table (may have other Redis data types nested)<br />
&emsp;&emsp; · Redis status reply -> Lua table with a single ok field containing the status<br />
&emsp;&emsp; · Redis error reply -> Lua table with a single err field containing the error<br />
&emsp;&emsp; · Redis Nil bulk reply and Nil multi bulk reply -> Lua false boolean type<br />
Lua to Redis conversion table.<br />
&emsp;&emsp; · Lua number -> Redis integer reply (the number is converted into an integer)<br />
&emsp;&emsp; · Lua string -> Redis bulk reply<br />
&emsp;&emsp; · Lua table (array) -> Redis multi bulk reply (truncated to the first nil inside the Lua array if any)<br />
&emsp;&emsp; · Lua table with a single ok field -> Redis status reply<br />
&emsp;&emsp; · Lua table with a single err field -> Redis error reply<br />
&emsp;&emsp; · Lua boolean false -> Redis Nil bulk reply.<br />
There is an additional Lua-to-Redis conversion rule that has no corresponding Redis to Lua conversion rule:<br />
&emsp;&emsp; · Lua boolean true -> Redis integer reply with value of 1.
Also there are two important rules to note:<br />
&emsp;&emsp; · Lua has a single numerical type, Lua numbers. There is no distinction between integers and floats. So we always convert Lua numbers into integer replies, removing the decimal part of the number if any. If you want to return a float from Lua you should return it as a string, exactly like Redis itself does (see for instance the `ZSCORE` command).<br />
&emsp;&emsp; · There is no simple way to have nils inside Lua arrays, this is a result of Lua table semantics, so when Redis converts a Lua array into Redis protocol the conversion is stopped if a nil is encountered.<br />
- > **Helper functions to return Redis types**<br />
There are two helper functions to return Redis types from Lua.<br />
&emsp;&emsp; · `redis.error_reply(error_string)` returns an error reply. This function simply returns a single field table with the err field set to the specified string for you.<br />
&emsp;&emsp; · `redis.status_reply(status_string)` returns a status reply. This function simply returns a single field table with the ok field set to the specified string for you.
There is no difference between using the helper functions or directly returning the table with the specified format, so the following two forms are equivalent:<br />
`return {err="My Error"}`<br />
`return redis.error_reply("My Error")`<br />
- > **Atomicity of scripts**<br />
Redis uses the same Lua interpreter to run all the commands. Also Redis guarantees that a script is executed in an atomic way: no other script or Redis command will be executed while a script is being executed. This semantic is similar to the one of `MULTI / EXEC`. From the point of view of all the other clients the effects of a script are either still not visible or already completed.<br />
However this also means that executing slow scripts is not a good idea. It is not hard to create fast scripts, as the script overhead is very low, but if you are going to use slow scripts you should be aware that while the script is running no other client can execute commands.<br />
- > **Bandwidth and EVALSHA**<br />
The `EVAL` command forces you to send the script body again and again. Redis does not need to recompile the script every time as it uses an internal caching mechanism, however paying the cost of the additional bandwidth may not be optimal in many contexts.<br />
In order to avoid these problems while avoiding the bandwidth penalty, Redis implements the `EVALSHA` command.<br />
`EVALSHA` works exactly like EVAL, but instead of having a script as the first argument it has the SHA1 digest of a script. The behavior is the following:<br />
&emsp;&emsp; · If the server still remembers a script with a matching SHA1 digest, the script is executed.<br />
&emsp;&emsp; · If the server does not remember a script with this SHA1 digest, a special error is returned telling the client to use `EVAL` instead.<br />
Passing keys and arguments as additional `EVAL` arguments is also very useful in this context as the script string remains constant and can be efficiently cached by Redis.<br />

ALEX ZHOU 注：<br />
脚本的 SHA1 值由 `SCRIPT LOAD` 产生，例如：<br />
`> SCRIPT LOAD "return redis.call('get','foo')"`<br />
`"6b1bf486c81ceb7edf3c093f4c48582e38c0e791"`<br />
- > **Script cache semantics**<br />
Executed scripts are guaranteed to be in the script cache of a given execution of a Redis instance forever. This means that if an `EVAL` is performed against a Redis instance all the subsequent `EVALSHA` calls will succeed.<br />
The reason why scripts can be cached for long time is that it is unlikely for a well written application to have enough different scripts to cause memory problems. Every script is conceptually like the implementation of a new command, and even a large application will likely have just a few hundred of them. Even if the application is modified many times and scripts will change, the memory used is negligible.<br />
The only way to flush the script cache is by explicitly calling the `SCRIPT FLUSH` command, which will completely flush the scripts cache removing all the scripts executed so far.<br />
This is usually needed only when the instance is going to be instantiated for another customer or application in a cloud environment.<br />
Also, as already mentioned, restarting a Redis instance flushes the script cache, which is not persistent. However from the point of view of the client there are only two ways to make sure a Redis instance was not restarted between two different commands.<br />
&emsp;&emsp; · The connection we have with the server is persistent and was never closed so far.<br />
&emsp;&emsp; · The client explicitly checks the runid field in the INFO command in order to make sure the server was not restarted and is still the same process.<br />
Practically speaking, for the client it is much better to simply assume that in the context of a given connection, cached scripts are guaranteed to be there unless an administrator explicitly called the `SCRIPT FLUSH` command.<br />
For instance an application with a persistent connection to Redis can be sure that if a script was sent once it is still in memory, so `EVALSHA` can be used against those scripts in a pipeline without the chance of an error being generated due to an unknown script.<br />
A common pattern is to call `SCRIPT LOAD` to load all the scripts that will appear in a pipeline, then use `EVALSHA` directly inside the pipeline without any need to check for errors resulting from the script hash not being recognized.<br />
- > **The SCRIPT command**<br />
Redis offers a `SCRIPT` command that can be used in order to control the scripting subsystem. `SCRIPT` currently accepts three different commands:<br />
&emsp;&emsp; · `SCRIPT FLUSH`<br />
&emsp;&emsp; This command is the only way to force Redis to flush the scripts cache. It is most useful in a cloud environment where the same instance can be reassigned to a different user. It is also useful for testing client libraries' implementations of the scripting feature.<br />
&emsp;&emsp; · `SCRIPT EXISTS sha1 sha2 ... shaN`<br />
&emsp;&emsp; Given a list of SHA1 digests as arguments this command returns an array of 1 or 0, where 1 means the specific SHA1 is recognized as a script already present in the scripting cache, while 0 means that a script with this SHA1 was never seen before (or at least never seen after the latest `SCRIPT FLUSH` command).<br />
&emsp;&emsp; · `SCRIPT LOAD script`<br />
&emsp;&emsp; This command registers the specified script in the Redis script cache. The command is useful in all the contexts where we want to make sure that EVALSHA will not fail (for instance during a pipeline or `MULTI/EXEC` operation), without the need to actually execute the script.<br />
&emsp;&emsp; · `SCRIPT KILL`<br />
&emsp;&emsp; This command is the only way to interrupt a long-running script that reaches the configured maximum execution time for scripts. The `SCRIPT KILL` command can only be used with scripts that did not modify the dataset during their execution (since stopping a read-only script does not violate the scripting engine's guaranteed atomicity).<br />
- > **Scripts as pure functions**<br />
A very important part of scripting is writing scripts that are pure functions. Scripts executed in a Redis instance are, by default, replicated on slaves and into the AOF file by sending the script itself -- not the resulting commands.<br />
The reason is that sending a script to another Redis instance is often much faster than sending the multiple commands the script generates, so if the client is sending many scripts to the master, converting the scripts into individual commands for the slave / AOF would result in too much bandwidth for the replication link or the Append Only File (and also too much CPU since dispatching a command received via network is a lot more work for Redis compared to dispatching a command invoked by Lua scripts).<br />
Normally replicating scripts instead of the effects of the scripts makes sense, let's call this replication mode whole scripts replication, however not in all the cases. So starting with Redis 3.2, the scripting engine is able to, alternatively, replicate the sequence of write commands resulting from the script execution, instead of replication the script itself.<br />
The main drawback with the whole scripts replication approach is that scripts are required to have the following property:<br />
&emsp;&emsp; · The script must always evaluates the same Redis write commands with the same arguments given the same input data set. Operations performed by the script cannot depend on any hidden (non-explicit) information or state that may change as script execution proceeds or between different executions of the script, nor can it depend on any external input from I/O devices.<br />
Things like using the system time, calling Redis random commands like `RANDOMKEY`, or using Lua random number generator, could result into scripts that will not always evaluate in the same way.<br />
In order to enforce this behavior in scripts Redis does the following:<br />
&emsp;&emsp; · Lua does not export commands to access the system time or other external state.<br />
&emsp;&emsp; · Redis will block the script with an error if a script calls a Redis command able to alter the data set after a Redis random command like `RANDOMKEY`, `SRANDMEMBER`, `TIME`. This means that if a script is read-only and does not modify the data set it is free to call those commands. Note that a random command does not necessarily mean a command that uses random numbers: any non-deterministic command is considered a random command (the best example in this regard is the `TIME` command).<br />
&emsp;&emsp; · Redis commands that may return elements in random order, like `SMEMBERS` (because Redis Sets are unordered) have a different behavior when called from Lua, and undergo a silent lexicographical sorting filter before returning data to Lua scripts. So `redis.call("smembers",KEYS[1])` will always return the Set elements in the same order, while the same command invoked from normal clients may return different results even if the key contains exactly the same elements.<br />
`> sadd myset a b c d 1 e f 2 0`<br />
`(integer) 9`<br />
`> smembers myset`<br />
`1) "a"`<br />
`2) "b"`<br />
`3) "2"`<br />
`4) "d"`<br />
`5) "e"`<br />
`6) "c"`<br />
`7) "f"`<br />
`8) "1"`<br />
`9) "0"`<br />
`> eval "return redis.call('smembers', 'myset')" 0`<br />
`1) "0"`<br />
`2) "1"`<br />
`3) "2"`<br />
`4) "a"`<br />
`5) "b"`<br />
`6) "c"`<br />
`7) "d"`<br />
`8) "e"`<br />
`9) "f"`<br />
&emsp;&emsp; · Lua pseudo random number generation functions `math.random` and `math.randomseed` are modified in order to always have the same seed every time a new script is executed. This means that calling `math.random` will always generate the same sequence of numbers every time a script is executed if `math.randomseed` is not used.<br />
However the user is still able to write commands with random behavior using the following simple trick. Imagine I want to write a Redis script that will populate a list with N random integers.<br />
I can start with this small Ruby program:<br />
``` Ruby
require 'rubygems'
require 'redis'

r = Redis.new

RandomPushScript = <<EOF
    local i = tonumber(ARGV[1])
    local res
    while (i > 0) do
        res = redis.call('lpush',KEYS[1],math.random())
        i = i-1
    end
    return res
EOF

r.del(:mylist)
puts r.eval(RandomPushScript,[:mylist],[10,rand(2**32)])
```
> Every time this script executed the resulting list will have exactly the following elements:<br />
```
> lrange mylist 0 -1
 1) "0.74509509873814"
 2) "0.87390407681181"
 3) "0.36876626981831"
 4) "0.6921941534114"
 5) "0.7857992587545"
 6) "0.57730350670279"
 7) "0.87046522734243"
 8) "0.09637165539729"
 9) "0.74990198051087"
10) "0.17082803611217"
```
> In order to make it a pure function, but still be sure that every invocation of the script will result in different random elements, we can simply add an additional argument to the script that will be used in order to seed the Lua pseudo-random number generator. The new script is as follows:<br />
``` Ruby
require 'rubygems'
require 'redis'

r = Redis.new

RandomPushScript = <<EOF
    local i = tonumber(ARGV[1])
    local res
    math.randomseed(tonumber(ARGV[2]))
    while (i > 0) do
        res = redis.call('lpush',KEYS[1],math.random())
        i = i-1
    end
    return res
EOF

r.del(:mylist)
puts r.eval(RandomPushScript,1,:mylist,10,rand(2**32))
```
> What we are doing here is sending the seed of the PRNG as one of the arguments. This way the script output will be the same given the same arguments, but we are changing one of the arguments in every invocation, generating the random seed client-side. The seed will be propagated as one of the arguments both in the replication link and in the Append Only File, guaranteeing that the same changes will be generated when the AOF is reloaded or when the slave processes the script.<br />
Note: an important part of this behavior is that the PRNG that Redis implements as math.random and math.randomseed is guaranteed to have the same output regardless of the architecture of the system running Redis. 32-bit, 64-bit, big-endian and little-endian systems will all produce the same output.<br />

ALEX ZHOU 注：<br />
PRNG(伪随机序列发生器)  pseudo-random number generator<br />
- > **Replicating commands instead of scripts**<br />
Starting with Redis 3.2, it is possible to select an alternative replication method. Instead of replication whole scripts, we can just replicate single write commands generated by the script. We call this script effects replication.<br />
In this replication mode, while Lua scripts are executed, Redis collects all the commands executed by the Lua scripting engine that actually modify the dataset. When the script execution finishes, the sequence of commands that the script generated are wrapped into a `MULTI / EXEC` transaction and are sent to slaves and AOF.<br />
This is useful in several ways depending on the use case:<br />
&emsp;&emsp; · When the script is slow to compute, but the effects can be summarized by a few write commands, it is a shame to re-compute the script on the slaves or when reloading the AOF. In this case to replicate just the effect of the script is much better.<br />
&emsp;&emsp; · When script effects replication is enabled, the controls about non deterministic functions are disabled. You can, for example, use the `TIME` or `SRANDMEMBER` commands inside your scripts freely at any place.<br />
&emsp;&emsp; · The Lua PRNG in this mode is seeded randomly at every call.<br />
In order to enable script effects replication, you need to issue the following Lua command before any write operated by the script:<br />
`redis.replicate_commands()`<br />
The function returns true if the script effects replication was enabled, otherwise if the function was called after the script already called some write command, it returns false, and normal whole script replication is used.<br />
``` Ruby
require 'rubygems'
require 'redis'

r = Redis.new

RandomPushScript = <<EOF
    local i = tonumber(ARGV[1])
    local res
    redis.replicate_commands()
    while (i > 0) do
        res = redis.call('lpush',KEYS[1],math.random())
        i = i-1
    end
    return res
EOF

r.del(:mylist)
puts r.eval(RandomPushScript,1,:mylist,10,rand(2**32))
```
- > **Selective replication of commands**<br />
When script effects replication is selected (see the previous section), it is possible to have more control in the way commands are replicated to slaves and AOF. This is a very advanced feature since a misuse can do damage by breaking the contract that the master, slaves, and AOF, all must contain the same logical content.<br />
However this is a useful feature since, sometimes, we need to execute certain commands only in the master in order to create, for example, intermediate values.<br />
Think at a Lua script where we perform an intersection between two sets. Pick five random elements, and create a new set with this five random elements. Finally we delete the temporary key representing the intersection between the two original sets. What we want to replicate is only the creation of the new set with the five elements. It's not useful to also replicate the commands creating the temporary key.<br />
For this reason, Redis 3.2 introduces a new command that only works when script effects replication is enabled, and is able to control the scripting replication engine. The command is called redis.set_repl() and fails raising an error if called when script effects replication is disabled.<br />
The command can be called with four different arguments:<br />
`redis.set_repl(redis.REPL_ALL) -- Replicate to AOF and slaves.`<br />
`redis.set_repl(redis.REPL_AOF) -- Replicate only to AOF.`<br />
`redis.set_repl(redis.REPL_SLAVE) -- Replicate only to slaves.`<br />
`redis.set_repl(redis.REPL_NONE) -- Don't replicate at all.`<br />
By default the scripting engine is always set to REPL_ALL. By calling this function the user can switch on/off AOF and or slaves replication, and turn them back later at her/his wish.<br />

未完待续，先要完成replication部分

## **Redis 出版订阅**
以下内容摘自 [Redis Pub-Sub server](https://redis.io/topics/pubsub)<br />
- > **Pub/Sub**<br />
`SUBSCRIBE`, `UNSUBSCRIBE` and `PUBLISH` implement the Publish/Subscribe messaging paradigm where (citing Wikipedia) senders (publishers) are not programmed to send their messages to specific receivers (subscribers). Rather, published messages are characterized into channels, without knowledge of what (if any) subscribers there may be. Subscribers express interest in one or more channels, and only receive messages that are of interest, without knowledge of what (if any) publishers there are. This decoupling of publishers and subscribers can allow for greater scalability and a more dynamic network topology.<br />
A client subscribed to one or more channels should not issue commands, although it can subscribe and unsubscribe to and from other channels. The replies to subscription and unsubscription operations are sent in the form of messages, so that the client can just read a coherent stream of messages where the first element indicates the type of message. The commands that are allowed in the context of a subscribed client are `SUBSCRIBE`, `PSUBSCRIBE`, `UNSUBSCRIBE`, `PUNSUBSCRIBE`, `PING` and `QUIT`.<br />
Please note that redis-cli will not accept any commands once in subscribed mode and can only quit the mode with Ctrl-C.<br />
- > **Format of pushed messages**<br />
A message is a [Array reply](https://redis.io/topics/protocol#array-reply) with three elements.<br />
The first element is the kind of message:<br />
&emsp;&emsp; · subscribe: means that we successfully subscribed to the channel given as the second element in the reply. The third argument represents the number of channels we are currently subscribed to.<br />
&emsp;&emsp; · unsubscribe: means that we successfully unsubscribed from the channel given as second element in the reply. The third argument represents the number of channels we are currently subscribed to. When the last argument is zero, we are no longer subscribed to any channel, and the client can issue any kind of Redis command as we are outside the Pub/Sub state.<br />
&emsp;&emsp; · message: it is a message received as result of a PUBLISH command issued by another client. The second element is the name of the originating channel, and the third argument is the actual message payload.<br />
- > **Database & Scoping**<br />
Pub/Sub has no relation to the key space. It was made to not interfere with it on any level, including database numbers.<br />
Publishing on db 10, will be heard by a subscriber on db 1.<br />
If you need scoping of some kind, prefix the channels with the name of the environment (test, staging, production, ...).<br />
- > **Pattern-matching subscriptions**<br />
The Redis Pub/Sub implementation supports pattern matching. Clients may subscribe to glob-style patterns in order to receive all the messages sent to channel names matching a given pattern.<br />
For instance:<br />
`PSUBSCRIBE news.*`<br />
Will receive all the messages sent to the channel news.art.figurative, news.music.jazz, etc. All the glob-style patterns are valid, so multiple wildcards are supported.<br />
`PUNSUBSCRIBE news.*`<br />
Will then unsubscribe the client from that pattern. No other subscriptions will be affected by this call.<br />
Messages received as a result of pattern matching are sent in a different format:<br />
&emsp;&emsp; · The type of the message is pmessage: it is a message received as result of a `PUBLISH` command issued by another client, matching a pattern-matching subscription. The second element is the original pattern matched, the third element is the name of the originating channel, and the last element the actual message payload.<br />
Similarly to SUBSCRIBE and UNSUBSCRIBE, PSUBSCRIBE and PUNSUBSCRIBE commands are acknowledged by the system sending a message of type psubscribe and punsubscribe using the same format as the subscribe and unsubscribe message format.<br />
- > **Messages matching both a pattern and a channel subscription**<br />
A client may receive a single message multiple times if it's subscribed to multiple patterns matching a published message, or if it is subscribed to both patterns and channels matching the message. Like in the following example:<br />
`SUBSCRIBE foo`<br />
`PSUBSCRIBE f*`<br />
In the above example, if a message is sent to channel foo, the client will receive two messages: one of type message and one of type pmessage.<br />
- > **The meaning of the subscription count with pattern matching**<br />
In `subscribe`, `unsubscribe`, `psubscribe` and `punsubscribe` message types, the last argument is the count of subscriptions still active. This number is actually the total number of channels and patterns the client is still subscribed to. So the client will exit the Pub/Sub state only when this count drops to zero as a result of unsubscribing from all the channels and patterns.<br />
- > **Programming example**<br />
Pieter Noordhuis provided a great example using EventMachine and Redis to create [a multi user high performance web chat](https://gist.github.com/pietern/348262).<br />

ALEX ZHOU 注：<br />
需要翻墙访问 https://gist.github.com/pietern/348262<br />
如果需要测试该 Ruby 程序，需要：<br />
`sudo yum install gcc-c++`<br />
`sudo /usr/local/bin/gem install eventmachine`<br />
`sudo /usr/local/bin/gem install sinatra`<br />
`sudo /usr/local/bin/gem install cramp`<br />
但是目前仍然未测试成功，可能是该程序使用的 gem 包版本老旧问题；<br />
建议看懂程序后使用其他语言重新实现一次。<br />
- > **Client library implementation hints**<br />
Because all the messages received contain the original subscription causing the message delivery (the channel in the case of message type, and the original pattern in the case of pmessage type) client libraries may bind the original subscription to callbacks (that can be anonymous functions, blocks, function pointers), using a hash table.<br />
When a message is received an O(1) lookup can be done in order to deliver the message to the registered callback.<br />

## **Redis 批量插入**
以下内容摘自 [Redis Mass Insertion](https://redis.io/topics/mass-insert)<br />
Sometimes Redis instances need to be loaded with a big amount of preexisting or user generated data in a short amount of time, so that millions of keys will be created as fast as possible.<br />
This is called a mass insertion, and the goal of this document is to provide information about how to feed Redis with data as fast as possible.<br />
- > **Use the protocol, Luke**<br />
Using a normal Redis client to perform mass insertion is not a good idea for a few reasons: the naive approach of sending one command after the other is slow because you have to pay for the round trip time for every command. It is possible to use pipelining, but for mass insertion of many records you need to write new commands while you read replies at the same time to make sure you are inserting as fast as possible.<br />
Only a small percentage of clients support non-blocking I/O, and not all the clients are able to parse the replies in an efficient way in order to maximize throughput. For all this reasons the preferred way to mass import data into Redis is to generate a text file containing the Redis protocol, in raw format, in order to call the commands needed to insert the required data.<br />
For instance if I need to generate a large data set where there are billions of keys in the form: 'keyN -> ValueN' I will create a file containing the following commands **in the Redis protocol format**:<br />
`SET Key0 Value0`<br />
`SET Key1 Value1`<br />
`...`<br />
`SET KeyN ValueN`<br />
Once this file is created, the remaining action is to feed it to Redis as fast as possible. In the past the way to do this was to use the netcat with the following command:<br />
`(cat data.txt; sleep 10) | nc localhost 6379 > /dev/null`<br />
However this is not a very reliable way to perform mass import because netcat does not really know when all the data was transferred and can't check for errors. In 2.6 or later versions of Redis the redis-cli utility supports a new mode called pipe mode that was designed in order to perform mass insertion.<br />
Using the pipe mode the command to run looks like the following:<br />
`cat data.txt | redis-cli --pipe`<br />
That will produce an output similar to this:<br />
`All data transferred. Waiting for the last reply...`<br />
`Last reply received from server.`<br />
`errors: 0, replies: 1000000`<br />
The redis-cli utility will also make sure to only redirect errors received from the Redis instance to the standard output.<br />
- > **Generating Redis Protocol**<br />
The Redis protocol is extremely simple to generate and parse, and is Documented here. However in order to generate protocol for the goal of mass insertion you don't need to understand every detail of the protocol, but just that every command is represented in the following way:<br />
`*<args><cr><lf>`<br />
`$<len><cr><lf>`<br />
`<arg0><cr><lf>`<br />
`<arg1><cr><lf>`<br />
`...`<br />
`<argN><cr><lf>`<br />
For instance the command SET key value is represented by the following protocol:<br />
`*3<cr><lf>`<br />
`$3<cr><lf>`<br />
`SET<cr><lf>`<br />
`$3<cr><lf>`<br />
`key<cr><lf>`<br />
`$5<cr><lf>`<br />
`value<cr><lf>`<br />
Or represented as a quoted string:<br />
`"*3\r\n$3\r\nSET\r\n$3\r\nkey\r\n$5\r\nvalue\r\n"`<br />
The file you need to generate for mass insertion is just composed of commands represented in the above way, one after the other.<br />
The following Ruby function generates valid protocol:<br />
``` Ruby
def gen_redis_proto(*cmd)
    proto = ""
    proto << "*"+cmd.length.to_s+"\r\n"
    cmd.each{|arg|
        proto << "$"+arg.to_s.bytesize.to_s+"\r\n"
        proto << arg.to_s+"\r\n"
    }
    proto
end

puts gen_redis_proto("SET","mykey","Hello World!").inspect
```
> Using the above function it is possible to easily generate the key value pairs in the above example, with this program:<br />
``` Ruby
(0...1000).each{|n|
    STDOUT.write(gen_redis_proto("SET","Key#{n}","Value#{n}"))
}
```
> We can run the program directly in pipe to redis-cli in order to perform our first mass import session.<br />
```
$ ruby proto.rb | redis-cli --pipe
All data transferred. Waiting for the last reply...
Last reply received from server.
errors: 0, replies: 1000
```

ALEX ZHOU 注：<br />
实际在 vagrant centos/7 环境下测试，得到结果如下：<br />
&emsp;&emsp; · 插入 100000 条，耗时 0.493313274s<br />
&emsp;&emsp; · 插入 1000000 条，耗时 4.915205049s<br />
测试程序如下：
``` Ruby
def gen_redis_proto(*cmd)
    proto = ""
    proto << "*"+cmd.length.to_s+"\r\n"
    cmd.each{|arg|
        proto << "$"+arg.to_s.bytesize.to_s+"\r\n"
        proto << arg.to_s+"\r\n"
    }
    proto
end

start = Time.now
(0...1000000).each{|n|
    STDOUT.write(gen_redis_proto("SET","Key#{n}","Value#{n}"))
}
STDOUT.write(gen_redis_proto("SET", "Elapsed", (Time.now-start).to_s))
```
100000 条返回：<br />
```
> get Elapsed
"0.493313274"
```
1000000 条返回：<br />
``` 
> get Elapsed
"4.915205049"
```
- > **How the pipe mode works under the hoods**<br />
The magic needed inside the pipe mode of redis-cli is to be as fast as netcat and still be able to understand when the last reply was sent by the server at the same time.<br />
This is obtained in the following way:<br />
&emsp;&emsp; · `redis-cli --pipe` tries to send data as fast as possible to the server.<br />
&emsp;&emsp; · At the same time it reads data when available, trying to parse it.<br />
&emsp;&emsp; · Once there is no more data to read from stdin, it sends a special `ECHO` command with a random 20 bytes string: we are sure this is the latest command sent, and we are sure we can match the reply checking if we receive the same 20 bytes as a bulk reply.<br />
&emsp;&emsp; · Once this special final command is sent, the code receiving replies starts to match replies with this 20 bytes. When the matching reply is reached it can exit with success.<br />
Using this trick we don't need to parse the protocol we send to the server in order to understand how many commands we are sending, but just the replies.<br />
However while parsing the replies we take a counter of all the replies parsed so that at the end we are able to tell the user the amount of commands transferred to the server by the mass insert session.<br />

## **Redis 事务**
`MULTI`, `EXEC`, `DISCARD` and `WATCH` are the foundation of transactions in Redis. They allow the execution of a group of commands in a single step, with two important guarantees:<br />
&emsp;&emsp; · All the commands in a transaction are serialized and executed sequentially. It can never happen that a request issued by another client is served in the middle of the execution of a Redis transaction. This guarantees that the commands are executed as a single isolated operation.<br />
&emsp;&emsp; · Either all of the commands or none are processed, so a Redis transaction is also atomic. The `EXEC` command triggers the execution of all the commands in the transaction, so if a client loses the connection to the server in the context of a transaction before calling the `MULTI` command none of the operations are performed, instead if the `EXEC` command is called, all the operations are performed. When using the append-only file Redis makes sure to use a single write(2) syscall to write the transaction on disk. However if the Redis server crashes or is killed by the system administrator in some hard way it is possible that only a partial number of operations are registered. Redis will detect this condition at restart, and will exit with an error. Using the `redis-check-aof` tool it is possible to fix the append only file that will remove the partial transaction so that the server can start again.<br />
Starting with version 2.2, Redis allows for an extra guarantee to the above two, in the form of optimistic locking in a way very similar to a check-and-set (CAS) operation.<br />
- > **Usage**<br />
A Redis transaction is entered using the `MULTI` command. The command always replies with OK. At this point the user can issue multiple commands. Instead of executing these commands, Redis will queue them. All the commands are executed once `EXEC` is called.<br />
Calling `DISCARD` instead will flush the transaction queue and will exit the transaction.<br />
- > **Errors inside a transaction**<br />
During a transaction it is possible to encounter two kind of command errors:<br />
&emsp;&emsp; · A command may fail to be queued, so there may be an error before EXEC is called. For instance the command may be syntactically wrong (wrong number of arguments, wrong command name, ...), or there may be some critical condition like an out of memory condition (if the server is configured to have a memory limit using the maxmemory directive).<br />
&emsp;&emsp; · A command may fail after EXEC is called, for instance since we performed an operation against a key with the wrong value (like calling a list operation against a string value).<br />
However starting with Redis 2.6.5, the server will remember that there was an error during the accumulation of commands, and will refuse to execute the transaction returning also an error during `EXEC`, and discarding the transaction automatically.<br />
Errors happening after `EXEC` instead are not handled in a special way: all the other commands will be executed even if some command fails during the transaction.<br />
It's important to note that even when a command fails, all the other commands in the queue are processed – Redis will not stop the processing of commands.<br />
- > **Why Redis does not support roll backs?**<br />
If you have a relational databases background, the fact that Redis commands can fail during a transaction, but still Redis will execute the rest of the transaction instead of rolling back, may look odd to you.<br />
However there are good opinions for this behavior:<br />
&emsp;&emsp; · Redis commands can fail only if called with a wrong syntax (and the problem is not detectable during the command queueing), or against keys holding the wrong data type: this means that in practical terms a failing command is the result of a programming errors, and a kind of error that is very likely to be detected during development, and not in production.<br />
&emsp;&emsp; · Redis is internally simplified and faster because it does not need the ability to roll back.<br />
An argument against Redis point of view is that bugs happen, however it should be noted that in general the roll back does not save you from programming errors. For instance if a query increments a key by 2 instead of 1, or increments the wrong key, there is no way for a rollback mechanism to help. Given that no one can save the programmer from his or her errors, and that the kind of errors required for a Redis command to fail are unlikely to enter in production, we selected the simpler and faster approach of not supporting roll backs on errors.<br />
- > **Discarding the command queue**<br />
`DISCARD` can be used in order to abort a transaction. In this case, no commands are executed and the state of the connection is restored to normal.
- > **Optimistic locking using check-and-set**<br />
`WATCH` is used to provide a check-and-set (CAS) behavior to Redis transactions.<br />
WATCHed keys are monitored in order to detect changes against them. If at least one watched key is modified before the `EXEC` command, the whole transaction aborts, and `EXEC` returns a Null reply to notify that the transaction failed.<br />
Thanks to WATCH we are able to model the problem very well:<br >
`WATCH mykey`<br />
`val = GET mykey`<br />
`val = val + 1`<br />
`MULTI`<br />
`SET mykey $val`<br />
`EXEC`<br />
Using the above code, if there are race conditions and another client modifies the result of val in the time between our call to `WATCH` and our call to `EXEC`, the transaction will fail.<br />
We just have to repeat the operation hoping this time we'll not get a new race. This form of locking is called optimistic locking and is a very powerful form of locking. In many use cases, multiple clients will be accessing different keys, so collisions are unlikely – usually there's no need to repeat the operation.<br />
- > **WATCH explained**<br />
So what is `WATCH` really about? It is a command that will make the `EXEC` conditional: we are asking Redis to perform the transaction only if none of the WATCHed keys were modified. (But they might be changed by the same client inside the transaction without aborting it. [More on this](https://github.com/antirez/redis-doc/issues/734).) Otherwise the transaction is not entered at all. (Note that if you `WATCH` a volatile key and Redis expires the key after you WATCHed it, `EXEC` will still work. [More on this](http://code.google.com/p/redis/issues/detail?id=270).)<br />
`WATCH` can be called multiple times. Simply all the WATCH calls will have the effects to watch for changes starting from the call, up to the moment `EXEC` is called. You can also send any number of keys to a single `WATCH` call.<br />
When `EXEC` is called, all keys are UNWATCHed, regardless of whether the transaction was aborted or not. Also when a client connection is closed, everything gets UNWATCHed.<br />
It is also possible to use the `UNWATCH` command (without arguments) in order to flush all the watched keys. Sometimes this is useful as we optimistically lock a few keys, since possibly we need to perform a transaction to alter those keys, but after reading the current content of the keys we don't want to proceed. When this happens we just call `UNWATCH` so that the connection can already be used freely for new transactions.<br />
- > **Using WATCH to implement ZPOP**<br />
A good example to illustrate how `WATCH` can be used to create new atomic operations otherwise not supported by Redis is to implement `ZPOP`, that is a command that pops the element with the lower score from a sorted set in an atomic way. This is the simplest implementation:<br />
```
WATCH zset
element = ZRANGE zset 0 0
MULTI
ZREM zset element
EXEC
```
> If `EXEC` fails (i.e. returns a Null reply) we just repeat the operation.<br />
- > **Redis scripting and transactions**<br />
A Redis script is transactional by definition, so everything you can do with a Redis transaction, you can also do with a script, and usually the script will be both simpler and faster.<br />
This duplication is due to the fact that scripting was introduced in Redis 2.6 while transactions already existed long before. However we are unlikely to remove the support for transactions in the short time because it seems semantically opportune that even without resorting to Redis scripting it is still possible to avoid race conditions, especially since the implementation complexity of Redis transactions is minimal.<br />
However it is not impossible that in a non immediate future we'll see that the whole user base is just using scripts. If this happens we may deprecate and finally remove transactions.<br />

## **Redis 分布式锁**
Distributed locks are a very useful primitive in many environments where different processes must operate with shared resources in a mutually exclusive way.<br />
There are a number of libraries and blog posts describing how to implement a DLM (Distributed Lock Manager) with Redis, but every library uses a different approach, and many use a simple approach with lower guarantees compared to what can be achieved with slightly more complex designs.<br />
This page is an attempt to provide a more canonical algorithm to implement distributed locks with Redis. We propose an algorithm, called `Redlock`, which implements a DLM which we believe to be safer than the vanilla single instance approach. We hope that the community will analyze it, provide feedback, and use it as a starting point for the implementations or more complex or alternative designs.<br />
- > **Safety and Liveness guarantees**<br />
We are going to model our design with just three properties that, from our point of view, are the minimum guarantees needed to use distributed locks in an effective way.<br />
&emsp;&emsp; · Safety property: Mutual exclusion. At any given moment, only one client can hold a lock.<br />
&emsp;&emsp; · Liveness property A: Deadlock free. Eventually it is always possible to acquire a lock, even if the client that locked a resource crashes or gets partitioned.<br />
&emsp;&emsp; · Liveness property B: Fault tolerance. As long as the majority of Redis nodes are up, clients are able to acquire and release locks.<br />
- > **Why failover-based implementations are not enough**<br />
To understand what we want to improve, let’s analyze the current state of affairs with most Redis-based distributed lock libraries.<br />
The simplest way to use Redis to lock a resource is to create a key in an instance. The key is usually created with a limited time to live, using the Redis expires feature, so that eventually it will get released (property 2 in our list). When the client needs to release the resource, it deletes the key.<br />
Superficially this works well, but there is a problem: this is a single point of failure in our architecture. What happens if the Redis master goes down? Well, let’s add a slave! And use it if the master is unavailable. This is unfortunately not viable. By doing so we can’t implement our safety property of mutual exclusion, because Redis replication is asynchronous.<br />
There is an obvious race condition with this model:<br />
vClient A acquires the lock in the master.<br />
&emsp;&emsp; · The master crashes before the write to the key is transmitted to the slave.<br />
&emsp;&emsp; · The slave gets promoted to master.<br />
&emsp;&emsp; · Client B acquires the lock to the same resource A already holds a lock for. SAFETY VIOLATION!<br />
Sometimes it is perfectly fine that under special circumstances, like during a failure, multiple clients can hold the lock at the same time. If this is the case, you can use your replication based solution. Otherwise we suggest to implement the solution described in this document.<br />
- > **Correct implementation with a single instance**<br />
Before trying to overcome the limitation of the single instance setup described above, let’s check how to do it correctly in this simple case, since this is actually a viable solution in applications where a race condition from time to time is acceptable, and because locking into a single instance is the foundation we’ll use for the distributed algorithm described here.<br />
To acquire the lock, the way to go is the following:<br />
`SET resource_name my_random_value NX PX 30000`<br />
The command will set the key only if it does not already exist (NX option), with an expire of 30000 milliseconds (PX option). The key is set to a value “myrandomvalue”. This value must be unique across all clients and all lock requests.<br />
Basically the random value is used in order to release the lock in a safe way, with a script that tells Redis: remove the key only if it exists and the value stored at the key is exactly the one I expect to be. This is accomplished by the following Lua script:<br />
``` Lua
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```
> This is important in order to avoid removing a lock that was created by another client. For example a client may acquire the lock, get blocked in some operation for longer than the lock validity time (the time at which the key will expire), and later remove the lock, that was already acquired by some other client. Using just `DEL` is not safe as a client may remove the lock of another client. With the above script instead every lock is “signed” with a random string, so the lock will be removed only if it is still the one that was set by the client trying to remove it.<br />
What should this random string be? I assume it’s 20 bytes from /dev/urandom, but you can find cheaper ways to make it unique enough for your tasks. For example a safe pick is to seed RC4 with /dev/urandom, and generate a pseudo random stream from that. **A simpler solution** is to use a combination of unix time with microseconds resolution, concatenating it with a client ID, it is not as safe, but probably **up to the task in most environments**.<br />
The time we use as the key time to live, is called the “lock validity time”. It is both the auto release time, and the time the client has in order to perform the operation required before another client may be able to acquire the lock again, without technically violating the mutual exclusion guarantee, which is only limited to a given window of time from the moment the lock is acquired.<br />
So now we have a good way to acquire and release the lock. The system, reasoning about a non-distributed system composed of a single, always available, instance, is safe. Let’s extend the concept to a distributed system where we don’t have such guarantees.<br />
### **The Redlock algorithm**
In the distributed version of the algorithm we assume we have N Redis masters. Those nodes are totally independent, so we don’t use replication or any other implicit coordination system. We already described how to acquire and release the lock safely in a single instance. We take for granted that the algorithm will use this method to acquire and release the lock in a single instance. In our examples we set N=5, which is a reasonable value, so we need to run 5 Redis masters on different computers or virtual machines in order to ensure that they’ll fail in a mostly independent way.<br />
In order to acquire the lock, the client performs the following operations:<br />
1. It gets the current time in milliseconds.<br />
2. It tries to acquire the lock in all the N instances sequentially, using the same key name and random value in all the instances. During step 2, when setting the lock in each instance, the client uses a timeout which is small compared to the total lock auto-release time in order to acquire it. For example if the auto-release time is 10 seconds, the timeout could be in the ~ 5-50 milliseconds range. This prevents the client from remaining blocked for a long time trying to talk with a Redis node which is down: if an instance is not available, we should try to talk with the next instance ASAP.<br />
3. The client computes how much time elapsed in order to acquire the lock, by subtracting from the current time the timestamp obtained in step 1. If and only if the client was able to acquire the lock in the majority of the instances (at least 3), and the total time elapsed to acquire the lock is less than lock validity time, the lock is considered to be acquired.<br />
4. If the lock was acquired, its validity time is considered to be the initial validity time minus the time elapsed, as computed in step 3.<br />
5. If the client failed to acquire the lock for some reason (either it was not able to lock N/2+1 instances or the validity time is negative), it will try to unlock all the instances (even the instances it believed it was not able to lock).<br />

## **Redis Keyspace Notifications**
以下内容摘自 [Redis Keyspace Notifications](https://redis.io/topics/notifications)<br />
- > **Feature overview** <br />
Keyspace notifications allows clients to subscribe to Pub/Sub channels in order to receive events affecting the Redis data set in some way.<br />
Examples of the events that is possible to receive are the following:<br />
&emsp;&emsp; · All the commands affecting a given key.<br />
&emsp;&emsp; · All the keys receiving an LPUSH operation.<br />
&emsp;&emsp; · All the keys expiring in the database 0.<br />
Events are delivered using the normal Pub/Sub layer of Redis, so clients implementing Pub/Sub are able to use this feature without modifications.<br />
Because Redis Pub/Sub is fire and forget currently there is no way to use this feature if your application demands reliable notification of events, that is, if your Pub/Sub client disconnects, and reconnects later, all the events delivered during the time the client was disconnected are lost.<br />
In the future there are plans to allow for more reliable delivering of events, but probably this will be addressed at a more general level either bringing reliability to Pub/Sub itself, or allowing Lua scripts to intercept Pub/Sub messages to perform operations like pushing the events into a list.<br />
- > **Type of events**<br />
Keyspace notifications are implemented sending two distinct type of events for every operation affecting the Redis data space. For instance a DEL operation targeting the key named mykey in database 0 will trigger the delivering of two messages, exactly equivalent to the following two `PUBLISH` commands:<br />
`PUBLISH __keyspace@0__:mykey del`<br />
`PUBLISH __keyevent@0__:del mykey`<br />
It is easy to see how one channel allows to listen to all the events targeting the key mykey and the other channel allows to obtain information about all the keys that are target of a del operation.<br />
The first kind of event, with keyspace prefix in the channel is called a Key-space notification, while the second, with the keyevent prefix, is called a Key-event notification.<br />
In the above example a del event was generated for the key mykey. What happens is that:<br />
&emsp;&emsp; · The Key-space channel receives as message the name of the event.<br />
&emsp;&emsp; · The Key-event channel receives as message the name of the key.<br />
It is possible to enable only one kind of notification in order to deliver just the subset of events we are interested in.<br />
- > **Configuration**<br />
By default keyspace events notifications are disabled because while not very sensible the feature uses some CPU power. Notifications are enabled using the notify-keyspace-events of redis.conf or via the `CONFIG SET`.<br />
Setting the parameter to the empty string disables notifications. In order to enable the feature a non-empty string is used, composed of multiple characters, where every character has a special meaning according to the following table:<br />
```
K     Keyspace events, published with __keyspace@<db>__ prefix.
E     Keyevent events, published with __keyevent@<db>__ prefix.
g     Generic commands (non-type specific) like DEL, EXPIRE, RENAME, ...
$     String commands
l     List commands
s     Set commands
h     Hash commands
z     Sorted set commands
x     Expired events (events generated every time a key expires)
e     Evicted events (events generated when a key is evicted for maxmemory)
A     Alias for g$lshzxe, so that the "AKE" string means all the events.
```
At least K or E should be present in the string, otherwise no event will be delivered regardless of the rest of the string.<br />
For instance to enable just Key-space events for lists, the configuration parameter must be set to Kl, and so forth.<br />
The string KEA can be used to enable every possible event.<br />
`CONFIG SET notify-keyspace-events KEA`<br />
- > **Events generated by different commands**<br />
Different commands generate different kind of events according to the following list.<br />
&emsp;&emsp; · `DEL` generates a del event for every deleted key.<br />
&emsp;&emsp; · `RENAME` generates two events, a rename_from event for the source key, and a rename_to event for the destination key.<br />
&emsp;&emsp; · `EXPIRE` generates an expire event when an expire is set to the key, or an expired event every time a positive timeout set on a key results into the key being deleted (see `EXPIRE` documentation for more info).<br />
&emsp;&emsp; · `SORT` generates a sortstore event when `STORE` is used to set a new key. If the resulting list is empty, and the `STORE` option is used, and there was already an existing key with that name, the result is that the key is deleted, so a del event is generated in this condition.<br />
```
lpush mylist 3
lpush mylist 2
lpush mylist 1
lpush mylist2 0
sort mylist limit 0 0 mylist2
```
&emsp;&emsp; · `SET` and all its variants (`SETEX`, `SETNX`, `GETSET`) generate set events. However `SETEX` will also generate an expire events.<br />
&emsp;&emsp; · `MSET` generates a separated set event for every key.<br />
&emsp;&emsp; · `SETRANGE` generates a setrange event.<br />
&emsp;&emsp; · `INCR`, `DECR`, `INCRBY`, `DECRBY` commands all generate incrby events.<br />
&emsp;&emsp; · `INCRBYFLOAT` generates an incrbyfloat events.<br />
&emsp;&emsp; · `APPEND` generates an append event.<br />
&emsp;&emsp; · `LPUSH` and `LPUSHX` generates a single lpush event, even in the variadic case.<br />
&emsp;&emsp; · `RPUSH` and `RPUSHX` generates a single rpush event, even in the variadic case.<br />
&emsp;&emsp; · `RPOP` generates an rpop event. Additionally a del event is generated if the key is removed because the last element from the list was popped.<br />
&emsp;&emsp; · `LPOP` generates an lpop event. Additionally a del event is generated if the key is removed because the last element from the list was popped.<br />
&emsp;&emsp; · `LINSERT` generates an linsert event.<br />
&emsp;&emsp; · `LSET` generates an lset event.<br />
&emsp;&emsp; · `LREM` generates an lrem event, and additionally a del event if the resulting list is empty and the key is removed.<br />
&emsp;&emsp; · `LTRIM` generates an ltrim event, and additionally a del event if the resulting list is empty and the key is removed.<br />
&emsp;&emsp; · `RPOPLPUSH` and BRPOPLPUSH generate an rpop event and an lpush event. In both cases the order is guaranteed (the lpush event will always be delivered after the rpop event). Additionally a del event will be generated if the resulting list is zero length and the key is removed.<br />
&emsp;&emsp; · `HSET`, `HSETNX` and `HMSET` all generate a single hset event.<br />
&emsp;&emsp; · `HINCRBY` generates an hincrby event.<br />
&emsp;&emsp; · `HINCRBYFLOAT` generates an hincrbyfloat event.<br />
&emsp;&emsp; · `HDEL` generates a single hdel event, and an additional del event if the resulting hash is empty and the key is removed.<br />
&emsp;&emsp; · `SADD` generates a single sadd event, even in the variadic case.<br />
&emsp;&emsp; · `SREM` generates a single srem event, and an additional del event if the resulting set is empty and the key is removed.<br />
&emsp;&emsp; · `SMOVE` generates an srem event for the source key, and an sadd event for the destination key.<br />
&emsp;&emsp; · `SPOP` generates an spop event, and an additional del event if the resulting set is empty and the key is removed.<br />
&emsp;&emsp; · `SINTERSTORE`, `SUNIONSTORE`, `SDIFFSTORE` generate sinterstore, sunionostore, sdiffstore events respectively. In the special case the resulting set is empty, and the key where the result is stored already exists, a del event is generated since the key is removed.<br />
&emsp;&emsp; · `ZINCR` generates a zincr event.<br />
&emsp;&emsp; · `ZADD` generates a single zadd event even when multiple elements are added.<br />
&emsp;&emsp; · `ZREM` generates a single zrem event even when multiple elements are deleted. When the resulting sorted set is empty and the key is generated, an additional del event is generated.<br />
&emsp;&emsp; · `ZREMBYSCORE` generates a single zrembyscore event. When the resulting sorted set is empty and the key is generated, an additional del event is generated.<br />
&emsp;&emsp; · `ZREMBYRANK` generates a single zrembyrank event. When the resulting sorted set is empty and the key is generated, an additional del event is generated.<br />
&emsp;&emsp; · `ZINTERSTORE` and `ZUNIONSTORE` respectively generate zinterstore and zunionstore events. In the special case the resulting sorted set is empty, and the key where the result is stored already exists, a del event is generated since the key is removed.<br />
&emsp;&emsp; · Every time a key with a time to live associated is removed from the data set because it expired, an expired event is generated.<br />
&emsp;&emsp; · Every time a key is evicted from the data set in order to free memory as a result of the maxmemory policy, an evicted event is generated.<br />
**IMPORTANT** all the commands generate events only if the target key is really modified. For instance an `SREM` deleting a non-existing element from a Set will not actually change the value of the key, so no event will be generated.<br />
- > **Timing of expired events**<br />
Keys with a time to live associated are expired by Redis in two ways:<br />
&emsp;&emsp; · When the key is accessed by a command and is found to be expired.<br />
&emsp;&emsp; · Via a background system that looks for expired keys in background, incrementally, in order to be able to also collect keys that are never accessed.<br />
The expired events are generated when a key is accessed and is found to be expired by one of the above systems, as a result there are no guarantees that the Redis server will be able to generate the expired event at the time the key time to live reaches the value of zero.<br />
If no command targets the key constantly, and there are many keys with a TTL associated, there can be a significant delay between the time the key time to live drops to zero, and the time the expired event is generated.<br />
Basically expired events are generated when the Redis server deletes the key and not when the time to live theoretically reaches the value of zero.<br />

## **Redis 二级索引**
以下内容摘自 [Secondary indexing with Redis](https://redis.io/topics/indexes#secondary-indexing-with-redis)<br />
这是一篇很有意思的文章，其中提到的 Representing and querying graphs using an hexastore 以及 Multi dimensional indexes 等内容值得再次深入研读。<br />
### **Secondary indexing with Redis**
Redis is not exactly a key-value store, since values can be complex data structures. However it has an external key-value shell: at API level data is addressed by the key name. It is fair to say that, natively, Redis only offers primary key access. However since Redis is a data structures server, its capabilities can be used for indexing, in order to create secondary indexes of different kinds, including composite (multi-column) indexes.<br />
This document explains how it is possible to create indexes in Redis using the following data structures:<br />
&emsp;&emsp; · Sorted sets to create secondary indexes by ID or other numerical fields.<br />
&emsp;&emsp; · Sorted sets with lexicographical ranges for creating more advanced secondary indexes, composite indexes and graph traversal indexes.<br />
&emsp;&emsp; · Sets for creating random indexes.<br />
&emsp;&emsp; · Lists for creating simple iterable indexes and last N items indexes.<br />
Implementing and maintaining indexes with Redis is an advanced topic, so most users that need to perform complex queries on data should understand if they are better served by a relational store. **However often, especially in caching scenarios, there is the explicit need to store indexed data into Redis in order to speedup common queries which require some form of indexing in order to be executed.**<br />
### **Simple numerical indexes with sorted sets**
The simplest secondary index you can create with Redis is by using the sorted set data type, which is a data structure representing a set of elements ordered by a floating point number which is the score of each element. Elements are ordered from the smallest to the highest score.<br />
Since the score is a double precision float, indexes you can build with vanilla sorted sets are limited to things where the indexing field is a number within a given range.<br />
The two commands to build these kind of indexes are `ZADD` and `ZRANGEBYSCORE` to respectively add items and retrieve items within a specified range.<br />
By using the `WITHSCORES` option of ZRANGEBYSCORE it is also possible to obtain the scores associated with the returned elements.<br />
The `ZCOUNT` command can be used in order to retrieve the number of elements within a given range, without actually fetching the elements, which is also useful, especially given the fact the operation is executed in logarithmic time regardless of the size of the range.<br />
Ranges can be inclusive or exclusive, please refer to the `ZRANGEBYSCORE` command documentation for more information.
Note: Using the `ZREVRANGEBYSCORE` it is possible to query a range in reversed order, which is often useful when data is indexed in a given direction (ascending or descending) but we want to retrieve information the other way around.<br />
#### **Using objects IDs as associated values**
In the above example we associated names to ages. However in general we may want to index some field of an object which is stored elsewhere. Instead of using the sorted set value directly to store the data associated with the indexed field, it is possible to store just the ID of the object.<br />
In the next examples we'll almost **always use IDs** as values associated with the index, since this is usually **the more sounding design**, with a few exceptions.<br />
#### **Updating simple sorted set indexes**
Often we index things which change over time. In the above example, the age of the user changes every year. In such a case it would make sense to use the birth date as index instead of the age itself, but there are other cases where we simple want some field to change from time to time, and the index to reflect this change.<br />
The `ZADD` command makes updating simple indexes a very trivial operation since re-adding back an element with a different score and the same value will simply update the score and move the element at the right position.<br />
The operation may be wrapped in a `MULTI/EXEC` transaction in order to make sure both fields are updated or none.<br />
#### **Turning multi dimensional data into linear data**
Indexes created with sorted sets are able to index only a single numerical value. Because of this you may think it is impossible to index something which has multiple dimensions using this kind of indexes, but actually this is not always true. If you can efficiently represent something multi-dimensional in a linear way, they it is often possible to use a simple sorted set for indexing.<br />
For example the **[Redis geo indexing API](https://redis.io/commands/geoadd)** uses a sorted set to index places by latitude and longitude using a technique called **[Geo hash](https://www.wikiwand.com/en/Geohash)**. The sorted set score represents alternating bits of longitude and latitude, so that we map the linear score of a sorted set to many small squares in the earth surface. By doing an 8+1 style center plus neighborhoods search it is possible to retrieve elements by radius.<br />
#### **Limits of the score**
Sorted set elements scores are double precision integers. It means that they can represent different decimal or integer values with different errors, because they use an exponential representation internally. However what is interesting for indexing purposes is that the score is always able to **represent without any error numbers** between -9007199254740992 and 9007199254740992, which is -/+ 2^53.<br />
When representing much larger numbers, you need a different form of indexing that is able to index numbers at any precision, called a lexicographical index.<br />

### **Lexicographical indexes**
Redis sorted sets have an interesting property. When elements are added with the same score, they are sorted lexicographically, comparing the strings as binary data with the memcmp() function.<br />
There are commands such as `ZRANGEBYLEX` and `ZLEXCOUNT` that are able to query and count ranges in a lexicographically fashion, assuming they are used with sorted sets where **all the elements have the same score**.<br />
This Redis feature is basically equivalent to a b-tree data structure which is often used in order to implement indexes with traditional databases. As you can guess, because of this, it is possible to use this Redis data structure in order to implement pretty fancy indexes.<br />
Note that in the range queries we prefixed the min and max elements identifying the range with the special characters [ and (. This prefixes are mandatory, and they specify if the elements of the range are inclusive or exclusive. So the range [a (b means give me all the elements lexicographically between a inclusive and b exclusive, which are all the elements starting with a.<br />
There are also two more special characters indicating the infinitely negative string and the infinitely positive string, which are - and +.<br />
#### **A first example: completion**
An interesting application of indexing is completion. Completion is what happens when you start typing your query into a search engine: the user interface will anticipate what you are likely typing, providing common queries that start with the same characters.<br />
A naive approach to completion is to just add every single query we get from the user into the index.<br />
Then when we want to complete the user input, we execute a range query using ZRANGEBYLEX. Imagine the user is typing "bit" inside the search form, and we want to offer possible search keywords starting for "bit". We send Redis a command like that:<br />
`ZRANGEBYLEX myindex "[bit" "[bit\xff"`<br />
Basically we create a range using the string the user is typing right now as start, and the same string plus a trailing byte set to 255, which is \xff in the example, as the end of the range. This way we get all the strings that start for the string the user is typing.<br />
ALEX 注：<br />
此处必须使用双引号！<br />
Note that we don't want too many items returned, so we may use the LIMIT option in order to reduce the number of results.<br />
#### **Adding frequency into the mix**
The above approach is a bit naive, because all the user searches are the same in this way. In a real system we want to complete strings according to their frequency: very popular searches will be proposed with an higher probability compared to search strings typed very rarely.<br />
In order to implement something which depends on the frequency, and at the same time automatically adapts to future inputs, by purging searches that are no longer popular, we can use a very simple *streaming algorithm*.<br />
To start, we modify our index in order to store not just the search term, but also the frequency the term is associated with. So instead of just adding banana we add banana:1, where 1 is the frequency.<br />
`ZADD myindex 0 banana:1`<br />
We also need logic in order to increment the index if the search term already exists in the index, so what we'll actually do is something like that:<br />
`ZRANGEBYLEX myindex "[banana:" + LIMIT 0 1`<br />
`1) "banana:1"`<br />
This will return the single entry of banana if it exists. Then we can increment the associated frequency and send the following two commands:<br />
`ZREM myindex 0 banana:1`<br />
`ZADD myindex 0 banana:2`<br />
Note that because it is possible that there are concurrent updates, the above three commands should be send via a Lua script instead, so that the Lua script will atomically get the old count and re-add the item with incremented score.<br />
So the result will be that, every time a user searches for banana we'll get our entry updated.<br />
There is more: our goal is to just have items searched very frequently. So we need some form of purging. When we actually query the index in order to complete the user input, we may see something like that:<br />
`ZRANGEBYLEX myindex "[banana:" + LIMIT 1 10`<br />
`1) "banana:123"`<br />
`2) "banahhh:1"`<br />
`3) "banned user:49"`<br />
`4) "banning:89"`<br />
This is what we can do. Out of the returned items, we pick a random one, decrement its score by one, and re-add it with the new score. However if the score reaches 0, we simply remove the item from the list. You can use much more advanced systems, but the idea is that the index in the long run will contain top searches, and if top searches will change over the time it will adapt automatically.<br />
A refinement to this algorithm is to pick entries in the list according to their weight: the higher the score, the less likely entries are picked in order to decrement its score, or evict them.<br />
#### **Normalizing strings for case and accents**
In the completion examples we always used lowercase strings. However reality is much more complex than that: languages have capitalized names, accents, and so forth.<br />
One simple way do deal with this issues is to actually normalize the string the user searches. Whatever the user searches for "Banana", "BANANA" or "Ba'nana" we may always turn it into "banana".<br />
However sometimes we may like to present the user with the original item typed, even if we normalize the string for indexing. In order to do this, what we do is to change the format of the index so that instead of just storing term:frequency we store normalized:frequency:original like in the following example:<br />
`ZADD myindex 0 banana:273:Banana`<br />
#### **Adding auxiliary information in the index**
In general we can add any kind of associated value to our indexing key. In order to use a lexicographical index to implement a simple key-value store we just store the entry as key:value:<br />
`ZADD myindex 0 mykey:myvalue`<br />
And search for the key with:<br />
`ZRANGEBYLEX myindex mykey: + LIMIT 1 1`<br />
`1) "mykey:myvalue"`<br />
Then we extract the part after the colon to retrieve the value. However a problem to solve in this case is collisions. The colon character may be part of the key itself, so it must be chosen in order to never collide with the key we add.<br />
Since lexicographical ranges in Redis are binary safe you can use any byte or any sequence of bytes. However if you receive untrusted user input, it is better to use some form of escaping in order to guarantee that the separator will never happen to be part of the key.<br />
For example if you use two null bytes as separator "\0\0", you may want to always escape null bytes into two bytes sequences in your strings.<br />
#### **Numerical padding**
Lexicographical indexes may look like good only when the problem at hand is to index strings. Actually it is very simple to use this kind of index in order to perform indexing of arbitrary precision numbers.<br />
In the ASCII character set, digits appear in the order from 0 to 9, so if we left-pad numbers with leading zeroes, the result is that comparing them as strings will order them by their numerical value.<br />
We effectively created an index using a numerical field which can be as big as we want. This also works with floating point numbers of any precision by making sure we left pad the numerical part with leading zeroes and the decimal part with trailing zeroes.<br />
#### **Using numbers in binary form**
Storing numbers in decimal may use too much memory. An alternative approach is just to store numbers, for example 128 bit integers, directly in their binary form. However for this to work, you need to store the numbers in big endian format, so that the most significant bytes are stored before the least significant bytes. This way when Redis compares the strings with memcmp(), it will effectively sort the numbers by their value.<br />
Keep in mind that data stored in binary format is less observable for debugging, harder to parse and export. So it is definitely a trade off.<br />
### **Composite indexes**
I need to run queries in order to retrieve all the products in a given room having a given price range. What I can do is to index each product in the following way:<br />
`ZADD myindex 0 0056:0028.44:90`<br />
`ZADD myindex 0 0034:0011.00:832`<br />
With an index like that, to get all the products in room 56 having a price between 10 and 30 dollars is very easy. We can just run the following command:<br />
`ZRANGEBYLEX myindex [0056:0010.00 [0056:0030.00`<br />
### **Updating lexicographical indexes**
The value of the index in a lexicographical index can get pretty fancy and hard or slow to rebuild from what we store about the object. So one approach to simplify the handling of the index, at the cost of using more memory, is to also take alongside to the sorted set representing the index a hash mapping the object ID to the current index value.<br />
So for example, when we index we also add to a hash:<br />
`MULTI`<br />
`ZADD myindex 0 0056:0028.44:90`<br />
`HSET index.content 90 0056:0028.44:90`<br />
`EXEC`<br />
This is not always needed, but simplifies the operations of updating the index. In order to remove the old information we indexed for the object ID 90, regardless of the current fields values of the object, we just have to retrieve the hash value by object ID and `ZREM` it in the sorted set view.<br />
### **Representing and querying graphs using an hexastore**
One cool thing about composite indexes is that they are handy in order to represent graphs, using a data structure which is called Hexastore.<br />
The hexastore provides a representation for relations between objects, formed by a subject, a predicate and an object. A simple relation between objects could be:<br />
`antirez is-friend-of matteocollina`<br />
Note that I prefixed my item with the string spo. It means that the item represents a subject,predicate,object relation.<br />
Make sure to check [Matteo Collina's slides about Levelgraph](http://nodejsconfit.levelgraph.io/) in order to better understand these ideas.
### **Multi dimensional indexes**
ALEX 注：<br />
这节内容非常有趣，值得注意的地方有<br />
1) `2 bits: x between 70 and 75, y between 200 and 201 (range=2)`<br />
应为<br />
`2 bits: x between 74 and 75, y between 200 and 201 (range=2)`<br />
2) 对 "However single squares may not cover all our search" 这句话的理解 —— 左下角(50, 100)，右上角(100, 300)，range 64X64，range的起始点都是(0, 0)，所以单个方块[(0, 64)~(63, 127)]是不够的，画图理解这句话更清晰。<br />
3) Ruby example 的运行结果<br />
```
[vagrant@localhost ~]$ ruby test.rb
0,1 x from 0 to 63, y from 64 to 127
ZRANGEBYLEX myindex [000001000000000000 [000001111111111111
0,2 x from 0 to 63, y from 128 to 191
ZRANGEBYLEX myindex [000100000000000000 [000100111111111111
0,3 x from 0 to 63, y from 192 to 255
ZRANGEBYLEX myindex [000101000000000000 [000101111111111111
0,4 x from 0 to 63, y from 256 to 319
ZRANGEBYLEX myindex [010000000000000000 [010000111111111111
1,1 x from 64 to 127, y from 64 to 127
ZRANGEBYLEX myindex [000011000000000000 [000011111111111111
1,2 x from 64 to 127, y from 128 to 191
ZRANGEBYLEX myindex [000110000000000000 [000110111111111111
1,3 x from 64 to 127, y from 192 to 255
ZRANGEBYLEX myindex [000111000000000000 [000111111111111111
1,4 x from 64 to 127, y from 256 to 319
ZRANGEBYLEX myindex [010010000000000000 [010010111111111111
```
### **Non range indexes**
So far we checked indexes which are useful to query by range or by single item. However other Redis data structures such as Sets or Lists can be used in order to build other kind of indexes. They are very commonly used but maybe we don't always realize they are actually a form of indexing.<br />
For instance I can index object IDs into a Set data type in order to use the get random elements operation via `SRANDMEMBER` in order to retrieve a set of random objects. Sets can also be used to check for existence when all I need is to test if a given item exists or not or has a single boolean property or not.<br />
Similarly lists can be used in order to index items into a fixed order. I can add all my items into a Redis list and rotate the list with `RPOPLPUSH` using the same key name as source and destination. This is useful when I want to process a given set of items again and again forever in the same order. Think of an RSS feed system that needs to refresh the local copy periodically.<br />
Another popular index often used with Redis is a capped list, where items are added with `LPUSH` and trimmed with `LTRIM`, in order to create a view with just the latest N items encountered, in the same order they were seen.<br />

## **Redis 持久化**
以下内容摘自 [Redis Persistence](https://redis.io/topics/persistence)<br />
### **AOF advantages**
For instance even if you flushed everything for an error using a `FLUSHALL` command, if **no rewrite of the log was performed** in the meantime you can still save your data set just stopping the server, removing the latest command, and restarting Redis again.<br />
### **What should I do if my AOF gets corrupted?**
It is possible that the server crashes while writing the AOF file (this still should never lead to inconsistencies), corrupting the file in a way that is no longer loadable by Redis. When this happens you can fix this problem using the following procedure:<br />
- Make a backup copy of your AOF file.<br />
- Fix the original file using the redis-check-aof tool that ships with Redis:<br />
`$ redis-check-aof --fix`<br />
- Optionally use diff -u to check what is the difference between two files.<br />
- Restart the server with the fixed file.<br />
**IMPORTANT**: remember to edit your redis.conf to turn on the AOF, otherwise when you restart the server the configuration changes will be lost and the server will start again with the old configuration.<br />
### **Backing up Redis data**
Redis is very data backup friendly since you can copy RDB files while the database is running: the RDB is never modified once produced, and while it gets produced it uses a temporary name and is renamed into its final destination atomically using rename(2) only when the new snapshot is complete.<br />
This means that copying the RDB file is completely safe while the server is running. This is what we suggest:<br />
- Create a cron job in your server creating hourly snapshots of the RDB file in one directory, and daily snapshots in a different directory.<br />
- Every time the cron script runs, make sure to call the find command to make sure too old snapshots are deleted: for instance you can take hourly snapshots for the latest 48 hours, and daily snapshots for one or two months. Make sure to name the snapshots with data and time information.<br />
- At least one time every day make sure to transfer an RDB snapshot outside your data center or at least outside the physical machine running your Redis instance.<br />

## **Redis 副本集**
以下内容摘自 [Replication](https://redis.io/topics/replication)<br />
The following are some very important facts about Redis replication:<br />
- Redis uses asynchronous replication, with asynchronous slave-to-master acknowledges of the amount of data processed.<br />
- A master can have multiple slaves.<br />
- Slaves are able to accept connections from other slaves. Aside from connecting a number of slaves to the same master, slaves can also be connected to other slaves in a cascading-like structure. Since Redis 4.0, all the sub-slaves will receive exactly the same replication stream from the master.<br />
- Redis replication is non-blocking on the master side. This means that the master will continue to handle queries when one or more slaves perform the initial synchronization or a partial resynchronization.<br />
- Replication is also largely non-blocking on the slave side. While the slave is performing the initial synchronization, it can handle queries using the old version of the dataset, assuming you configured Redis to do so in redis.conf. Otherwise, you can configure Redis slaves to return an error to clients if the replication stream is down. However, after the initial sync, the old dataset must be deleted and the new one must be loaded. The slave will block incoming connections during this brief window (that can be as long as many seconds for very large datasets). Since Redis 4.0 it is possible to configure Redis so that the deletion of the old data set happens in a different thread, however loading the new initial dataset will still happen in the main thread and block the slave.<br />
- Replication can be used both for scalability, in order to have multiple slaves for read-only queries (for example, slow O(N) operations can be offloaded to slaves), or simply for data safety.<br />
- It is possible to use replication to avoid the cost of having the master write the full dataset to disk: a typical technique involves configuring your master redis.conf to avoid persisting to disk at all, then connect a slave configured to save from time to time, or with AOF enabled. However this setup must be handled with care, since a restarting master will start with an empty dataset: if the slave tries to synchronized with it, the slave will be emptied as well.<br />
### **Safety of replication when master has persistence turned off**
In setups where Redis replication is used, it is strongly advised to have persistence turned on in the master and in the slaves. When this is not possible, for example because of latency concerns due to very slow disks, instances should be configured to **avoid restarting automatically** after a reboot.<br />
Every time data safety is important, and replication is used with master configured without persistence, auto restart of instances should be disabled.<br />
### **How Redis replication works**
Every Redis master has a replication ID: it is a large pseudo random string that marks a given story of the dataset. Each master also takes an offset that increments for every byte of replication stream that it is produced to be sent to slaves, in order to update the state of the slaves with the new changes modifying the dataset. The replication offset is incremented even if no slave is actually connected, so basically every given pair of:<br />
Replication ID, offset<br />
Identifies an exact version of the dataset of a master.<br />
When slaves connects to master, they use the PSYNC command in order to send their old master replication ID and the offsets they processed so far. This way the master can send just the incremental part needed. However if there is not enough backlog in the master buffers, or if the slave is referring to an history (replication ID) which is no longer known, than a full resynchronization happens: in this case the slave will get a full copy of the dataset, from scratch.<br />
### **Configuration**
To configure basic Redis replication is trivial: just add the following line to the slave configuration file:<br />
`slaveof 192.168.1.1 6379`<br />
There are also a few parameters for tuning the replication backlog taken in memory by the master to perform the partial resynchronization. See the example redis.conf shipped with the Redis distribution for more information.<br />
Diskless replication can be enabled using the repl-diskless-sync configuration parameter. The delay to start the transfer in order to wait more slaves to arrive after the first one, is controlled by the repl-diskless-sync-delay parameter. Please refer to the example redis.conf file in the Redis distribution for more details.<br />
### **Read-only slave**
Since Redis 2.6, slaves support a read-only mode that is enabled by default. This behavior is controlled by the slave-read-only option in the redis.conf file, and can be enabled and disabled at runtime using `CONFIG SET`.<br />
### **Setting a slave to authenticate to a master**
If your master has a password via requirepass, it's trivial to configure the slave to use that password in all sync operations.<br />
To do it on a running instance, use redis-cli and type:<br />
`config set masterauth <password>`<br />
To set it permanently, add this to your config file:<br />
`masterauth <password>`<br />
### **Allow writes only with N attached replicas**
Starting with Redis 2.8, it is possible to configure a Redis master to accept write queries only if at least N slaves are currently connected to the master.<br />
This is how the feature works:<br />
- Redis slaves ping the master every second, acknowledging the amount of replication stream processed.<br />
- Redis masters will remember the last time it received a ping from every slave.<br />
- The user can configure a minimum number of slaves that have a lag not greater than a maximum number of seconds.<br />
If there are at least N slaves, with a lag less than M seconds, then the write will be accepted.<br />
There are two configuration parameters for this feature:<br />
- min-slaves-to-write \<number of slaves\><br />
- min-slaves-max-lag \<number of seconds\><br />
For more information, please check the example redis.conf file shipped with the Redis source distribution.<br />
### **The INFO and ROLE command**
There are two Redis commands that provide a lot of information on the current replication parameters of master and slave instances. One is `INFO`. If the command is called with the replication argument as INFO replication only information relevant to the replication are displayed. Another more computer-friendly command is `ROLE`, that provides the replication status of masters and slaves together with their replication offsets, list of connected slaves and so forth.<br />

## **Redis 模块简介**
以下内容摘自 [Redis Modules: an introduction to the API](https://redis.io/topics/modules-intro)<br />
Redis modules make possible to extend Redis functionality using external modules, implementing new Redis commands at a speed and with features similar to what can be done inside the core itself.<br />
Redis modules are dynamic libraries, that can be loaded into Redis at startup or using the `MODULE LOAD` command. <br />
Modules are designed in order to be loaded into different versions of Redis, so a given module does not need to be designed, or recompiled, in order to run with a specific version of Redis. For this reason, the module will register to the Redis core using a specific API version. The current API version is "1".<br />
### **Loading modules**
In order to test the module you are developing, you can load the module using the following redis.conf configuration directive:<br />
`loadmodule /path/to/mymodule.so`<br />
It is also possible to load a module at runtime using the following command:<br />
`MODULE LOAD /path/to/mymodule.so`<br />
In order to list all loaded modules, use:<br />
`MODULE LIST`<br />
Finally, you can unload (and later reload if you wish) a module using the following command:<br />
`MODULE UNLOAD mymodule`<br />
Note that mymodule above is not the filename without the .so suffix, but instead, the name the module used to register itself into the Redis core. The name can be obtained using `MODULE LIST`. However it is good practice that the filename of the dynamic library is the same as the name the module uses to register itself into the Redis core.<br />
### **Module initialization**
`RedisModule_Init()` should be the first function called by the module OnLoad function. The following is the function prototype:<br />
`int RedisModule_Init(RedisModuleCtx *ctx, const char *modulename,`<br />
                     `int module_version, int api_version);`<br />
The Init function announces the Redis core that the module has a given name, its version (that is reported by `MODULE LIST`), and that is willing to use a specific version of the API.<br />
The second function called, `RedisModule_CreateCommand`, is used in order to register commands into the Redis core. The following is the prototype:<br />
`int RedisModule_CreateCommand(RedisModuleCtx *ctx, const char *cmdname,`<br />
                              `RedisModuleCmdFunc cmdfunc);`<br />
To create a new command, the above function needs the context, the command name, and the function pointer of the function implementing the command, which must have the following prototype:<br />
`int mycommand(RedisModuleCtx *ctx, RedisModuleString **argv, int argc);`<br />
As you can see, the arguments are provided as pointers to a specific data type, the RedisModuleString. This is an opaque data type you have API functions to access and use, direct access to its fields is never needed.<br />
Zooming into the example command implementation, we can find another call:<br />
`int RedisModule_ReplyWithLongLong(RedisModuleCtx *ctx, long long integer);`<br />
This function returns an integer to the client that invoked the command, exactly like other Redis commands do, like for example `INCR` or `SCARD`.<br />
### **Setup and dependencies of a Redis module**
Redis modules don't depend on Redis or some other library, nor they need to be compiled with a specific redismodule.h file. In order to create a new module, just copy a recent version of redismodule.h in your source tree, link all the libraries you want, and create a dynamic library having the `RedisModule_OnLoad()` function symbol exported.<br />
The module will be able to load into different versions of Redis.<br />
### **Passing configuration parameters to Redis modules**
When the module is loaded with the `MODULE LOAD` command, or using the loadmodule directive in the redis.conf file, the user is able to pass configuration parameters to the module by adding arguments after the module file name:<br />
`loadmodule mymodule.so foo bar 1234`<br />
In the above example the strings foo, bar and 123 will be passed to the module `OnLoad()` function in the argv argument as an array of RedisModuleString pointers. The number of arguments passed is into argc.<br />
The way you can access those strings will be explained in the rest of this document. Normally the module will store the module configuration parameters in some static global variable that can be accessed module wide, so that the configuration can change the behavior of different commands.<br />
### **Working with RedisModuleString objects**
There are a few functions in order to work with string objects:<br />
`const char *RedisModule_StringPtrLen(RedisModuleString *string, size_t *len);`<br />
You can create new string objects using the following API:<br />
`RedisModuleString *RedisModule_CreateString(RedisModuleCtx *ctx, const char *ptr, size_t len);`<br />
The string returned by the above command must be freed using a corresponding call to `RedisModule_FreeString()`:<br />
`void RedisModule_FreeString(RedisModuleString *str);`<br />
Note that the strings provided via the argument vector argv never need to be freed. You only need to free new strings you create, or new strings returned by other APIs, where it is specified that the returned string must be freed.<br />
#### **Creating strings from numbers or parsing strings as numbers**
Creating a new string from an integer is a very common operation, so there is a function to do this:<br />
`RedisModuleString *mystr = RedisModule_CreateStringFromLongLong(ctx,10);`<br />
Similarly in order to parse a string as a number:<br />
`long long myval;`<br />
`if (RedisModule_StringToLongLong(ctx,argv[1],&myval) == REDISMODULE_OK) {`<br />
    `/* Do something with 'myval' */`<br />
`}`<br />
#### **Accessing Redis keys from modules**
Most Redis modules, in order to be useful, have to interact with the Redis data space (this is not always true, for example an ID generator may never touch Redis keys). Redis modules have two different APIs in order to access the Redis data space, one is a low level API that provides very fast access and a set of functions to manipulate Redis data structures. The other API is more high level, and allows to call Redis commands and fetch the result, similarly to how Lua scripts access Redis.<br />
The high level API is also useful in order to access Redis functionalities that are not available as APIs.<br />
In general modules developers should prefer the low level API, because commands implemented using the low level API run at a speed comparable to the speed of native Redis commands. However there are definitely use cases for the higher level API. For example often the bottleneck could be processing the data and not accessing it.<br />
Also note that sometimes using the low level API is not harder compared to the higher level one.<br />
### **Calling Redis commands**
The high level API to access Redis is the sum of the `RedisModule_Call()` function, together with the functions needed in order to access the reply object returned by `Call()`.<br />
`RedisModule_Call` uses a special calling convention, with a format specifier that is used to specify what kind of objects you are passing as arguments to the function.<br />
Redis commands are invoked just using a command name and a list of arguments. However when calling commands, the arguments may originate from different kind of strings: null-terminated C strings, RedisModuleString objects as received from the argv parameter in the command implementation, binary safe C buffers with a pointer and a length, and so forth.<br />
For example if I want to call INCRBY using a first argument (the key) a string received in the argument vector argv, which is an array of RedisModuleString object pointers, and a C string representing the number "10" as second argument (the increment), I'll use the following function call:<br />
`RedisModuleCallReply *reply;`<br />
`reply = RedisModule_Call(ctx,"INCR","sc",argv[1],"10");`<br />
The first argument is the context, and the second is always a null terminated C string with the command name. The third argument is the format specifier where each character corresponds to the type of the arguments that will follow. In the above case "sc" means a RedisModuleString object, and a null terminated C string. The other arguments are just the two arguments as specified. In fact argv[1] is a RedisModuleString and "10" is a null terminated C string.<br />
This is the full list of format specifiers:<br />
c -- Null terminated C string pointer.<br />
b -- C buffer, two arguments needed: C string pointer and size_t length.<br />
s -- RedisModuleString as received in argv or by other Redis module APIs returning a RedisModuleString object.<br />
l -- Long long integer.<br />
v -- Array of RedisModuleString objects.<br />
! -- This modifier just tells the function to replicate the command to slaves and AOF. It is ignored from the point of view of arguments parsing.<br />
The function returns a RedisModuleCallReply object on success, on error NULL is returned.<br />
#### **Working with RedisModuleCallReply objects.**
RedisModuleCall returns reply objects that can be accessed using the RedisModule_CallReply* family of functions.<br />
In order to obtain the type or reply (corresponding to one of the data types supported by the Redis protocol), the function RedisModule_CallReplyType() is used:<br />
`reply = RedisModule_Call(ctx,"INCR","sc",argv[1],"10");`<br />
`if (RedisModule_CallReplyType(reply) == REDISMODULE_REPLY_INTEGER) {`<br />
`    long long myval = RedisModule_CallReplyInteger(reply);`<br />
`    /* Do something with myval. */`<br />
`}`<br />
Valid reply types are:<br />
- REDISMODULE_REPLY_STRING Bulk string or status replies.<br />
- REDISMODULE_REPLY_ERROR Errors.<br />
- REDISMODULE_REPLY_INTEGER Signed 64 bit integers.<br />
- REDISMODULE_REPLY_ARRAY Array of replies.<br />
- REDISMODULE_REPLY_NULL NULL reply.<br />
Strings, errors and arrays have an associated length. For strings and errors the length corresponds to the length of the string. For arrays the length is the number of elements. To obtain the reply length the following function is used:<br />
`size_t reply_len = RedisModule_CallReplyLength(reply);`<br />
Sub elements of array replies are accessed this way:<br />
`RedisModuleCallReply *subreply;`<br />
`subreply = RedisModule_CallReplyArrayElement(reply,idx);`<br />
Strings and errors (which are like strings but with a different type) can be accessed using in the following way, making sure to never write to the resulting pointer (that is returned as as const pointer so that misusing must be pretty explicit):<br />
`size_t len;`<br />
`char *ptr = RedisModule_CallReplyStringPtr(reply,&len);`<br />
You can use the following function in order to create a new string object from a call reply of type string, error or integer:<br />
`RedisModuleString *mystr = RedisModule_CreateStringFromCallReply(myreply);`<br />
If the reply is not of the right type, NULL is returned. The returned string object should be released with RedisModule_FreeString() as usually, or by enabling automatic memory management (see corresponding section).<br />
### **Releasing call reply objects**
Reply objects must be freed using `RedisModule_FreeCallReply`. For arrays, you need to free only the top level reply, not the nested replies.<br />
If you use automatic memory management (explained later in this document) you don't need to free replies (but you still could if you wish to release memory ASAP).<br />
#### **Returning values from Redis commands**
All the functions to send a reply to the client are called RedisModule_ReplyWith\<something\>.<br />
To return an error, use:<br />
`RedisModule_ReplyWithError(RedisModuleCtx *ctx, const char *err);`<br />
To reply with a simple string, that can't contain binary values or newlines, (so it's suitable to send small words, like "OK") we use:<br />
`RedisModule_ReplyWithSimpleString(ctx,"OK");`<br />
It's possible to reply with "bulk strings" that are binary safe, using two different functions:<br />
`int RedisModule_ReplyWithStringBuffer(RedisModuleCtx *ctx, const char *buf, size_t len);`<br />
`int RedisModule_ReplyWithString(RedisModuleCtx *ctx, RedisModuleString *str);`<br />
In order to reply with an array, you just need to use a function to emit the array length, followed by as many calls to the above functions as the number of elements of the array are:<br />
`RedisModule_ReplyWithArray(ctx,2);`<br />
`RedisModule_ReplyWithStringBuffer(ctx,"age",3);`<br />
`RedisModule_ReplyWithLongLong(ctx,22);`<br />
To return nested arrays is easy, your nested array element just uses another call to `RedisModule_ReplyWithArray()` followed by the calls to emit the sub array elements.<br />
#### **Returning arrays with dynamic length**
Sometimes it is not possible to know beforehand the number of items of an array. This is accomplished with a special argument to `RedisModule_ReplyWithArray()`:<br />
`RedisModule_ReplyWithArray(ctx, REDISMODULE_POSTPONED_ARRAY_LEN);`<br />
The above call starts an array reply so we can use other ReplyWith calls in order to produce the array items. Finally in order to set the length we use the following call:<br />
`RedisModule_ReplySetArrayLength(ctx, number_of_items);`<br />
In the case of the FACTOR command, this translates to some code similar to this:<br />
`RedisModule_ReplyWithArray(ctx, REDISMODULE_POSTPONED_ARRAY_LEN);`<br />
`number_of_factors = 0;`<br />
`while(still_factors) {`<br />
`    RedisModule_ReplyWithLongLong(ctx, some_factor);`<br />
`    number_of_factors++;`<br />
`}`<br />
`RedisModule_ReplySetArrayLength(ctx, number_of_factors);`<br />
Another common use case for this feature is iterating over the arrays of some collection and only returning the ones passing some kind of filtering.<br />
### **Arity and type checks**
Often commands need to check that the number of arguments and type of the key is correct. In order to report a wrong arity, there is a specific function called `RedisModule_WrongArity()`. The usage is trivial:<br />
`if (argc != 2) return RedisModule_WrongArity(ctx);`<br />
Checking for the wrong type involves opening the key and checking the type:<br />
`RedisModuleKey *key = RedisModule_OpenKey(ctx,argv[1],`<br />
`    REDISMODULE_READ|REDISMODULE_WRITE);`<br />

`int keytype = RedisModule_KeyType(key);`<br />
`if (keytype != REDISMODULE_KEYTYPE_STRING &&`<br />
`    keytype != REDISMODULE_KEYTYPE_EMPTY)`<br />
`{`<br />
`    RedisModule_CloseKey(key);`<br />
`    return RedisModule_ReplyWithError(ctx,REDISMODULE_ERRORMSG_WRONGTYPE);`<br />
`}`<br />
Note that you often want to proceed with a command both if the key is of the expected type, or if it's empty.<br />
#### **Low level access to keys**
Low level access to keys allow to perform operations on value objects associated to keys directly, with a speed similar to what Redis uses internally to implement the built-in commands.<br />
Once a key is opened, a key pointer is returned that will be used with all the other low level API calls in order to perform operations on the key or its associated value.<br />
Because the API is meant to be very fast, it cannot do too many run-time checks, so the user must be aware of certain rules to follow:<br />
- Opening the same key multiple times where at least one instance is opened for writing, is undefined and may lead to crashes.<br />
- While a key is open, it should only be accessed via the low level key API. For example opening a key, then calling DEL on the same key using the RedisModule_Call() API will result into a crash. However it is safe to open a key, perform some operation with the low level API, closing it, then using other APIs to manage the same key, and later opening it again to do some more work.<br />
In order to open a key the RedisModule_OpenKey function is used. It returns a key pointer, that we'll use with all the next calls to access and modify the value:<br />
`RedisModuleKey *key;`<br />
`key = RedisModule_OpenKey(ctx,argv[1],REDISMODULE_READ);`<br />
The second argument is the key name, that must be a RedisModuleString object. The third argument is the mode: REDISMODULE_READ or REDISMODULE_WRITE. It is possible to use | to bitwise OR the two modes to open the key in both modes. <br />
You can open non exisitng keys for writing, since the keys will be created when an attempt to write to the key is performed. However when opening keys just for reading, RedisModule_OpenKey will return NULL if the key does not exist.<br />
Once you are done using a key, you can close it with:<br />
`RedisModule_CloseKey(key);`<br />
Note that if automatic memory management is enabled, you are not forced to close keys. When the module function returns, Redis will take care to close all the keys which are still open.<br />
#### **Getting the key type**
In order to obtain the value of a key, use the RedisModule_KeyType() function:<br />
`int keytype = RedisModule_KeyType(key);`<br />
It returns one of the following values:<br />
`REDISMODULE_KEYTYPE_EMPTY`<br />
`REDISMODULE_KEYTYPE_STRING`<br />
`REDISMODULE_KEYTYPE_LIST`<br />
`REDISMODULE_KEYTYPE_HASH`<br />
`REDISMODULE_KEYTYPE_SET`<br />
`REDISMODULE_KEYTYPE_ZSET`<br />
#### **Creating new keys**
To create a new key, open it for writing and then write to it using one of the key writing functions. Example:<br />
`RedisModuleKey *key;`<br />
`key = RedisModule_OpenKey(ctx,argv[1],REDISMODULE_READ);`<br />
`if (RedisModule_KeyType(key) == REDISMODULE_KEYTYPE_EMPTY) {`<br />
`    RedisModule_StringSet(key,argv[2]);`<br />
`}`<br />
#### **Deleting keys**
Just use:<br />
`RedisModule_DeleteKey(key);`<br />
#### **Managing key expires (TTLs)**
To control key expires two functions are provided, that are able to set, modify, get, and unset the time to live associated with a key.<br />
One function is used in order to query the current expire of an open key:<br />
`mstime_t RedisModule_GetExpire(RedisModuleKey *key);`<br />
The function returns the time to live of the key in milliseconds, or REDISMODULE_NO_EXPIRE as a special value to signal the key has no associated expire or does not exist at all (you can differentiate the two cases checking if the key type is REDISMODULE_KEYTYPE_EMPTY).<br />
In order to change the expire of a key the following function is used instead:<br />
`int RedisModule_SetExpire(RedisModuleKey *key, mstime_t expire);`<br />
#### **Obtaining the length of values**
There is a single function in order to retrieve the length of the value associated to an open key. The returned length is value-specific, and is the string length for strings, and the number of elements for the aggregated data types (how many elements there is in a list, set, sorted set, hash).<br />
`size_t len = RedisModule_ValueLength(key);`<br />
If the key does not exist, 0 is returned by the function.<br />
#### **String type API**
Setting a new string value, like the Redis SET command does, is performed using:<br />
`int RedisModule_StringSet(RedisModuleKey *key, RedisModuleString *str);`<br />
Accessing existing string values is performed using DMA (direct memory access) for speed. The API will return a pointer and a length, so that's possible to access and, if needed, modify the string directly.<br />
`size_t len, j;`<br />
`char *myptr = RedisModule_StringDMA(key,&len,REDISMODULE_WRITE);`<br />
`for (j = 0; j < len; j++) myptr[j] = 'A';`<br />
Sometimes when we want to manipulate strings directly, we need to change their size as well. For this scope, the RedisModule_StringTruncate function is used. Example:<br />
`RedisModule_StringTruncate(mykey,1024);`<br />
The function truncates, or enlarges the string as needed, padding it with zero bytes if the previos length is smaller than the new length we request. If the string does not exist since key is associated to an open empty key, a string value is created and associated to the key.<br />
Note that every time StringTruncate() is called, we need to re-obtain the DMA pointer again, since the old may be invalid.<br />
#### **List type API**
It's possible to push and pop values from list values:<br />
`int RedisModule_ListPush(RedisModuleKey *key, int where, RedisModuleString *ele);`<br />
`RedisModuleString *RedisModule_ListPop(RedisModuleKey *key, int where);`<br />
In both the APIs the where argument specifies if to push or pop from tail or head, using the following macros:<br />
`REDISMODULE_LIST_HEAD`<br />
`REDISMODULE_LIST_TAIL`<br />
Elements returned by RedisModule_ListPop() are like strings created with RedisModule_CreateString(), they must be released with RedisModule_FreeString() or by enabling automatic memory management.<br />
#### **Set type API**
Work in progress.<br />
#### **Sorted set type API**
Documentation missing, please refer to the top comments inside module.c for the following functions:<br />
- RedisModule_ZsetAdd<br />
- RedisModule_ZsetIncrby<br />
- RedisModule_ZsetScore<br />
- RedisModule_ZsetRem<br />
And for the sorted set iterator:<br />
- RedisModule_ZsetRangeStop<br />
- RedisModule_ZsetFirstInScoreRange<br />
- RedisModule_ZsetLastInScoreRange<br />
- RedisModule_ZsetFirstInLexRange<br />
- RedisModule_ZsetLastInLexRange<br />
- RedisModule_ZsetRangeCurrentElement<br />
- RedisModule_ZsetRangeNext<br />
- RedisModule_ZsetRangePrev<br />
- RedisModule_ZsetRangeEndReached<br />
#### **Hash type API**
Documentation missing, please refer to the top comments inside module.c for the following functions:<br />
- RedisModule_HashSet<br />
- RedisModule_HashGet<br />
#### **Iterating aggregated values**
Work in progress.<br />
### **Replicating commands**
When using the higher level APIs to invoke commands, replication happens automatically if you use the "!" modifier in the format string of `RedisModule_Call()` as in the following example:<br />
`reply = RedisModule_Call(ctx,"INCR","!sc",argv[1],"10");`<br />
As you can see the format specifier is "!sc". The bang is not parsed as a format specifier, but it internally flags the command as "must replicate".<br />
If you use the above programming style, there are no problems. However sometimes things are more complex than that, and you use the low level API. In this case, if there are no side effects in the command execution, and it consistently always performs the same work, what is possible to do is to replicate the command verbatim as the user executed it. To do that, you just need to call the following function:<br />
`RedisModule_ReplicateVerbatim(ctx);`<br />
When you use the above API, you should not use any other replication function since they are not guaranteed to mix well.<br />
However this is not the only option. It's also possible to exactly tell Redis what commands to replicate as the effect of the command execution, using an API similar to RedisModule_Call() but that instead of calling the command sends it to the AOF / slaves stream. Example:<br />
`RedisModule_Replicate(ctx,"INCRBY","cl","foo",my_increment);`<br />
It's possible to call RedisModule_Replicate multiple times, and each will emit a command. All the sequence emitted is wrapped between a MULTI/EXEC transaction, so that the AOF and replication effects are the same as executing a single command.<br />
Note that `Call()` replication and Replicate() replication have a rule, in case you want to mix both forms of replication (not necessarily a good idea if there are simpler approaches). Commands replicated with `Call()` are always the first emitted in the final `MULTI/EXEC` block, while all the commands emitted with `Replicate()` will follow.<br />
### **Automatic memory management**
Normally when writing programs in the C language, programmers need to manage memory manually. This is why the Redis modules API has functions to release strings, close open keys, free replies, and so forth.<br />
However given that commands are executed in a contained environment and with a set of strict APIs, Redis is able to provide automatic memory management to modules, at the cost of some performance (most of the time, a very low cost).<br />
When automatic memory management is enabled:<br />
1. You don't need to close open keys.<br />
2. You don't need to free replies.<br />
3. You don't need to free RedisModuleString objects.<br />
However you can still do it, if you want. For example, automatic memory management may be active, but inside a loop allocating a lot of strings, you may still want to free strings no longer used.<br />
In order to enable automatic memory management, just call the following function at the start of the command implementation:<br />
`RedisModule_AutoMemory(ctx);`<br />
Automatic memory management is usually the way to go, however experienced C programmers may not use it in order to gain some speed and memory usage benefit.<br />
### **Allocating memory into modules**
Normal C programs use malloc() and free() in order to allocate and release memory dynamically. While in Redis modules the use of malloc is not technically forbidden, it is a lot better to use the Redis Modules specific functions, that are exact replacements for malloc, free, realloc and strdup. These functions are:<br />
`void *RedisModule_Alloc(size_t bytes);`<br />
`void* RedisModule_Realloc(void *ptr, size_t bytes);`<br />
`void RedisModule_Free(void *ptr);`<br />
`void RedisModule_Calloc(size_t nmemb, size_t size);`<br />
`char *RedisModule_Strdup(const char *str);`<br />
#### **Pool allocator**
Sometimes in commands implementations, it is required to perform many small allocations that will be not retained at the end of the command execution, but are just functional to execute the command itself.<br />
This work can be more easily accomplished using the Redis pool allocator:<br />
`void *RedisModule_PoolAlloc(RedisModuleCtx *ctx, size_t bytes);`<br />
### **Writing commands compatible with Redis Cluster**
Documentation missing, please check the following functions inside module.c:<br />
`RedisModule_IsKeysPositionRequest(ctx);`<br />
`RedisModule_KeyAtPos(ctx,pos);`<br />

## **Redis 高可用性**
以下内容摘自 [Redis Sentinel Documentation](https://redis.io/topics/sentinel)<br />
Redis Sentinel provides high availability for Redis.<br />
Redis Sentinel also provides other collateral tasks such as monitoring, notifications and acts as a configuration provider for clients.<br />
This is the full list of Sentinel capabilities at a macroscopical level (i.e. the big picture):<br />
- Monitoring. Sentinel constantly checks if your master and slave instances are working as expected.<br />
- Notification. Sentinel can notify the system administrator, another computer programs, via an API, that something is wrong with one of the monitored Redis instances.<br />
- Automatic failover. If a master is not working as expected, Sentinel can start a failover process where a slave is promoted to master, the other additional slaves are reconfigured to use the new master, and the applications using the Redis server informed about the new address to use when connecting.<br />
- Configuration provider. Sentinel acts as a source of authority for clients service discovery: clients connect to Sentinels in order to ask for the address of the current Redis master responsible for a given service. If a failover occurs, Sentinels will report the new address.<br />
### **Quick Start**
#### **Obtaining Sentinel**
The current version of Sentinel is called Sentinel 2. It is a rewrite of the initial Sentinel implementation using stronger and simpler to predict algorithms.<br />
A stable release of Redis Sentinel is shipped since Redis 2.8.<br />
Redis Sentinel version 1, shipped with Redis 2.6, is deprecated and should not be used.<br />
#### **Running Sentinel**
If you are using the redis-sentinel executable (or if you have a symbolic link with that name to the redis-server executable) you can run Sentinel with the following command line:<br />
`redis-sentinel /path/to/sentinel.conf`<br />
Otherwise you can use directly the redis-server executable starting it in Sentinel mode:<br />
`redis-server /path/to/sentinel.conf --sentinel`<br />
Both ways work the same.<br />
However **it is mandatory** to use a configuration file when running Sentinel, as this file will be used by the system in order to save the current state that will be reloaded in case of restarts. Sentinel will simply refuse to start if no configuration file is given or if the configuration file path is not writable.<br />
Sentinels by default run **listening for connections to TCP port 26379**, so for Sentinels to work, port 26379 of your servers **must be open** to receive connections from the IP addresses of the other Sentinel instances. Otherwise Sentinels can't talk and can't agree about what to do, so failover will never be performed.<br />
#### **Fundamental things to know about Sentinel before deploying**
1. You need at least three Sentinel instances for a robust deployment.<br />
2. The three Sentinel instances should be placed into computers or virtual machines that are believed to fail in an independent way. So for example different physical servers or Virtual Machines executed on different availability zones.<br />
3. Sentinel + Redis distributed system does not guarantee that acknowledged writes are retained during failures, since Redis uses asynchronous replication. However there are ways to deploy Sentinel that make the window to lose writes limited to certain moments, while there are other less secure ways to deploy it.<br />
4. You need Sentinel support in your clients. Popular client libraries have Sentinel support, but not all.<br />
5. There is no HA setup which is safe if you don't test from time to time in development environments, or even better if you can, in production environments, if they work. You may have a misconfiguration that will become apparent only when it's too late (at 3am when your master stops working).<br />
6. Sentinel, Docker, or other forms of Network Address Translation or Port Mapping should be mixed with care: Docker performs port remapping, breaking Sentinel auto discovery of other Sentinel processes and the list of slaves for a master. Check the section about Sentinel and Docker later in this document for more information.<br />
#### **Configuring Sentinel**
The Redis source distribution contains a file called sentinel.conf that is a self-documented example configuration file you can use to configure Sentinel, however a typical minimal configuration file looks like the following:<br />
```
sentinel monitor mymaster 127.0.0.1 6379 2
sentinel down-after-milliseconds mymaster 60000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1

sentinel monitor resque 192.168.1.3 6380 4
sentinel down-after-milliseconds resque 10000
sentinel failover-timeout resque 180000
sentinel parallel-syncs resque 5
```
The example configuration above, basically monitor two sets of Redis instances, each composed of a master and an undefined number of slaves. One set of instances is called mymaster, and the other resque.<br />
The meaning of the arguments of sentinel monitor statements is the following:<br />
`sentinel monitor <master-group-name> <ip> <port> <quorum>`<br />
- The quorum is the number of Sentinels that need to agree about the fact the master is not reachable, in order for really mark the slave as failing, and eventually start a fail over procedure if possible.<br />
- However the quorum is only used to detect the failure. In order to actually perform a failover, one of the Sentinels need to be elected leader for the failover and be authorized to proceed. This only happens with the vote of the majority of the Sentinel processes.<br />
#### **Other Sentinel options**
The other options are almost always in the form:<br />
`sentinel <option_name> <master_name> <option_value>`<br />
And are used for the following purposes:<br />
- down-after-milliseconds is the time in milliseconds an instance should not be reachable (either does not reply to our PINGs or it is replying with an error) for a Sentinel starting to think it is down.<br />
- parallel-syncs sets the number of slaves that can be reconfigured to use the new master after a failover at the same time. The lower the number, the more time it will take for the failover process to complete, however if the slaves are configured to serve old data, you may not want all the slaves to re-synchronize with the master at the same time. While the replication process is mostly non blocking for a slave, there is a moment when it stops to load the bulk data from the master. You may want to make sure only one slave at a time is not reachable by setting this option to the value of 1.<br />
All the configuration parameters can be modified at runtime using the `SENTINEL SET` command.<br />
#### **Example Sentinel deployments**
Note that we will never show setups where just two Sentinels are used, since Sentinels always need to talk with the majority in order to start a failover.<br />
So please **deploy at least three Sentinels in three different boxes** always.<br />

## **Redis 集群**
### **Redis 集群教程**
以下内容摘自 [Redis cluster tutorial](https://redis.io/topics/cluster-tutorial)<br />
#### **Redis Cluster 101**
Redis Cluster provides a way to run a Redis installation where data is automatically sharded across multiple Redis nodes.<br />
So in practical terms, what you get with Redis Cluster?<br />
- The ability to automatically split your dataset among multiple nodes.<br />
- The ability to continue operations when a subset of the nodes are experiencing failures or are unable to communicate with the rest of the cluster.<br />
#### **Redis Cluster TCP ports**
Every Redis Cluster node requires two TCP connections open. The normal Redis TCP port used to serve clients, for example 6379, plus the port obtained by adding 10000 to the data port, so 16379 in the example.<br />
This second high port is used for the Cluster bus, that is a node-to-node communication channel using a binary protocol. The Cluster bus is used by nodes for failure detection, configuration update, failover authorization and so forth. Clients should never try to communicate with the cluster bus port, but always with the normal Redis command port, however make sure you open both ports in your firewall, otherwise Redis cluster nodes will be not able to communicate.<br />
The command port and cluster bus port offset is fixed and is always 10000.<br />
Note that for a Redis Cluster to work properly you need, for each node:<br />
1. The normal client communication port (usually 6379) used to communicate with clients to be open to all the clients that need to reach the cluster, plus all the other cluster nodes (that use the client port for keys migrations).<br />
2. The cluster bus port (the client port + 10000) must be reachable from all the other cluster nodes.<br />
The cluster bus uses a different, binary protocol, for node to node data exchange, which is more suited to exchange information between nodes using little bandwidth and processing time.<br />
#### **Redis Cluster and Docker**
Currently Redis Cluster does not support NATted environments and in general environments where IP addresses or TCP ports are remapped.<br />
In order to make Docker compatible with Redis Cluster you need to use the host networking mode of Docker. Please check the --net=host option in the Docker documentation for more information.<br />
#### **Redis Cluster data sharding**
Redis Cluster does not use consistent hashing, but a different form of sharding where every key is conceptually part of what we call an hash slot.<br />
There are 16384 hash slots in Redis Cluster, and to compute what is the hash slot of a given key, we simply take the CRC16 of the key modulo 16384.<br />
Every node in a Redis Cluster is responsible for a subset of the hash slots, so for example you may have a cluster with 3 nodes, where:<br />
- Node A contains hash slots from 0 to 5500.<br />
- Node B contains hash slots from 5501 to 11000.<br />
- Node C contains hash slots from 11001 to 16383.<br />
This allows to add and remove nodes in the cluster easily. For example if I want to add a new node D, I need to move some hash slot from nodes A, B, C to D. Similarly if I want to remove node A from the cluster I can just move the hash slots served by A to B and C. When the node A will be empty I can remove it from the cluster completely.<br />
Because moving hash slots from a node to another does not require to stop operations, adding and removing nodes, or changing the percentage of hash slots hold by nodes, does not require any downtime.<br />
Redis Cluster supports multiple key operations as long as all the keys involved into a single command execution (or whole transaction, or Lua script execution) all belong to the same hash slot. The user can force multiple keys to be part of the same hash slot by using a concept called hash tags.<br />
#### **Redis Cluster master-slave model**
In order to remain available when a subset of master nodes are failing or are not able to communicate with the majority of nodes, Redis Cluster uses a master-slave model where every hash slot has from 1 (the master itself) to N replicas (N-1 additional slaves nodes).<br />
#### **Redis Cluster consistency guarantees**
Redis Cluster is not able to guarantee strong consistency. In practical terms this means that under certain conditions it is possible that Redis Cluster will lose writes that were acknowledged by the system to the client.<br />
The first reason why Redis Cluster can lose writes is because it uses asynchronous replication. This means that during writes the following happens:<br />
- Your client writes to the master B.<br />
- The master B replies OK to your client.<br />
- The master B propagates the write to its slaves B1, B2 and B3.<br />
As you can see B does not wait for an acknowledge from B1, B2, B3 before replying to the client, since this would be a prohibitive latency penalty for Redis, so if your client writes something, B acknowledges the write, but crashes before being able to send the write to its slaves, one of the slaves (that did not receive the write) can be promoted to master, losing the write forever.<br />
There is another notable scenario where Redis Cluster will lose writes, that happens during a network partition where a client is isolated with a minority of instances including at least a master.<br />
Take as an example our 6 nodes cluster composed of A, B, C, A1, B1, C1, with 3 masters and 3 slaves. There is also a client, that we will call Z1.<br />
After a partition occurs, it is possible that in one side of the partition we have A, C, A1, B1, C1, and in the other side we have B and Z1.<br />
Z1 is still able to write to B, that will accept its writes. If the partition heals in a very short time, the cluster will continue normally. However if the partition lasts enough time for B1 to be promoted to master in the majority side of the partition, the writes that Z1 is sending to B will be lost.<br />
Note that there is a maximum window to the amount of writes Z1 will be able to send to B: if enough time has elapsed for the majority side of the partition to elect a slave as master, every master node in the minority side stops accepting writes.<br />
This amount of time is a very important configuration directive of Redis Cluster, and is called the node timeout.<br />
After node timeout has elapsed, a master node is considered to be failing, and can be replaced by one of its replicas. Similarly after node timeout has elapsed without a master node to be able to sense the majority of the other master nodes, it enters an error state and stops accepting writes.<br />
### **Redis Cluster configuration parameters**
We are about to create an example cluster deployment. Before we continue, let's introduce the configuration parameters that Redis Cluster introduces in the redis.conf file. Some will be obvious, others will be more clear as you continue reading.<br />
- cluster-enabled \<yes/no\>: If yes enables Redis Cluster support in a specific Redis instance. Otherwise the instance starts as a stand alone instance as usually.<br />
- cluster-config-file \<filename\>: Note that despite the name of this option, this is not an user editable configuration file, but the file where a Redis Cluster node automatically persists the cluster configuration (the state, basically) every time there is a change, in order to be able to re-read it at startup. The file lists things like the other nodes in the cluster, their state, persistent variables, and so forth. Often this file is rewritten and flushed on disk as a result of some message reception.<br />
- cluster-node-timeout \<milliseconds\>: The maximum amount of time a Redis Cluster node can be unavailable, without it being considered as failing. If a master node is not reachable for more than the specified amount of time, it will be failed over by its slaves. This parameter controls other important things in Redis Cluster. Notably, every node that can't reach the majority of master nodes for the specified amount of time, will stop accepting queries.<br />
- cluster-slave-validity-factor <factor>: If set to zero, a slave will always try to failover a master, regardless of the amount of time the link between the master and the slave remained disconnected. If the value is positive, a maximum disconnection time is calculated as the node timeout value multiplied by the factor provided with this option, and if the node is a slave, it will not try to start a failover if the master link was disconnected for more than the specified amount of time. For example if the node timeout is set to 5 seconds, and the validity factor is set to 10, a slave disconnected from the master for more than 50 seconds will not try to failover its master. Note that any value different than zero may result in Redis Cluster to be unavailable after a master failure if there is no slave able to failover it. In that case the cluster will return back available only when the original master rejoins the cluster.<br />
- cluster-migration-barrier \<count\>: Minimum number of slaves a master will remain connected with, for another slave to migrate to a master which is no longer covered by any slave. See the appropriate section about replica migration in this tutorial for more information.<br />
- cluster-require-full-coverage \<yes/no\>: If this is set to yes, as it is by default, the cluster stops accepting writes if some percentage of the key space is not covered by any node. If the option is set to no, the cluster will still serve queries even if only requests about a subset of keys can be processed.<br />
### **Creating and using a Redis Cluster**
To create a cluster, the first thing we need is to have a few empty Redis instances running in cluster mode. This basically means that clusters are not created using normal Redis instances as a special mode needs to be configured so that the Redis instance will enable the Cluster specific features and commands.<br />
The following is a minimal Redis cluster configuration file:<br />
```
port 7000
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
appendonly yes
```
Note that the minimal cluster that works as expected requires to contain at least three master nodes. For your first tests it is strongly suggested to start a six nodes cluster with three masters and three slaves.<br />
To do so, enter a new directory, and create the following directories named after the port number of the instance we'll run inside any given directory.<br />
Something like:<br />
```
mkdir cluster-test
cd cluster-test
mkdir 7000 7001 7002 7003 7004 7005
```
Create a redis.conf file inside each of the directories, from 7000 to 7005. As a template for your configuration file just use the small example above, but make sure to replace the port number 7000 with the right port number according to the directory name.<br />
Now copy your redis-server executable, compiled from the latest sources in the unstable branch at GitHub, into the cluster-test directory, and finally open 6 terminal tabs in your favorite terminal application.<br />
Start every instance like that, one every tab:<br />
```
cd 7000
../redis-server ./redis.conf
```
#### **Creating the cluster**
Now that we have a number of instances running, we need to create our cluster by writing some meaningful configuration to the nodes.<br />
This is very easy to accomplish as we are helped by the Redis Cluster command line utility called redis-trib, a Ruby program executing special commands on instances in order to create new clusters, check or reshard an existing cluster, and so forth.<br />
The redis-trib utility is in the src directory of the Redis source code distribution. You need to install redis gem to be able to run redis-trib.<br />
`gem install redis`<br />
To create your cluster simply type:<br />
`./redis-trib.rb create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 \`<br />
`127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005`<br />
```
[vagrant@localhost redis-cluster-test]$ ./redis-trib.rb create --replicas 1 127.0.0.1:7000 127
.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
127.0.0.1:7000
127.0.0.1:7001
127.0.0.1:7002
Adding replica 127.0.0.1:7003 to 127.0.0.1:7000
Adding replica 127.0.0.1:7004 to 127.0.0.1:7001
Adding replica 127.0.0.1:7005 to 127.0.0.1:7002
M: 6fb6ba6d5ebdcc3db98aeec07e7e98f60e5316bf 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
M: 4d547aeb6ff2ff25253d13757b641e1ba85adea6 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
M: c03739ed66d09a8f39618b9766291e4e1cffd01a 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
S: 7d40acbc9e5bde6bf7e8f6cb30e0305c945c54e7 127.0.0.1:7003
   replicates 6fb6ba6d5ebdcc3db98aeec07e7e98f60e5316bf
S: 46e1dfd41403a924e3881ab0f5f679ecdc7952b8 127.0.0.1:7004
   replicates 4d547aeb6ff2ff25253d13757b641e1ba85adea6
S: 0d985dab118bb822df2a7498f88e5d00a144e11d 127.0.0.1:7005
   replicates c03739ed66d09a8f39618b9766291e4e1cffd01a
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join...
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: 6fb6ba6d5ebdcc3db98aeec07e7e98f60e5316bf 127.0.0.1:7000
   slots:0-5460 (5461 slots) master
   1 additional replica(s)
M: c03739ed66d09a8f39618b9766291e4e1cffd01a 127.0.0.1:7002
   slots:10923-16383 (5461 slots) master
   1 additional replica(s)
S: 7d40acbc9e5bde6bf7e8f6cb30e0305c945c54e7 127.0.0.1:7003
   slots: (0 slots) slave
   replicates 6fb6ba6d5ebdcc3db98aeec07e7e98f60e5316bf
S: 0d985dab118bb822df2a7498f88e5d00a144e11d 127.0.0.1:7005
   slots: (0 slots) slave
   replicates c03739ed66d09a8f39618b9766291e4e1cffd01a
M: 4d547aeb6ff2ff25253d13757b641e1ba85adea6 127.0.0.1:7001
   slots:5461-10922 (5462 slots) master
   1 additional replica(s)
S: 46e1dfd41403a924e3881ab0f5f679ecdc7952b8 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 4d547aeb6ff2ff25253d13757b641e1ba85adea6
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
The command used here is create, since we want to create a new cluster. The option --replicas 1 means that we want a slave for every master created. The other arguments are the list of addresses of the instances I want to use to create the new cluster.<br />
Obviously the only setup with our requirements is to create a cluster with 3 masters and 3 slaves.<br />
Redis-trib will propose you a configuration. Accept the proposed configuration by typing yes. The cluster will be configured and joined, which means, instances will be bootstrapped into talking with each other. Finally, if everything went well, you'll see a message like that:<br />
`[OK] All 16384 slots covered`<br />
This means that there is at least a master instance serving each of the 16384 slots available.<br />
#### **Creating a Redis Cluster using the create-cluster script**
If you don't want to create a Redis Cluster by configuring and executing individual instances manually as explained above, there is a much simpler system (but you'll not learn the same amount of operational details).<br />
Just check utils/create-cluster directory in the Redis distribution. There is a script called create-cluster inside (same name as the directory it is contained into), it's a simple bash script. In order to start a 6 nodes cluster with 3 masters and 3 slaves just type the following commands:<br />
1. create-cluster start<br />
2. create-cluster create<br />
Reply to yes in step 2 when the redis-trib utility wants you to accept the cluster layout.<br />
You can now interact with the cluster, the first node will start at port 30001 by default. When you are done, stop the cluster with:<br />
1. create-cluster stop.<br />
Please read the README inside this directory for more information on how to run the script.<br />
ALEX 注：<br />
create-cluster clean-logs<br />
create-cluster clean<br />
#### **Playing with the cluster**
The following is an example of interaction using the redis-cli command line utility:
```
$ redis-cli -c -p 7000
redis 127.0.0.1:7000> set foo bar
-> Redirected to slot [12182] located at 127.0.0.1:7002
OK
redis 127.0.0.1:7002> set hello world
-> Redirected to slot [866] located at 127.0.0.1:7000
OK
redis 127.0.0.1:7000> get foo
-> Redirected to slot [12182] located at 127.0.0.1:7002
"bar"
redis 127.0.0.1:7000> get hello
-> Redirected to slot [866] located at 127.0.0.1:7000
"world"
```
#### **Resharding the cluster**
Resharding basically means to move hash slots from a set of nodes to another set of nodes, and like cluster creation it is accomplished using the redis-trib utility.<br />
To start a resharding just type:<br />
`./redis-trib.rb reshard 127.0.0.1:7000`<br />
I can always find the ID of a node with the following command if I need:<br />
```
$ redis-cli -p 7000 cluster nodes | grep myself
97a3a64667477371c4479320d683e4c8db5858b1 :0 myself,master - 0 0 0 connected 0-5460
```
At the end of the resharding, you can test the health of the cluster with the following command:<br />
`./redis-trib.rb check 127.0.0.1:7000`<br />
#### **Scripting a resharding operation**
Reshardings can be performed automatically without the need to manually enter the parameters in an interactive way. This is possible using a command line like the following:<br />
`./redis-trib.rb reshard --from <node-id> --to <node-id> --slots <number of slots> --yes <host>:<port>`<br />
#### **Testing the failover**
Let's crash node 7002 with the `DEBUG SEGFAULT` command:<br />
```
$ redis-cli -p 7002 debug segfault
Error: Server closed the connection
```
We can check what is the cluster setup.<br />
```
$ redis-cli -p 7000 cluster nodes
3fc783611028b1707fd65345e763befb36454d73 127.0.0.1:7004 slave 3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 0 1385503418521 0 connected
a211e242fc6b22a9427fed61285e85892fa04e08 127.0.0.1:7003 slave 97a3a64667477371c4479320d683e4c8db5858b1 0 1385503419023 0 connected
97a3a64667477371c4479320d683e4c8db5858b1 :0 myself,master - 0 0 0 connected 0-5959 10922-11422
3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 127.0.0.1:7005 master - 0 1385503419023 3 connected 11423-16383
3e3a6cb0d9a9a87168e266b0a0b24026c0aae3f0 127.0.0.1:7001 master - 0 1385503417005 0 connected 5960-10921
2938205e12de373867bf38f1ca29d31d0ddb3e46 127.0.0.1:7002 slave 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 0 1385503418016 3 connected
```
The output of the CLUSTER NODES command may look intimidating, but it is actually pretty simple, and is composed of the following tokens:<br />
- Node ID<br />
- ip:port<br />
- flags: master, slave, myself, fail, ...<br />
- if it is a slave, the Node ID of the master<br />
- Time of the last pending PING still waiting for a reply.<br />
- Time of the last PONG received.<br />
- Configuration epoch for this node (see the Cluster specification).<br />
- Status of the link to this node.<br />
- Slots served...<br />
#### **Manual failover**
Manual failovers are supported by Redis Cluster using the `CLUSTER FAILOVER` command, that must be executed in one of the slaves of the master you want to failover.<br />
#### **Adding a new node**
Adding a new node is basically the process of adding an empty node and then moving some data into it, in case it is a new master, or telling it to setup as a replica of a known node, in case it is a slave.<br />
In both cases the first step to perform is adding an empty node.<br />
Now we can use redis-trib as usually in order to add the node to the existing cluster.<br />
`./redis-trib.rb add-node 127.0.0.1:7006 127.0.0.1:7000`<br />
Now it is possible to assign hash slots to this node using the resharding feature of redis-trib. It is basically useless to show this as we already did in a previous section, there is no difference, it is just a resharding having as a target the empty node.<br />
#### **Adding a new node as a replica**
Adding a new Replica can be performed in two ways. The obvious one is to use redis-trib again, but with the --slave option, like this:<br />
`./redis-trib.rb add-node --slave 127.0.0.1:7006 127.0.0.1:7000`<br />
However you can specify exactly what master you want to target with your new replica with the following command line:<br />
`./redis-trib.rb add-node --slave --master-id 3c3a0c74aae0b56170ccb03a76b60cfe7dc1912e 127.0.0.1:7006 127.0.0.1:7`<br />
A more manual way to add a replica to a specific master is to add the new node as an empty master, and then turn it into a replica using the `CLUSTER REPLICATE` command. This also works if the node was added as a slave but you want to move it as a replica of a different master.<br />
#### **Removing a node**
To remove a slave node just use the del-node command of redis-trib:<br />
``./redis-trib del-node 127.0.0.1:7000 `<node-id>` ``<br />
The first argument is just a random node in the cluster, the second argument is the ID of the node you want to remove.<br />
You can remove a master node in the same way as well, however in order to remove a master node it must be empty. If the master is not empty you need to reshard data away from it to all the other master nodes before.<br />
An alternative to remove a master node is to perform a manual failover of it over one of its slaves and remove the node after it turned into a slave of the new master. Obviously this does not help when you want to reduce the actual number of masters in your cluster, in that case, a resharding is needed.<br />
#### **Replicas migration**
In Redis Cluster it is possible to reconfigure a slave to replicate with a different master at any time just using the following command:<br />
`CLUSTER REPLICATE <master-node-id>`<br />
However there is a special scenario where you want replicas to move from one master to another one automatically, without the help of the system administrator. The automatic reconfiguration of replicas is called replicas migration and is able to improve the reliability of a Redis Cluster.<br />
The reason why you may want to let your cluster replicas to move from one master to another under certain condition, is that usually the Redis Cluster is as resistant to failures as the number of replicas attached to a given master.<br />
For example a cluster where every master has a single replica can't continue operations if the master and its replica fail at the same time, simply because there is no other instance to have a copy of the hash slots the master was serving. However while netsplits are likely to isolate a number of nodes at the same time, many other kind of failures, like hardware or software failures local to a single node, are a very notable class of failures that are unlikely to happen at the same time, so it is possible that in your cluster where every master has a slave, the slave is killed at 4am, and the master is killed at 6am. This still will result in a cluster that can no longer operate.<br />
To improve reliability of the system we have the option to add additional replicas to every master, but this is expensive. Replica migration allows to add more slaves to just a few masters. So you have 10 masters with 1 slave each, for a total of 20 instances. However you add, for example, 3 instances more as slaves of some of your masters, so certain masters will have more than a single slave.<br />
With replicas migration what happens is that if a master is left without slaves, a replica from a master that has multiple slaves will migrate to the orphaned master. So after your slave goes down at 4am as in the example we made above, another slave will take its place, and when the master will fail as well at 5am, there is still a slave that can be elected so that the cluster can continue to operate.<br />
So what you should know about replicas migration in short?<br />
- The cluster will try to migrate a replica from the master that has the greatest number of replicas in a given moment.<br />
- To benefit from replica migration you have just to add a few more replicas to a single master in your cluster, it does not matter what master.<br />
- There is a configuration parameter that controls the replica migration feature that is called cluster-migration-barrier: you can read more about it in the example redis.conf file provided with Redis Cluster.<br />
#### **Upgrading nodes in a Redis Cluster**
Upgrading slave nodes is easy since you just need to stop the node and restart it with an updated version of Redis. If there are clients scaling reads using slave nodes, they should be able to reconnect to a different slave if a given one is not available.<br />
Upgrading masters is a bit more complex, and the suggested procedure is:<br />
1. Use CLUSTER FAILOVER to trigger a manual failover of the master to one of its slaves (see the "Manual failover" section of this documentation).<br />
2. Wait for the master to turn into a slave.<br />
3. Finally upgrade the node as you do for slaves.<br />
4. If you want the master to be the node you just upgraded, trigger a new manual failover in order to turn back the upgraded node into a master.<br />
Following this procedure you should upgrade one node after the other until all the nodes are upgraded.<br />
#### **Migrating to Redis Cluster**
Users willing to migrate to Redis Cluster may have just a single master, or may already using a preexisting sharding setup, where keys are split among N nodes, using some in-house algorithm or a sharding algorithm implemented by their client library or Redis proxy.<br />
In both cases it is possible to migrate to Redis Cluster easily, however what is the most important detail is if multiple-keys operations are used by the application, and how. There are three different cases:<br />
1. Multiple keys operations, or transactions, or Lua scripts involving multiple keys, are not used. Keys are accessed independently (even if accessed via transactions or Lua scripts grouping multiple commands, about the same key, together).<br />
2. Multiple keys operations, transactions, or Lua scripts involving multiple keys are used but only with keys having the same hash tag, which means that the keys used together all have a {...} sub-string that happens to be identical. For example the following multiple keys operation is defined in the context of the same hash tag: SUNION {user:1000}.foo {user:1000}.bar.<br />
3. Multiple keys operations, transactions, or Lua scripts involving multiple keys are used with key names not having an explicit, or the same, hash tag.<br />
The third case is not handled by Redis Cluster: the application requires to be modified in order to don't use multi keys operations or only use them in the context of the same hash tag.<br />
Case 1 and 2 are covered, so we'll focus on those two cases, that are handled in the same way, so no distinction will be made in the documentation.<br />
Assuming you have your preexisting data set split into N masters, where N=1 if you have no preexisting sharding, the following steps are needed in order to migrate your data set to Redis Cluster:<br />
1. Stop your clients. No automatic live-migration to Redis Cluster is currently possible. You may be able to do it orchestrating a live migration in the context of your application / environment.<br />
2. Generate an append only file for all of your N masters using the BGREWRITEAOF command, and waiting for the AOF file to be completely generated.<br />
3. Save your AOF files from aof-1 to aof-N somewhere. At this point you can stop your old instances if you wish (this is useful since in non-virtualized deployments you often need to reuse the same computers).<br />
4. Create a Redis Cluster composed of N masters and zero slaves. You'll add slaves later. Make sure all your nodes are using the append only file for persistence.<br />
5. Stop all the cluster nodes, substitute their append only file with your pre-existing append only files, aof-1 for the first node, aof-2 for the second node, up to aof-N.<br />
6. Restart your Redis Cluster nodes with the new AOF files. They'll complain that there are keys that should not be there according to their configuration.<br />
7. Use redis-trib fix command in order to fix the cluster so that keys will be migrated according to the hash slots each node is authoritative or not.<br />
8. Use redis-trib check at the end to make sure your cluster is ok.<br />
9. Restart your clients modified to use a Redis Cluster aware client library.<br />
There is an alternative way to import data from external instances to a Redis Cluster, which is to use the redis-trib import command.<br />
