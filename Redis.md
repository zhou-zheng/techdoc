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
- 用 SET 随机填充一百万个 key：<br />
` $ redis-benchmark -t set -n 1000000 -r 100000000 `

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
### **Expire**
- > **Pattern: Navigation session**<br />
Imagine you have a web service and you are interested in the latest N pages recently visited by your users, such that each adjacent page view was not performed more than 60 seconds after the previous. Conceptually you may consider this set of page views as a Navigation session of your user, that may contain interesting information about what kind of products he or she is looking for currently, so that you can recommend related products.<br />
You can easily model this pattern in Redis using the following strategy: every time the user does a page view you call the following commands:<br />
`MULTI`<br />
`RPUSH pagewviews.user:<userid> http://.....`<br />
`EXPIRE pagewviews.user:<userid> 60`<br />
`EXEC`<br />
If the user will be idle more than 60 seconds, the key will be deleted and only subsequent page views that have less than 60 seconds of difference will be recorded.<br />
This pattern is easily modified to use counters using INCR instead of lists using `RPUSH`.<br />
- > **Expire accuracy**<br />
In Redis 2.4 the expire might not be pin-point accurate, and it could be between zero to one seconds out.<br />
Since Redis 2.6 the expire error is from 0 to 1 milliseconds.<br />
- > **Expires and persistence**<br />
Keys expiring information is stored as absolute Unix timestamps (in milliseconds in case of Redis version 2.6 or greater). This means that the time is flowing even when the Redis instance is not active.<br />
For expires to work well, the computer time must be taken stable. If you move an RDB file from two computers with a big desync in their clocks, funny things may happen (like all the keys loaded to be expired at loading time).<br />
Even running instances will always check the computer clock, so for instance if you set a key with a time to live of 1000 seconds, and then set your computer time 2000 seconds in the future, the key will be expired immediately, instead of lasting for 1000 seconds.<br />
- > **How Redis expires keys**<br />
Redis keys are expired in two ways: a passive way, and an active way.<br />
A key is passively expired simply when some client tries to access it, and the key is found to be timed out.<br />
Of course this is not enough as there are expired keys that will never be accessed again. These keys should be expired anyway, so periodically Redis tests a few keys at random among keys with an expire set. All the keys that are already expired are deleted from the keyspace.<br />
Specifically this is what Redis does 10 times per second:<br />
&emsp;&emsp; · Test 20 random keys from the set of keys with an associated expire.<br />
&emsp;&emsp; · Delete all the keys found expired.<br />
&emsp;&emsp; · If more than 25% of keys were expired, start again from step 1.<br />
- > **How expires are handled in the replication link and AOF file**<br />
In order to obtain a correct behavior without sacrificing consistency, when a key expires, a `DEL` operation is synthesized in both the AOF file and gains all the attached slaves. This way the expiration process is centralized in the master instance, and there is no chance of consistency errors.<br />
However while the slaves connected to a master will not expire keys independently (but will wait for the `DEL` coming from the master), they'll still take the full state of the expires existing in the dataset, so when a slave is elected to a master it will be able to expire the keys independently, fully acting as a master.<br />

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