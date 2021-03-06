# 分布式锁

## **背景&理论**

在单机时代，如果有多个线程要同时访问某个共享资源的时候，可采用线程间加锁的机制**(Synchronize/Lock-源码和实现)**；

分布式系统，线程间的所机制就失效了，系统会有多份并且部署在不同的机器上，这些资源已经不是在线程之间共享了，而是属于进程之间的共享资源。

分布式锁要求：

- 排他性：同一时间只会有一个客户端能获取到锁；
- 避免死锁：有限的时间，一定会被释放(正常释放或异常释放)；
- 高可用：获取或释放锁的机制必须高可用且性能佳；

------

##  分布式锁的实现方式

三种，实现的复杂度难道依次递增：

- 基于数据库的实现；
- 基于Redis实现；
- 基于Zookeeper实现；

**基于数据库实现**：

- 基于数据库的乐观锁；
- 基于数据库的悲观锁(**InnoDB中只有字段加了索引的，才会是行级锁，否则是表级锁**)；

**基于Redis实现**：SETNX命令

**基于Zookeeper实现**：

原理：当某客户端要进行逻辑的加锁时，就在Zookeeper上的某个指定节点的目录下，**生成一个唯一的临时有序节点，然后判断自己是否是这些有序节点中序号最小的一个**，如果是，则算获取了锁；如果不是，则需要在序列中找到比自己小的那个节点，并对其调用exist()方法，**对其注册事件监听，当监听到这个节点被删除了，那就再去判断一次自己当初创建的节点是否变成序列中最小的**(宕机的话，临时节点会被自动删除)。

## Redis分布式锁

**实现方式**：

1. setnx+lu

2. set key value px milliseconds nx；

   ```
   //获取锁(unique_value可以是UUID等)
   SET resource_name unique_value NX PX 30000
   // 释放锁(lua脚本中，一定要比较value，防止误解锁)
   if redis.call("get", KEY[1] == AVG[1]) then
   	return redis.call("del", KEY[1])
   else
       return 0
   end    
   ```

   这种实现方式有3大要点：

   - set命令要用set key value px milliseconds nx；
   - value 要具有唯一性；防止误删除别人的锁
   - 释放锁时要验证value值，不能误解锁；

   这种实现的锁最大的缺点：**加锁只能作用在一个Redis节点上**，即使Redis通过sentinel保证高可用，**如果这个master节点由于某些原因发生了主从切换，那么就会出现锁丢失的情况**：

   - 在Redis的master节点上拿到了锁；
   - 但是在加锁的key还没有同步到slave节点；
   - master故障，发生故障转移，slave节点升级为master节点；
   - 导致锁丢失；

**RedLock实现**：

**redlock算法**：

在Redis的分布式环境中，我们假设有N个Redis Master；这些节点完全互相独立，不存在主从复制或者其他集群协调机制，我们确保将在N个实例上使用与**在Redis单例实例下相同方法获取和释放锁**；

客户端执行如下操作：

获取当前Unix时间，以毫秒为单位；依次尝试从5个实例，使用相同的key和具有唯一性的value获取锁。**当想Redis请求获取锁时，客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间**。目的：避免服务器端Redis已经挂掉，如果服务器端没有在规定时间内响应，客户端尝试去另外一个Reids实例请求获取锁。

客户端使用当前时间减去开始获取锁时间(步骤1记录时间)就得到获取锁使用的时间，当且仅当从大多数(N/2+1)的Redis节点都取到锁，**并且使用的时间小于锁失效的时间时，锁才算获取成功**。

获取到锁，**key的真正有效时间**等于**有效时间减去获取锁所使用的时间**；

如果因为某些原因，获取锁失败(没有在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间)，客户端应该在所有的Redis实例上进行解锁(即便某些Redis实例根本就没有加锁成功，防止某些节点获取到锁，但是客户端没有得到相应，而导致下来的一段时间不能被重新获取锁)。