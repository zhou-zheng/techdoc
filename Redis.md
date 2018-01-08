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

以下内容摘自 [Redis FAQ](https://redis.io/topics/faq)
- > **What's the Redis memory footprint?**<br />
To give you a few examples (all obtained using 64-bit instances):<br />
&emsp;&emsp; · An empty instance uses ~ 3MB of memory.<br />
&emsp;&emsp; · 1 Million small Keys -> String Value pairs use ~ 85MB of memory.<br />
&emsp;&emsp; · 1 Million Keys -> Hash value, representing an object with 5 fields, use ~ 160 MB of memory.<br />
To test your use case is trivial using the redis-benchmark utility to generate random data sets and check with the INFO memory command the space used.<br />
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
请参考 [这篇文章](https://stackoverflow.com/questions/20673131/can-someone-explain-redis-setbit-command)<br />
0x7f 解析如下<br />
位置：0 1 2 3 4 5 6 7<br />
取位：0 1 1 1 1 1 1 1<br />
&emsp;&emsp;&emsp;MSB&emsp;&emsp;&emsp;&emsp;LSB
- > **Use hashes when possible**<br />
Small hashes are encoded in a very small space, so you should try representing your data using hashes every time it is possible. For instance if you have objects representing users in a web application, instead of using different keys for name, surname, email, password, use a single hash with all the required fields.


 two on-disk storage formats (RDB and AOF) 