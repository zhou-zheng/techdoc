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
问题原因：在README 有这么一段话：
> Allocator  
> --------- 
> Selecting a non-default memory allocator when building Redis is done by setting  
> the \`MALLOC\` environment variable. Redis is compiled and linked against libc  
> malloc by default, with the exception of jemalloc being the default on Linux  
> systems. This default was picked because jemalloc has proven to have fewer  
> fragmentation problems than libc malloc.  
>  
> To force compiling against libc malloc, use:  
>  
> &emsp;&emsp;% make MALLOC=libc  
>  
> To compile against jemalloc on Mac OS X systems, use:  
>  
> &emsp;&emsp;% make MALLOC=jemalloc

意思是说关于分配器 allocator， 如果有 MALLOC 这个环境变量，会有用这个环境变量去建立Redis。
而且 libc 并不是默认的分配器，默认的是 jemalloc, 因为 jemalloc 被证明比 libc 有更少的fragmentation problems。但是如果你没有 jemalloc 而只有 libc，当然 make 出错。<br />
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
用 SET 随机填充一百万个 key：<br />
` $ redis-benchmark -t set -n 1000000 -r 100000000 `<br />
用 redis-cli 查看内存占用情况：<br />
` INFO memory `<br />










 two on-disk storage formats (RDB and AOF) 