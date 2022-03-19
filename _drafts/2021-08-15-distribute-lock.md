---
layout: post
title: Java分布式锁原理及实现方案
categories: Java
description: java分布式锁原理及几种常见的实现方式
keywords: Java, 分布式锁
---
本文讲述了Java分布式锁的原理及几种常见的实现方式

# 原理
在使用多线程时，常常会提到线程安全问题。解决线程安全问题，通常使用加锁的方式，比如java提供的**synchronized**关键字，在jdk1.5中，java又引入了更灵活的**ReentrantLock**。简单来说，加锁就是占坑，谁先占住了，谁就可以继续执行，没有占成功的需要等待。对于需要使用相同资源的线程来说，"坑"是唯一，这样就能保证一次只能有一个线程访问该资源。这种锁称为互斥锁，即只有获得锁的线程才能访问资源。

对于单服务来说，这个"坑"可以是内存中的一个标志位，所有的线程都去竞争这个标志位即可（需要通过原子指令实现）。然而在分布式场景下，需要竞争锁资源的线程往往不在同一个服务，此时这个"坑"该放在哪，如何保证"坑"的唯一性，是分布式锁需要解决的首要问题。

除了互斥外，有些场景还需要支持锁的可重入，上面提到java的两种锁机制，都支持可重入。可重入锁，简单来说，一个线程如果已经获取了锁，再次取获取锁是，也是成功的。比如递归的时候，就可能会出现这种场景

一般来说，都会使用中间件来保证互斥，常见的方式有
- 数据库
- zookeeper
- redis
- etcd
引入中间件后，还需要考虑可靠性和可用性等问题。
接下来，本文会基于Java语言，详细说明这几种方案的实现。当然其他语言实现的原理也是一样的

# 1 基于数据库实现分布式锁
## 1.1 如何保证唯一性
数据库要数据的唯一性，首先想到的就是通过表的唯一约束来做了。可见建立下面一张表：
```
CREATE TABLE `distribute_lock` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `resource` varchar(190) NOT NULL DEFAULT "" COMMENT '资源标识',
    `lock_id` varchar(190) NOT NULL COMMENT '锁标识',
    `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `unique_resource` (`resource`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='分布式锁表';
```
注意事项：
- 资源标识，比如需要给一张user表加锁，那么资源标识就是user，如果是要给user中userId=1的行加锁，那么资源标识就是user.userId=1。
- 锁标识，其实就是线程的标识，可以使用 设备id+服务id+线程id，所以可以唯一确定该线程的标识即可
- 唯一键，注意，唯一键是 **resource**，只要存在该资源标识的记录，就代表该资源已被占用。
### 1.1.1 申请锁 
申请锁sql就是一条insert语句：
```
INSERT INTO distribute_lock(resource,lock_id) VALUES ("resource","lock_id");
```
当insert失败时，则代表申请锁失败。
### 1.1.2 释放锁
释放锁sql：
```
DELETE FROM distribute_lock where resource='resource' and lock_id='lock_id';
```
## 1.2 如何实现可重入锁
要实现可重入锁，直接在表中增加一个count字段记录加锁的次数即可。
```
CREATE TABLE `distribute_lock` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `resource` varchar(190) NOT NULL DEFAULT "" COMMENT '资源标识',
    `lock_id` varchar(190) NOT NULL COMMENT '锁标识',
	`count` int(11) NOT NULL COMMENT '可重入锁, 加锁次数',
    `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE KEY `unique_resource` (`resource`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='分布式锁表';
```
### 1.2.1 申请锁
申请锁时分为两种情况
- 初次申请锁，直接insert一条记录
```
INSERT INTO distribute_lock(resource,lock_id,count) VALUES ("resource","lock_id",1);
```
- 非初次申请锁，update count=count+1
```
UPDATE distribute_lock set count=count+1 where resource ='resource' and lock_id = 'lock_id';
```
这里分两条语句执行，可以**不需要**加事务，直接先执行insert语句，如果失败，再执行update语句，如果也失败，则获取锁失败。只要有一个成功，则返回获取锁成功。
当然，也可以将这两条语句合并为一条upsert语句，可以提高性能。
```
INSERT INTO distribute_lock(resource,lock_id,count) VALUES ("resource","lock_id",1) ON DUPLICATE KEY UPDATE count=count+1;
```
### 1.2.2 释放锁
释放锁也要分两次执行，先将count-1
```
UPDATE distribute_lock set count=count-1 where resource='resource' and lock_id='lock_id';
```
再判断count < 1时删除记录
```
DELETE FROM distribute_lock where resource='resource' and lock_id='lock_id' and count < 1;
```
同理，对这两条语句也不需要事务，直接执行即可。
## 1.3 如何保证高可用
由于引入了mysql，就需要考虑mysql的高可用，即当一个mysql节点宕机后，锁的获取还能正常工作，此时就需要mysql部署集群。部署集群后，就会发生锁的可靠性问题。
## 1.4 如何保证可靠性
mysql搭建集群，集群直接的数据同步时有延迟的，此时锁的获取就会变的不可靠。当线程A获取资源成功后，如果此时mysql主节点宕机，数据还没同步到从节点，此时线程A也获取同一个资源的锁，也可以成功。那么线程A和线程B同时获取到了资源的锁，可能导致数据的不一致。对于这种情况，没有太好的解决办法，唯一的方法时参考redis的红锁机制，但这方法的代价太高，需要部署多个独立的mysql节点。具体可以见下文中redis红锁实现。

除此之外，由于没有超时机制，当程序获取锁后，由于程序崩溃及网络等问题，可能导致锁无法释放。所以还需要另外加定时任务删除长时间未释放的锁。
## 1.5 小结
优点
- 理解简单，不需要维护额外的其他中间件（绝大多数系统中都会使用数据库），比如reids，ZK
缺点
- 性能问题，访问数据库性能相对较低
- 可靠性问题，mysql节点之间数据不是强一直，没有太好的解决方法
- 没有成熟的mysql客户端集成锁机制，需要自己开发
- 需要使用定时任务删除长时间未释放的锁，且锁的有效时间无法延长（也可以延长，需要再表中再加字段保存锁的有效时间）

上面的实现中是以mysql为例，实际实现不限于mysql。
## 1.6 代码实现
xxx
# 2 ZooKeeper实现
