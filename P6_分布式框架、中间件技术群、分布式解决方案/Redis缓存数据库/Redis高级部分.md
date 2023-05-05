# 管道

http://www.redis.cn/topics/pipelining.html

一次请求/响应服务器能实现处理新的请求即使旧的请求还未被响应。这样就可以将*多个命令*发送到服务器，而不用等待回复，最后在一个步骤中读取该答复。

这就是管道（pipelining），是一种几十年来广泛使用的技术。例如许多POP3协议已经实现支持这个功能，大大加快了从服务器下载新邮件的过程。

Redis很早就支持管道（pipelining）技术，因此无论你运行的是什么版本，你都可以使用管道（pipelining）操作Redis。下面是一个使用的例子：

```shell
# 使用nc命令，没有就安装下
yum install nc

# 新打开一个终端
[root@manager-node vagrant]# nc localhost 6379
keys *
*0
set k1 hello
+OK

# 使用redis客户端看下k1的值
127.0.0.1:6379> get k1
"hello"
127.0.0.1:6379>

# 管道，批量发送
[root@manager-node vagrant]# echo -e "set k2 99\nincr k2\nget k2" | nc localhost 6379
+OK
:100
$3
100
```

# redis的发布和订阅

http://www.redis.cn/topics/pubsub.html

```shell
# 查看帮助
127.0.0.1:6379> help @pubsub

# 先打开一个终端1，推送一个消息
127.0.0.1:6379> PUBLISH ooxx hello
(integer) 0

# 再打开一个终端2，订阅消，此时发现没有我们刚才发送的hello消息
# 这时因为redis的发布和订阅是实时的
# 必须订阅client先启动，推送的消息才能接收到
127.0.0.1:6379> SUBSCRIBE ooxx
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "ooxx"
3) (integer) 1
```

消息分为实时性的和历史性

历史性

- 查看最近3天
- 查看更老的数据

全量的消息一定存储再DB里面的

最新3天的数据可以存储再redis的sorted_set类型中，使用时间做score

```shell
# 可以通过这个remove命令，来移除掉3天之外的消息，只保存3天的message
ZREMRANGEBYRANK key start stop
  summary: Remove all members in a sorted set within the given indexes
  since: 2.0.0
```

![image-20230402150006878](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/redis发布订阅消息处理.png)

![image-20230402150055627](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/发布订阅消息处理2.png)

# Redis事务

redis的事务不像MySQL那么完善

我们使用redis主要是因为它的快

http://www.redis.cn/topics/transactions.html

> 重要的我三个命令：
>
> WATCH：监控key的变化
>
> MULTI：开启失去
>
> EXEC：执行事务
>
> DISCARD：测试事务

![image-20230402152033528](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/redis事务.png)

redis是单进程，所有的事务都会排队依次执行，谁先执行exec命令，哪个事务就先执行

```shell
127.0.0.1:6379> help @transactions

# 事务1的终端
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> get k1
QUEUED
127.0.0.1:6379>

# 事务2的终端
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> DEL k1
QUEUED

# 如果事务2先执行，那么事务1执行后就获取不到k1的值

# WATCH 监听

# 事务1的终端
127.0.0.1:6379> WATCH k1  # 开始监控k1
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> get k1
QUEUED
127.0.0.1:6379> keys *
QUEUED
127.0.0.1:6379> EXEC
(nil)

# 事务2的终端，事务2先执行
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> del k1
QUEUED
127.0.0.1:6379> set k1 bbbb
QUEUED
127.0.0.1:6379> EXEC
1) (integer) 1
2) OK

# 事务2先执行，此时再执行事务1，事务1不会获取到k1的值。因为WATCH监听到k1的值变化了，后面事务的操作就不会执行了
```

Q：为什么 Redis 不支持回滚（roll back）？

A：如果你有使用关系式数据库的经验， 那么 “Redis 在事务失败时不进行回滚，而是继续执行余下的命令”这种做法可能会让你觉得有点奇怪。

以下是这种做法的优点：

- Redis 命令只会因为错误的语法而失败（并且这些问题不能在入队时发现），或是命令用在了错误类型的键上面：这也就是说，从实用性的角度来说，失败的命令是由编程错误造成的，而这些错误应该在开发的过程中被发现，而不应该出现在生产环境中。
- 因为不需要对回滚进行支持，所以 Redis 的内部可以保持简单且快速。

有种观点认为 Redis 处理事务的做法会产生 bug ， 然而需要注意的是， 在通常情况下， 回滚并不能解决编程错误带来的问题。 举个例子， 如果你本来想通过 [INCR](http://www.redis.cn/commands/incr.html) 命令将键的值加上 1 ， 却不小心加上了 2 ， 又或者对错误类型的键执行了 [INCR](http://www.redis.cn/commands/incr.html) ， 回滚是没有办法处理这些情况的。

# 布置实例

[Modules | Redis](https://redis.io/resources/modules/)

找到redisBloom：https://github.com/RedisBloom/RedisBloom

```shell
cd ~
cd soft
mkdir bf
cd bf
wget https://github.com/RedisLabsModules/rebloom/archive/v2.2.6.tar.gz
tar -zxvf v2.2.6.tar.gz
cd RedisBloom-2.2.6/
make   # 编译之后，当前目录下有个 redisbloom.so 文件，linux中的插件是以.so结尾的文件
cp redisbloom.so /opt/liufei/redis5/
servcie redis_6379 stop   # 停止redis
redis-server --loadmodule /opt/liufei/redis5/redisbloom.so  # 启动redis，这里使用绝对路径

# 此时再连接redis，就会有BF相关的命令了
[root@manager-node RedisBloom-2.2.6]# redis-cli
127.0.0.1:6379> BF.ADD key ...options...


127.0.0.1:6379> BF.ADD newFilter foo
(integer) 1
127.0.0.1:6379> BF.EXISTS newFilter foo
(integer) 1
127.0.0.1:6379> BF.EXISTS newFilter bar
(integer) 0
```

**布隆过滤器可以解决缓存穿透问题**

![image-20230402171410553](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/布隆过滤器.png)



![image-20230402171521773](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/过滤器插件.png)

**过滤器不止一种。**

# 缓存LRU

http://www.redis.cn/topics/lru-cache.html

![image-20230402213626597](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/redis作为数据库和缓存的区别.png)

redis作为缓存，只保留热数据。redis里的数据怎么能随着业务变化，只保留热数据，因为内存大小式有限的，也就是瓶颈

**key的有效期**

- 会随着访问延长？不会
- 发生写的时候，会剔除过期时间
- 倒计时，且，redis不能延长
- 定时
- 业务逻辑自己补全

```shell
# 1. 会随着访问延长？不会
127.0.0.1:6379> set k1 aaa ex 20
OK
127.0.0.1:6379> ttl k1
(integer) 17
127.0.0.1:6379> get k1
"aaa"
127.0.0.1:6379> ttl k1
(integer) 14

# 2. 发生写的时候，会剔除过期时间
127.0.0.1:6379> set k2 aaa ex 20
OK
127.0.0.1:6379> ttl k2
(integer) 17
127.0.0.1:6379> set k2 bbb  # 修改k2的值之后，缓存时间直接剔除了
OK
127.0.0.1:6379> ttl k2
(integer) -1
127.0.0.1:6379> get k2
"bbb"

# 3. 倒计时，且，redis不能延长
# 4. 定时
127.0.0.1:6379> TIME  # 查看当前时间
1) "1680439154"
2) "666008"
127.0.0.1:6379> EXPIRE k1 1680499154
(integer) 1
127.0.0.1:6379> ttl k1
(integer) 1680499152
127.0.0.1:6379> ttl k1
(integer) 1680499150
127.0.0.1:6379> ttl k1
(integer) 1680499148
127.0.0.1:6379> set k1 bb
OK
127.0.0.1:6379> ttl k1
(integer) -1
```

## key的过期原理

http://www.redis.cn/commands/expire.html

## Redis如何淘汰过期的keys

Redis keys过期有两种方式：**被动和主动方式。**（两种都在运行）

当一些客户端尝试访问它时，key会被发现并主动的过期。

当然，这样是不够的，因为有些过期的keys，永远不会访问他们。 无论如何，这些keys应该过期，所以定时随机测试设置keys的过期时间。所有这些过期的keys将会从密钥空间删除。

具体就是Redis每秒10次做的事情：

1. 测试随机的20个keys进行相关过期检测。
2. 删除所有已经过期的keys。
3. 如果有多于25%的keys过期，重复步奏1.

这是一个平凡的概率算法，基本上的假设是，我们的样本是这个密钥控件，并且我们不断重复过期检测，直到过期的keys的百分百低于25%,这意味着，在任何给定的时刻，最多会清除1/4的过期keys。

# 什么是零拷贝?

零拷贝(英语: Zero-copy) 没有说不需要拷贝，只是说减少冗余[不必要]的拷贝。

下面这些组件、框架中均使用了零拷贝技术：Kafka、Netty、Rocketmq、Nginx、Apache。

## Linux的I/O机制与DMA

在早期计算机中，用户进程需要读取磁盘数据，需要CPU中断和CPU参与，因此效率比较低，发起IO请求，每次的IO中断，都带来CPU的上下文切换。因此出现了——DMA。

DMA控制器，接管了数据读写请求，减少CPU的负担。这样一来，CPU能高效工作了。现代硬盘基本都支持DMA。

实际因此IO读取，涉及两个过程：

1、DMA等待数据准备好，把磁盘数据读取到操作系统内核缓冲区；

2、用户进程，将内核缓冲区的数据copy到用户空间。

这两个过程，都是阻塞的。

# Redis持久化

缓存：数据可以丢 急速！

数据库：数据绝对不能丢的 速度+持久性

掉电易失！

redis+mysql 》 数据库 《 不太对

**存储层：**

1，快照/副本

2，日志

![image-20230405181536185](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/redis的dbfile的处理.png)

**RDB持久化db.file，8:00开始， 8:30结束。如果再8:00-8:30之前，有key的值发生变化，那么持久化的数据中会发生变化吗？持久化的时间点是8:00还是8:30？**

先了解下下面两个知识点

## 管道

管道：

1，衔接，前一个命令的输出作为后一个命令的输入

2，管道会触发创建【子进程】

```shell
[root@manager-node vagrant]# num=0
[root@manager-node vagrant]# echo $num
0
[root@manager-node vagrant]# ((num++))
[root@manager-node vagrant]# echo $num
1
# 这里的num++ 并没有加。前一个命令的输出作为后一个命令的输入，这里的前后并没有关系
[root@manager-node vagrant]# ((num++)) | echo ok   
ok
[root@manager-node vagrant]# echo $num
1

# 获取当前进程的ID
[root@manager-node vagrant]# echo $$
2851
[root@manager-node vagrant]# echo $$ | more
2851
[root@manager-node vagrant]# echo $$ | more
2851
[root@manager-node vagrant]# echo $BASHPID
2851
[root@manager-node vagrant]# echo $BASHPID | more
26994
[root@manager-node vagrant]# echo $BASHPID | more
26996
```

Q：使用echo $$ | more 获取当前进程结果不会变， 但是使用 echo $BASHPID | more 结果会变，为什么？

A：$$ 优先级高于 |。| 会触发创建子进程，所以 echo $BASHPID | more 每次会发生变化

## 父子进程

使用linux的时候：

父子进程

- 父进程的数据，子进程可不可以看得到？

  > 常规思想，进程是数据隔离的！

- 进阶思想，父进程其实可以让子进程看到数据！

  > linux中
  >
  > export的环境变量，子进程的修改不会破坏父进程
  >
  > 父进程的修改也不会破坏子进程

```shell
# 安装pstree
yum -y install psmisc
```

![子进程获取不到父进程的num.png](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/子进程获取不到父进程的num.png)



```shell
# 可以通过export num来实现，子进程就可以获取到num
# 子进程只能看到父进程的num，不能修改

[root@manager-node vagrant]# echo $num
1
[root@manager-node vagrant]# export num
[root@manager-node vagrant]# /bin/bash    # 进入子进程
[root@manager-node vagrant]# echo $num
1
[root@manager-node vagrant]# exit    # 退出子进程
exit
```

验证两个问题

- 子进程修改了num，父进程能不能看到？
- 父进程修改了num，子进程能不能看到？

增加一个test.sh脚本

```shell
cat test.sh

#!/bin/bash

# 上面表示开启一个子进程取执行

echo $$
echo $num
num=999
echo num:$num

sleep 20   # 休眠10s

echo $num
```

最后 执行 chmod +x test.sh  修改下脚本的权限

**验证问题一：子进程修改了num，父进程能不能看到？**

```shell
# 结论是不会


[root@manager-node vagrant]# ./test.sh &     # 后台的形式启动test.sh
[1] 27081
[root@manager-node vagrant]# 27081    # 子进程当前的进程id是 27081
1                         # 子进程当前num=1
num:999                   # 子进程修改num=999

[root@manager-node vagrant]# echo $$       # 父进程当前的进程id是 2851
2851
[root@manager-node vagrant]# echo $num   # 父进程获取num还是1
1 
[root@manager-node vagrant]# 999        # 子进程获取num=999

[1]+  Done                    ./test.sh     # 子进程结束

```

**验证问题二：父进程修改了num，子进程能不能看到？**

```shell
# 结论是不会

[root@manager-node vagrant]# echo $$    # 当前是父进程
2851
[root@manager-node vagrant]# echo $num   # 父进程num=1
1
[root@manager-node vagrant]# ./test.sh &   
[1] 27083                               # 子进程id 27083
[root@manager-node vagrant]# 27083
1                    # 子进程获取num=1    
num:999              # 子进程修改num=999    

[root@manager-node vagrant]#
[root@manager-node vagrant]# echo $num    # 父进程num=1
1
[root@manager-node vagrant]# num=88       # 父进程修改num=88
[root@manager-node vagrant]# echo $num    # 父进程num=88
88
[root@manager-node vagrant]# 999         # 子进程num=999

[1]+  Done                    ./test.sh  # 子进程结束
```

创建子进程的速度应该是什么程度？

如果父进程是redis，内存数据比如10G

1，速度

2，内存空间够不够

![image-20230405180204516](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/父子进程相关知识.png)

![image-20230405181233657](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/写时复制.png)



> redis在fork子进程的时候，复制了指向内存的指针，这时候内存具体值的引用有了两份，在**父进程修改key的值的时候，新开辟一个内存保存新值**，这时候旧值的引用减少1，由2变成1，因为操作系统检测还有子进程引用了这个旧值的地址，所以不会删除旧值，达到牺牲少量内存提高创建子进程效率，这有点类似与jvm的内存回收机制。
>
> redis在RDB持久化时，fork子进程而不是用一个线程，就是为了保持redis主进程依然单线程操作数据，避免竞争条件和线程上下文切换。

![image-20230405181832774](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/fork子进程处理数据.png)

复制的是指针，不会全部数据都复制，所以增加的空间很少

父进程修改key的时候，会开辟一个新的内存空间指向新的地址

子进程还是指向老的内存地址key，所以持久化的时候，父进程对key进行修改的时候，fork子进程是不感知的

**所以持久化的时间点是8:00**

## RDB

![image-20230405185248214](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/RDB持久化方式.png)

> - 全量数据
> - 不支持拉链：是指日期拉链，是指文件不覆盖，再后面加上日期。但是redis不会，指挥有一个dump.rdb文件

## AOF

append only file （只会向文件中 追加）

![image-20230405220735678](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/AOF相关内容.png)

打开AOF

![image-20230405221952120](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/开启AOF配置.png)



打开混合

aof-use-rdb-preamble yes

```shell
127.0.0.1:6379>
127.0.0.1:6379> set k1 a
OK
127.0.0.1:6379> set k1 b
OK
127.0.0.1:6379> set k1 c
OK
127.0.0.1:6379> set k1 d
OK


----------------------
此时AOF文件记录的，还是操作命令
----------------------

# 执行 BGREWRITEAOF 之后
127.0.0.1:6379>  BGREWRITEAOF 之后
Background append only file rewriting started
127.0.0.1:6379>
```

![image-20230405223900765](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/aof文件出现RDB内容.png)

Q：aof写数据的操作是主进程完成的吗？还是有一个子进程？

A：redis在写入数据时，是追加的方式append到aof缓冲区的，这个过程是主进程完成，然后按照配置的参数刷新到AOF文件。

当需要AOF重写时，会fork子进程，与主进程共享内存空间，重写AOF并覆盖原有的AOF文件

![92cdff90ea6e2b972a3d83a54676a95c.png](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/AOF操作图示.png)



# AKF 

## 什么是AKF ？

AKF 立方体也叫做scala cube，它在《The Art of Scalability》一书中被首次提出，旨在提供一个系统化的扩展思路。AKF 把系统扩展分为以下三个维度：

- X 轴：直接水平复制应用进程来扩展系统。
- Y 轴：将功能拆分出来扩展系统。
- Z 轴：基于用户信息扩展系统。

如下图所示：

![img](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/AKF.jpg)

参考文章：https://www.cnblogs.com/-wenli/p/13584796.html

![Redis-AFK拆分](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/Redis-AKF拆分.png)

**单机、单节点、单实例的弊端？**

- 单点故障
- 容量有限
- 压力大

通过AKF拆分，一变多

解决了一写问题，同样会引入一些新的问题

- 数据一致性问题
  - 强一致性（同步阻塞，失去了可用性）
  - 弱一致性（异步方式，容忍丢失一部分数据）

# 主从复制

![主从复制](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/主从复制.png)

对主做高可用（HA）

自动故障转移：需要监控，代替人  -> 认识怎么监控的 -> 由一个技术活程序来实现（后面会将）-> 但是这个监控程序同样会存在单点故障问题 -> 怎么办？使用集群 -> 使用集群后同样会有上面提到的问题 -> 怎么办？

## 主从和主备的区别

主从：“从机”的“从”可以理解为“仆从”，仆从是要帮主人干活的，“从机”是需要提供读数据的功能的；
主备：“备机”一般被认为仅仅提供备份功能，不提供访问功能。
所以使用“主从”还是“主备”，是要看场景的，这两个词并不是完全等同。
一般”主从集群“和”主备集群“一起使用，让备机也提供读的服务，当主机宕机时备机代替主机工作提供写服务，其他从机继续提供读服务。
![image-20230408124003470](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/主从-主备-主主的区别.png)

## 为什么会需要选票过半？

1. 推导下：

> 统计不准确，不够势力范围
>
> 问题：网络分区
>
> 脑裂！

2 在3个节点成功解决脑裂问题

3 在4个节点成功解决脑力问题

3 在5个节点成功解决脑力问题

n/2+1  过半！

## 为什么使用奇数台？

如果有3台，需要2台节点成功，允许1台出现问题

如果有4台，需要有3台节点成果，也是允许1台出现问题

3台和4台承担的风险数量是一致的，但是成本不一样，另外4台比3台更容易出现1台节点故障

所以一般使用奇数台

# CAP原则

https://zhuanlan.zhihu.com/p/50990721

分布式系统（distributed system）正变得越来越重要，大型网站几乎都是分布式的。

分布式系统的最大难点，就是各个节点的状态如何保持一致。CAP理论是在设计分布式系统的过程中，处理数据一致性问题时必须考虑的理论。

CAP即：

- Consistency（一致性）
- Availability（可用性）
- Partition tolerance（分区容忍性）

# Redis主从复制演示

http://www.redis.cn/topics/replication.html

| 角色   | 端口号 | 监控port |
| ------ | ------ | -------- |
| master | 6379   | 26379    |
| slave  | 6380   | 26380    |
| slave  | 6381   | 26381    |

> 这里为了演示方便，只在一台机器上启动三个redis进程。实际生成应该是每台机器一个redis进程的

```shell
# 使用./install_server.sh命令，生成6380和6381两个进程的配置文件
# 然后修改下三个节点的配置文件

# 改成no，我们一前台的方式启动，这样更方便看下日志
daemonize no

# 把下面logfile的配置注释掉，因为我们需要再控制台上看日志
#logfile /var/log/redis_6380.log
```

```shell
# 查看下 SLAVEOF 命令，这里解释redis5之前是 SLAVEOF，redis5之后用 REPLICAOF
# 这里我们使用 REPLICAOF
127.0.0.1:6379> help SLAVEOF

  SLAVEOF host port
  summary: Make the server a replica of another instance, or promote it as master. Deprecated starting with Redis 5. Use REPLICAOF i                                                                                nstead.
  since: 1.0.0
  group: server

# 看下 REPLICAOF 的帮助
127.0.0.1:6379> help REPLICAOF

  REPLICAOF host port
  summary: Make the server a replica of another instance, or promote it as master.
  since: 5.0.0
  group: server

# 6380客户端上执行 REPLICAOF 命令
127.0.0.1:6380> REPLICAOF 127.0.0.1 6379
OK
```

![image-20230408181059386](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/主从复制6379控制台日志.png)

> 6379 控制台日志

![image-20230408181304584](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/主从复制6380控制台日志.png)

```shell
# 6381客户端上执行 REPLICAOF 命令
127.0.0.1:6381> REPLICAOF 127.0.0.1 6379
OK

# 从节点默认是不支持写数据的
127.0.0.1:6381> set k2 bbb
(error) READONLY You can't write against a read only replica.

```

## 从节点故障

思考：

如果slave挂了，后面经过一段时间我又修复好了，那么从节点同步master节点的数据，是增量同步，还是全量？

```shell
# 此时keys * 只有k1
127.0.0.1:6379> keys *
1) "k1"

# 现在将6381进程结束

# 然后再添加一个k2
127.0.0.1:6379> set k2 bbb
OK

# 然后使用下面命令启动6381
redis-server /etc/redis/6381.conf  --replicaof 127.0.0.1 6379

# 查看6381的keys *，发现k1 和 k2都有
127.0.0.1:6381> keys *
1) "k2"
2) "k1"
```

![image-20230408205206334](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/从节点挂了重启之后的控制台日志.png)

我们看此时的日志，没有FLUSHDB

RDB的dump文件会记录replicaId

```shell
# 此时再将6381进程结束

# 然后添加个k3
127.0.0.1:6379> set k3 ccc
OK

# 然后再启动6381进程，打开AOF     
redis-server /etc/redis/6381.conf  --replicaof 127.0.0.1 6379 --appendonly yes

# 查看6381的keys *，发现k2也有了
127.0.0.1:6381> keys *
1) "k2"
2) "k1"
3) "k3"
```

此时appendonly文件中论和的RDB内容是没有记录replicaId的

## 主节点故障

```shell
# 结束6379节点，让主节点故障

# 此时两个从节点的控制台就一直报下面错
28068:S 05 Apr 2023 16:35:13.270 # Error condition on socket for SYNC: Connection refused
28068:S 05 Apr 2023 16:35:14.311 * Connecting to MASTER 127.0.0.1:6379

# 我们现在让6380当作主节点
# 再6380节点上执行 REPLICAOF no one，解除其从节点的角色
# 此时6380节点的控制台就不会报上面的错误了，但是6381还是会报
127.0.0.1:6380> REPLICAOF no one
OK

# 现在让6381，执行主节点是6381
127.0.0.1:6381> REPLICAOF 127.0.0.1 6380
OK

# 此时两个节点都没有报错了
# 6380 转换位主节点了
```

这是我们手动选择主节点，当然我们也可以通过监控程序（sentinel）来帮我们自动选取

## 主从复制相关配置

```shell
# When a replica loses its connection with the master, or when the replication
# is still in progress, the replica can act in two different ways:
#
# 1) if replica-serve-stale-data is set to 'yes' (the default) the replica will
#    still reply to client requests, possibly with out of date data, or the
#    data set may just be empty if this is the first synchronization.
#
# 2) if replica-serve-stale-data is set to 'no' the replica will reply with
#    an error "SYNC with master in progress" to all the kind of commands
#    but to INFO, replicaOF, AUTH, PING, SHUTDOWN, REPLCONF, ROLE, CONFIG,
#    SUBSCRIBE, UNSUBSCRIBE, PSUBSCRIBE, PUNSUBSCRIBE, PUBLISH, PUBSUB,
#    COMMAND, POST, HOST: and LATENCY.
#
replica-serve-stale-data yes  # 如果是yes，表示从节点在同步的过程中，依然提供相关查询服务，此时可能会查询出过期的数据

# 从节点只读（只支持查询）
replica-read-only yes

# redis之前传递数据的时候，是否经过磁盘，no 表示使用磁盘。 diskless 是无盘的意思
# 方式一： redis  - 经过磁盘IO ->  rdb.dump 文件  --网络IO-->  redis
# 方式二： redis  --网络IO-->  redis
repl-diskless-sync no

# 增量复制对了大小，默认是注释掉的
# repl-backlog-size 1mb

# 默认是注释掉的，最小重写的次数
# min-replicas-to-write 3
# min-replicas-max-lag 10
```

![image-20230408213248257](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/redis增量复制和是否经过磁盘图.png)

## Sentinel

Sentinel 哨兵的意思

http://www.redis.cn/topics/sentinel.html

Redis 的 Sentinel 系统用于管理多个 Redis 服务器（instance）， 该系统执行以下三个任务：

- **监控（Monitoring**）： Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。
- **提醒（Notification）**： 当被监控的某个 Redis 服务器出现问题时， Sentinel 可以通过 API 向管理员或者其他应用程序发送通知。
- **自动故障迁移（Automatic failover）**： 当一个主服务器不能正常工作时， Sentinel 会开始一次自动故障迁移操作， 它会将失效主服务器的其中一个从服务器升级为新的主服务器， 并让失效主服务器的其他从服务器改为复制新的主服务器； 当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器。

```shell
# 哨兵启动的方式1
redis-server /path/to/sentinel.conf --sentinel

# 哨兵启动的方式2
redis-sentinel /path/to/sentinel.conf

# 增加三个哨兵需要的配置文件，可以简单点
cat /etc/redis/26379.conf
 port 26379
 sentinel monitor mymaster 127.0.0.1 6379 2

cat /etc/redis/26380.conf
 port 26380
 sentinel monitor mymaster 127.0.0.1 6379 2
 
cat /etc/redis/26381.conf
 port 26381
 sentinel monitor mymaster 127.0.0.1 6379 2
 
 
# 启动哨兵
redis-server /etc/redis/26379.conf --sentinel
redis-server /etc/redis/26380.conf --sentinel
redis-server /etc/redis/26381.conf --sentinel
```

哨兵启动完成之后，我们现在来关闭6379主节点进程

观察日志，发现，主节点进程结束之后，三个哨兵并不会立马就选出一个从节点作为主节点

因为节点之间是通过网络连接的，既然是网络，会有丢包、抖动等情况发生，所以并不会立即就选个主节点，而是过了一定时间才会选择一个主节点

**哨兵会改自己的配置文件**

![image-20230408220453619](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/哨兵修改之后的配置文件.png)

Q：哨兵之监控了master，那么它是怎么知道其他从节点和其他哨兵的呢？

A：是通过redis自带的发布和订阅模式来实现的

![image-20230408220824786](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/哨兵和节点和其他哨兵发布订阅通信信息.png)

> 可以看到发布订阅的通道名称：__sentinel__:hello

![image-20230408221056754](F:/gitee_project/mca-learning/P6_分布式框架、中间件技术群、分布式解决方案/Redis缓存数据库/img/sentinel总结图.png)

# 