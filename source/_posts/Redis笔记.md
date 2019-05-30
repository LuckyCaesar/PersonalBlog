title: Redis笔记
date: 2019-05-30
tag:
 - Redis

photos:
 - /img/RedisImg/redis.png

---

## 1. NoSQL数据库
### 数据模型 - 聚合模型
* **K-V键值对**
* **BSON，Binary存储的类JSON格式的数据**
* **列簇，列式存储数据**
* **图形**

### 四大分类
* **K-V键值对：redis等**
* **文档型数据库（bson格式比较多）：MongoDB等**
* **列存储数据库：HBase，分布式文件系统等**
* **图关系数据库：复杂的关系图谱，Neo4J，InfoGrid等**


## 2. 入门概述
### Redis：<u>Re</u>mote <u>Di</u>ctionary <u>S</u>erver

### 特点：
* **Redis支持数据的持久化。**
* **key-value、list、set、zset、hash等数据结构。**
* **支持数据备份，即master-slave模式的数据备份。**

### 场景：
* **内存存储和持久化。**
* **取最新N个数据。**
* **分布式session，分布式锁。**
* **发布、订阅消息系统。**
* **定时器、计数器。**


## 3. 数据类型
* **String：是Redis最基本的数据类型，key-value。二进制安全的，可以包含任何数据，比如jpg图片或者序列化的对象。一个redis字符串value最多支持512M。**
* **Hash：键值对集合，类似Map<String, Object>。是string类型的field和value的映射表，适合于存储对象。**
* **List：简单的字符串列表，按照插入顺序排序，底层是一个链表。**
* **Set：string类型的无序不重复集合，是通过HashTable实现的。**
* **Zset（sorted set）：string类型的无序不重复集合。不同的是每个元素会关联一个double类型的分数（score），redis正是通过分数来为集合中的成员进行从小到大的排序。Zset的成员是唯一的，但分数（score）却可以重复。**

## 4. 常用命令

[常用命令参考](http://redisdoc.com/)

## 5. 配置文件

![Img](/img/RedisImg/1.png)
![Img](/img/RedisImg/2.png)
![Img](/img/RedisImg/3.png)
![Img](/img/RedisImg/4.png)
![Img](/img/RedisImg/5.png)

## 6. 持久化

### RDB：Redis Database。在指定的时间间隔内，将内存中的数据集快照写入磁盘，而恢复时将快照文件直接读到内存中。
* **Redis会单独创建（fork）一个子进程来进行持久化，会先将数据写入到一个临时文件中，待持久化过程都结束了，再用这个临时文件替换上次持久化好的文件。整个过程中，主进程是不进行任何IO的，这就确保了极高的性能。如果需要大规模数据的恢复，且对于数据恢复的完整性不是非常敏感，那RDB方式要比AOF方式更加的高效。RDB的缺点是最后一次持久化后的数据可能丢失。**
- **fork：Fork的作用是复制一个与当前进程一样的进程，新进程的所有数据（变量、环境变量、程序计数器等）都和原进程一致，但是是一个全新的进程，并作为原进程的子进程。**
* **RDB保存的是dump.rdb文件。**
- **配置位置：配置文件里面的 SNAPSHOTTING。**
> **以下是触发RDB的配置：**
![Img](/img/RedisImg/6.png)
**默认配置：1分钟内修改了1万次，或者5分钟内修改了10次，或者15分钟内修改了1次。**
**可以直接使用 save 或者 bgsave 命令直接持久化。
save：阻塞，强制立刻持久化；bgsave：后台异步进行持久化。**

* **如何恢复：将备份文件dump.rdb移动到redis安装目录并启动服务即可。
<u>直接FLUSHALL或者SHUTDOWN的时候，会覆盖dump.rdb文件，导致这个文件内容也被清除掉。所以对于dump.rdb文件也要及时备份，保证丢失的数据能被恢复。</u>**
- **优势：适合大规模的数据备份和恢复；对数据完整性和一致性要求不高。**
* **劣势：可能有数据丢失，如服务down调等等；Fork子进程的时候，内存的占用也需要考虑。**
- **停止备份：redis-cli config set save ""**
* **修复.rdb文件的命令：redis-check-dump --fix dump.rdb**

![Img](/img/RedisImg/7.png)

### AOF：Append Only File。以日志的形式来记录每个写操作。

* **将Redis执行过的所有写指令记录下来（读操作不记录），只许追加文件不可以改写文件，Redis启动的时候会读取该文件重新构建数据，换言之，Redis重启的话就根据日志文件的内容将写指令从前到后执行一次以完成数据的恢复工作。**
- **配置位置：配置文件里的 APPEND ONLY MODE。默认保存的文件appendonly.aof。**
* **.aof和.rdb文件可以同时存在，Redis重启的时候优先加载.aof文件。**
- **策略 Appendfsync：Always，同步持久化，每次发生数据变更会被立即记录到磁盘，性能较差但数据完整性比较好。Everysec：出厂默认推荐，异步操作，隔一秒记录，若一秒内宕机，会有数据丢失。No：不记录。**
* **修复.aof文件的命令：redis-check-aof --fix appendonly.aof**
- **Rewrite：**
> - **AOF采用文件追加方式，文件会越来越大。为避免出现此种情况，新增了重写机制，当AOF文件的大小超过所设定的阈值时，Redis会启动AOF文件的内容压缩，只保留可以恢复数据的最小指令集，可以使用bgrewriteaof命令触发。**
> - **重写原理：aof文件持续增长而过大时，会fork出一个新的进程来将文件重写（也是先写临时文件最后再rename替换），遍历新进程的内存中的数据，每条记录有一条set语句。重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件，这个和快照有区别。**
> - **触发机制：Redis会记录上次重写时的AOF大小，默认配置是当AOF文件大小是上次rewrite后大小的一倍且文件大于64M时触发。**

* **优势：几个同步策略：每秒同步；每次修改同步；不同步。数据比较完整。**
- **劣势：对相同数据集的数据而言aof文件要远大于rdb文件，恢复速度慢于rdb；aof运行效率低于rdb，每秒同步策略效率较好。**

![Img](/img/RedisImg/9.png)

### 总结

![Img](/img/RedisImg/11.png)
![Img](/img/RedisImg/12.png)

## 7. 事务

### 定义：可以一次执行多个命令，本质是一组命令的集合。一个事务的所有命令都会序列化，按顺序地串行化执行而不会被其它命令插入，不许加塞。在一个队列中一次性、顺序性、排他性的执行一系列命令。

![Img](/img/RedisImg/13.png)
![Img](/img/RedisImg/14.png)

## 8. 消息订阅/发布模式

![Img](/img/RedisImg/15.png)

## 9. 主从复制

### 从库使用<slaveof 主库IP 主库端口>命令来配置。每次与master断开之后，都需要重新连接，除非你配置进redis.conf文件。info replication 命令查看当前Redis的角色、信息等等。

### 一主多从：master读写皆可，slaver只能读。master挂掉后，不会重新选举。slaver挂掉，需要要重新执行<slaveof 主库IP 主库端口>命令。
### 上一个slaver可以是下一个slaver的master，salver同样可以接收其他slavers的连接和同步请求，那么该slaver作为了链条中下一个slaver的master，可以有效减轻master的压力。中途变更转向：会清楚之前的数据，重新建立拷贝最新的。
##### slaveof no one 命令，使当前slaver停止与其他数据库的同步，转换为master。

![Img](/img/RedisImg/16.png)

## 10. 哨兵模式

### 能够从后台监控主机是否故障，如果故障了根据投票数自动将从库转换为主库。

![Img](/img/RedisImg/17.png)
### 原来的master恢复，则会变成新master的slaver。