---
title: Redis
date: '2018-10-28T23:02:00.000Z'
updated: '2019-01-15T15:30:40.000Z'
tags:
  - Java
  - Redis
---

# Redis

## 安装

### （1）Mac HomeBrew

```text
$ brew install redis
```

### （2）手动编译

```text
# 需要前置依赖 make、gcc
$ sudo apt-get update
$ sudo apt install make
$ sudo apt-get install gcc

# 下载、解压和编译 redis
$ wget http://download.redis.io/releases/redis-5.0.3.tar.gz
$ tar xzf redis-5.0.3.tar.gz
$ cd redis-5.0.3
$ make
```

### （3）安装到 CentOS 中

```text
sudo yum install epel-release yum-utils
sudo yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm
sudo yum-config-manager --enable remi

sudo yum install redis

sudo systemctl start redis
sudo systemctl enable redis

sudo systemctl status redis

vim /etc/redis.conf
# 注释掉 bind 一行，使得外部可访问

sudo systemctl restart redis

ps aux | grep redis
```

## 命令行使用

### 启动 Redis Server

```text
$./src/redis-server
```

### 进入 Redis 命令行

（1）本地连接

```text
$redis-cli
redis 127.0.0.1:6379>
redis 127.0.0.1:6379> PING

PONG
```

（2）远程连接

```text
$ redis-cli -h host -p port -a password
```

### 取键相关命令

```text
# 查找所有 key
> KEYS *
# 查找符合给定正则的 key
> KEYS pattern

# 删除某个 key (适用于各种数据结构的 key)
> DEL a_key
# 检查某个 key 是否存在
> EXISTS a_key
# 以秒为单位，返回给定 key 的剩余生存时间(TTL, time to live)。
> TTL a_key
# 获知某个 key 存储值的类型
> TYPE a_key

# 获取Size，可能会把失效的也计算在内
> DBSIZE
```

### 字符串取值相关命令

```text
# 设置指定 key 的值
> SET a_key a_value
# 获取指定 key 的字符串值
> GET a_key
```

> 注意：对有标点符号的 key，要用双引号（“”）包裹，否则会返回 `nil` 。比如使用 Spring Cache + Redis 时，会序列化缓存方法返回值，这是的 KEY 就要用双引号括起来，示例如下，
>
> `GET "cache1:\xac\xed\x00\x05t\x00$cb5775e6-1b39-4f63-85c8-13f134a54f32"`

### List 相关命令

```text
# 获知列表长度
> Llen a_key
# 获取列表指定范围内的元素。其中 0 表示列表的第一个元素， 1 表示列表的第二个元素，以此类推。 你也可以使用负数下标，以 -1 表示列表的最后一个元素， -2 表示列表的倒数第二个元素，以此类推。
> Lrange a_key start end

# 对一个列表进行修剪(trim)，就是说，让列表只保留指定区间内的元素，不在指定区间之内的元素都将被删除。
> Ltrim a_key start end

# 移除并返回列表的最后一个元素。
> Lpop a_key
```

### Set 相关命令

```text
# 返回集合中的所有成员
> smembers a_key
# 添加成员到集合中
> sadd a_key a_member
# 获取集合中的成员数
> scard a_key
# 移除集合中的一个或多个成员
> srem a_key a_member b_member
```

### 删库跑路相关命令

```text
# 删除所有数据库的所有key
> FLUSHALL
# 删除当前数据库的所有key
> FLUSHDB
```

## Jedis 使用

### 快速开始

（1）引入依赖

```groovy
compile "redis.clients:jedis:3.0.0"
```

（2）简单使用

```java
Jedis jedis = new Jedis("localhost");
jedis.set("foo", "bar");
String value = jedis.get("foo");
```

（3）使用 Hash

Redis hash 是一个string类型的field和value的映射表，hash特别适合用于存储对象。Redis 中每个 hash 可以存储 2^32 - 1 键值对（40多亿）。

```java
// 设置映射表 key 中的字段 field 的值为 value
jedis.hset(key, field, value);
// 获取映射表 key 中的字段 field 的值
jedis.hget(key, field);
// 删除映射表 key 中的字段 field
jedis.hdel(key, field);
```

（4）使用 Set

Redis 的 Set 是 String 类型的无序集合。集合成员是唯一的，这就意味着集合中不能出现重复的数据。Redis 中集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O\(1\)。集合中最大的成员数为 2^32 - 1 \(4294967295, 每个集合可存储40多亿个成员\)。

```java
// 向集合 key 中添加成员 member1 和 member2
jedis.sadd(key, member1, member2);
// 获取集合 key 中的所有成员
jedis.smembers(key);
```

（5）使用 Set 的交并补

```java
// 交集
jedis.sinter(key1, key2);
// 交集并存储到新的 Set
jedis.sinterstore(destination, key1, key2);
// 并集
jedis.sunion(key1, key2);
// 并集并存储到新的 Set
jedis.sunionstore(destination, key1, key2);
// 补集
jedis.sdiff(key1, key2);
// 补集并存储到新的 Set
jedis.sdiffstore(destination, key1, key2);
```

## 结构化查询

合理设计 Redis 存储可以达到类似 SQL 的结构化查询的效果。具体可参见：[像查询DB一样查询redis - JQ棣](https://blog.csdn.net/w13528476101/article/details/70146064)。

思路是这样的，先确定整个数据存储的唯一键，以以下格式存储完整数据：

* key：data:\[表名\]:\[主键\]
* value：json 字符串

当需要全表查询时，我们第一个想到的通常是 scan，但 scan 的效率不高。我们可以通过构建主键索引集合来解决，如下：

* key：idx:\[表名\]
* value：\[主键集合\]

当需要进行条件查询时，我们可以构建条件索引集合来解决，如下：

* key：idx:\[表名\]:\[字段名\]:\[字段值\]
* value：\[主键集合\]

当需要进行多个条件筛选查询时，我们可以使用 Redis Set 的交并补集功能。

## 查看 Redis 信息和状态

```text
$ redis-cli
$ redis 127.0.0.1:6379>info
# 查看已连接客户端的信息
$ redis 127.0.0.1:6379>info clients
# 查看服务器的内存信息
$ redis 127.0.0.1:6379>info memory
# 查看 CPU 的计算量统计信息
$ redis 127.0.0.1:6379>info cpu
# 查看跟 RDB 持久化和 AOF 持久化有关的信息
$ redis 127.0.0.1:6379>info persistence
# 查看一般统计信息
$ redis 127.0.0.1:6379>info stats
# 查看主/从复制信息
$ redis 127.0.0.1:6379>info replication
# 查看各种不同类型的命令的执行统计信息
$ redis 127.0.0.1:6379>info commandstats
# 查看和集群有关的信息
$ redis 127.0.0.1:6379>info cluster
# 查看数据库相关的统计信息
$ redis 127.0.0.1:6379>info keyspace
```

## 分析占用内存较大的 key

使用以下命令会生成一个报表（非阻塞式），描述占用内存较大的 key

```text
$ redis-cli -h host -p port -a password  --bigkeys
```

报表格式如下：

```text
# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Biggest string found so far 'jobflow_detail:9bc01d5d-b413-49c0-a08c-8cde2d04c7f1' with 36016 bytes
[00.00%] Biggest set    found so far 'url_set:pro29`98404f7f-e1ad-42f6-96bc-cbeb6a8b1086`ae_job_product' with 50 members
[00.00%] Biggest hash   found so far 'url:pro29`bc03f85a-c51e-45bf-9009-81d6cd021dd0`ae_job_product_detail' with 49 fields
[00.01%] Biggest string found so far 'jobflow_detail:d184d3b1-3ac9-4261-b803-2bb8574f8888' with 36557 bytes
[00.01%] Biggest set    found so far 'url_set:pro18`6cb2ec83-6ac0-4972-8a41-1360e834312b`ae_job_product_detail_mobile' with 95 members
[00.01%] Biggest hash   found so far 'url:pro29`319c3190-77b1-4ff8-9c50-9757d29fc3aa`ae_job_product' with 50 fields
[00.01%] Biggest list   found so far 'queue:pro29`a57f547d-bd77-4ae8-9d43-56827dee5c27`ae_job_product_feedback' with 49 items
[00.16%] Biggest hash   found so far 'url:pro18`64985a16-2591-41c4-8e23-a3b792fd360c`ae_job_product_feedback' with 98 fields
[00.20%] Biggest set    found so far 'url_set:pro18`ac4c49d2-5be4-4b90-92f5-151af1ff60f7`ae_job_product_wishes' with 99 members
[00.21%] Biggest list   found so far 'queue:pro20`fafd935a-24a3-47bc-badc-78445a5668bc`ae_job_product_detail_mobile' with 92 items
[00.33%] Biggest string found so far 'jobflow_detail:19e11854-5164-474f-a019-55e2629e681f' with 61682 bytes
[00.35%] Biggest set    found so far 'url_set:pro18`e18a889e-e34e-44b1-b105-554777872635`ae_job_product_wishes' with 100 members
[00.56%] Biggest list   found so far 'queue:pro20`933f6e17-ac03-4b76-889b-8ae637969402`ae_job_product' with 101 items
[00.59%] Biggest string found so far 'jobflow_detail:5ea576de-3115-49e5-9d5e-26c0840e5be9' with 62154 bytes
[00.78%] Biggest hash   found so far 'url:pro20`6e37acf7-5c16-48f5-a401-dc78e32ea1c9`ae_job_product' with 100 fields
[03.07%] Biggest string found so far 'jobflow_detail:7a25da13-1355-4434-b77c-dcc9b223c5c7' with 62569 bytes
[05.05%] Biggest hash   found so far 'url:shopee25`1c0bc59a-7d99-4282-9a62-9a9f52030aa2`shopee_job_product' with 650 fields
[05.77%] Biggest hash   found so far 'url:pro36:1554090684190:ae_job_product_detail_mobile' with 438725 fields
[11.43%] Biggest list   found so far 'queue:shopee25`1c0bc59a-7d99-4282-9a62-9a9f52030aa2`shopee_job_product_list' with 1351 items
[12.06%] Biggest set    found so far 'url_set:pro20`5acdb283-f1e1-41df-8bc7-d9743fd4b67d`ae_job_product' with 101 members
[12.87%] Biggest set    found so far 'url_set:shopee24`55ee626f-0730-4667-8b3b-bef873db2d9b`shopee_job_product' with 500 members
[13.21%] Biggest string found so far 'jobflow_detail:f41e19b0-43e1-4cb2-b69f-b6f351a03e8f' with 62672 bytes
[22.39%] Biggest set    found so far 'url_set:pro27`07102f9f-f146-4745-b338-33622f9785b6`shopee_job_product' with 2000 members
[28.73%] Biggest string found so far 'jobflow_detail:4285b1b0-f8be-45b1-b0d5-348e1c481a27' with 62690 bytes
[39.48%] Biggest set    found so far 'url_set:shopee27`196cf600-e029-428d-befc-09c0a3b2021b`shopee_job_product' with 64581 members
[44.82%] Biggest list   found so far 'queue:pro27`07102f9f-f146-4745-b338-33622f9785b6`shopee_job_product' with 1863 items
[44.96%] Biggest set    found so far 'url_set:pro36:1554090684190:ae_job_product_wishes' with 438725 members
[74.38%] Biggest list   found so far 'queue:shopee27`196cf600-e029-428d-befc-09c0a3b2021b`shopee_job_product_comment' with 9053 items

-------- summary -------

Sampled 89371 keys in the keyspace!
Total key length in bytes is 5801117 (avg len 64.91)

Biggest string found 'jobflow_detail:4285b1b0-f8be-45b1-b0d5-348e1c481a27' has 62690 bytes
Biggest   list found 'queue:shopee27`196cf600-e029-428d-befc-09c0a3b2021b`shopee_job_product_comment' has 9053 items
Biggest    set found 'url_set:pro36:1554090684190:ae_job_product_wishes' has 438725 members
Biggest   hash found 'url:pro36:1554090684190:ae_job_product_detail_mobile' has 438725 fields

16862 strings with 298218397 bytes (18.87% of keys, avg size 17685.83)
8286 lists with 407130 items (09.27% of keys, avg size 49.13)
36026 sets with 3206394 members (40.31% of keys, avg size 89.00)
28197 hashs with 3198477 fields (31.55% of keys, avg size 113.43)
0 zsets with 0 members (00.00% of keys, avg size 0.00)
0 streams with 0 entries (00.00% of keys, avg size 0.00)
```

## 批量删除指定键模板（key pattern）的 key

```text
> redis-cli -h localhost -p 6379 KEYS abc* | xargs redis-cli -h localhost -p 6379 DEL
```

## 排错

### 编译安装时出现：jemalloc/jemalloc.h: No such file or directory

```text
make MALLOC=libc
```

### 编译安装时出现：cc adlist.o /bin/sh:1:cc:not found

缺少 gcc 环境，安装 gcc 即可。

### 服务器连接 Redis 失败

（1）背景

服务器部署 Redis ，代码部署在另一主机，调用 Redis 时发生以下异常：

```text
Caused by: redis.clients.jedis.exceptions.JedisDataException: DENIED Redis is running in protected mode because protected mode is enabled, no bind address was specified, no authentication password is requested to clients. In this mode connections are only accepted from the loopback interface. If you want to connect from external computers to Redis you may adopt one of the following solutions: 
1) Just disable protected mode sending the command 'CONFIG SET protected-mode no' from the loopback interface by connecting to Redis from the same host the server is running, however MAKE SURE Redis is not publicly accessible from internet if you do so. Use CONFIG REWRITE to make this change permanent.
2) Alternatively you can just disable the protected mode by editing the Redis configuration file, and setting the protected mode option to 'no', and then restarting the server.
3) If you started the server manually just for testing, restart it with the '--protected-mode no' option. 
4) Setup a bind address or an authentication password. NOTE: You only need to do one of the above things in order for the server to start accepting connections from the outside.
```

（2）原因

查看 `redis.conf` 发现 `bind 127.0.0.1` ，意味着只有部署 Redis 的机器 可以正常访问 Redis。

还有 Redis 运行在保护模式下。

（3）解决方案

方案一：配置 bind 地址为内网的 IP 地址，修改 `protected-mode no` ，然后重启 Redis。

## 参考

1. [Download - redis.io](https://redis.io/download)
2. [Redis 命令参考 - redis.net](http://www.redis.net.cn/order/)

