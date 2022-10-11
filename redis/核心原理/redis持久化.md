5W2H

Why:为什么需要持久化

What:什么是redis的持久化

Where:持久化在哪里

When: 何时触发持久化

Who: redis做持久化的是工作线程还是后台线程

How:如何持久化的

How much:持久化的消耗是什么





### 为什么Redis需要持久化

1. redis是内存数据库，宕机后数据会消失
2. redis重启后快速恢复数据，要提供持久化机制
3. redis持久化是为了快速恢复数据而不是为了存储数据
4. redis主从复制依赖持久化



注意：redis的持久化不保证数据的完整性

当redis用作DB时，DB数据要完整，所以一定要有一个完整的数据源（文件、mysql）

在系统启动时，从这个完整的数据源中将数据load到Redis中

数据量较小，不易改变，比如：字典库（XML,Table）





### redis如何实现持久化的

redis的持久化方式有两种：

1. RDB (Redis DataBase)
2. AOF (Append Only File)



##### RDB

RDB（Redis DataBase），是redis**默认**的存储方式，RDB方式是通过**快照**（snapshotting）完成的。只关注这一时刻的数据，不关注如何生成数据的。



**如何实现的**

Redis会通过fork函数创建一个**子进程**来完成数据的持久化





**何时会持久化**

触发RDB的方式有：

1. 符合自定义配置的快照配置
2. 执行save或者bgsave命令
3. 执行flushall命令
4. 执行主从复制操作（第一次）



**配置参数定期执行**

在redis.conf中配置：save       多少秒数据变了多少

```shell
save "" # 不使用RDB存储 不能主从
```



**命令显示触发**

在客户端输入bgsave命令



**RDB的执行流程（原理）**





#####AOF









### Q&A

Linux fork函数作用



如何监控redis的持久化

可以通过info命令查看