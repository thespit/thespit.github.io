---
layout: post
title: "Redis Pipelining管道模式介绍"
date: 2018-04-05
author: Thespit
---
# 1. 背景介绍

Redis官网有一篇文章提到了[通过使用Pipelining来提升Redis处理性能](https://redis.io/topics/pipelining)。

## 1.1 RTT（Round Trip Time）

Redis是一个基于TCP的请求响应式服务器。一条Redis指令从客户端发送给Redis服务器，处理完之后，结果返回给客户端，如此一个来回这个指令才算执行完成。

因为游戏服务器和Redis服务器往往放在不同的机器上，所有的消息在游戏服务器和Redis服务器之间通信时都会受到一个短小的网络延迟的影响，称之为RTT（Round Trip Time）。

通过ping命令可以查看两台机器之间的RTT值。

![0F621C6E9DE56911CF49E1C46EFE6C](https://thespit.github.io/pic/30F621C6E9DE56911CF49E1C46EFE6C4.jpg)

可以看到哪怕是往远程服务器发送一个ping消息平均都需要2.438 ms才能走一个来回。

## 1.2 RTT意味着什么

假设RTT为2ms，那么在客户端是单线程串行处理的情况下，两台机器之间1秒钟最多只能请求/响应500次。要知道Redis对外宣称的性能是每秒数万笔请求的处理能力。

那是不是提升性能只能靠异步模式，或增加客户端的线程了呢？

Redis给出的答案是Pipelining和Lua Script。

## 1.3 原理

Pipelining模式将一批Redis命令整合成一个大命令列表，一次性发送。由Redis服务器统一处理，统一返回结果列表。Jedis客户端写法类似这样：

```java
        Pipeline p = jedis.pipelined();
        for (int i = 0; i < 1000; ++i) {
            p.set("key" + i, "a");
        }
        List<Object> results = p.syncAndReturnAll();
        jedis.close();
```

而Lua Script提升系统吞吐量的原理和Pipelining是类似的，在这种模式下，客户端传递一个Lua脚本给Redis服务器，由Redis服务器直接在本地执行脚本处理批量数据，并统一返回结果。他们其实都是从RTT着手去解决问题。

# 2. 版本问题

Redis最开始就支持Pipelining，2.6版本开始支持Lua Script。

> Redis supports pipelining since the very early days, so whatever version you are running, you can use pipelining with Redis.

[腾讯Redis的版本说明](
https://cloud.tencent.com/document/product/239/8034?fromSource=gwzcw.14349.14349.14349)

> 目前腾讯云Redis主从版主要版本使用的是2.8.23。同时腾讯云Redis即将支持3.2.6版本，尽请期待。

腾讯云现在的2.8版明确说明是支持Lua Script的，而Pipelining应该是默认就有的功能。

后来据说腾讯的Redis到底是否支持需要每个项目自己去沟通。

# 3. 使用时机

从根本上说Pipeline解决了阻塞运行模式下，I/O利用率低的问题。

我觉得游戏最有可能用到的情形是：

## 3.1 一次业务操作需要循环get多个key，甚至修改他们的值。

譬如，一个for循环，里面带着`redis.xxx()`的形式，可以考虑用Pipelining。

```java
    for (int i = 0; i < 100; ++i) {
        value = redis.get("key" + i);
    }
```

目前好友或者跨服玩法功能可能会出现类似情况。

但是因为业务相关的key们一般都是存在一个hashmap里，所以大部分情况下用hmset和hmget就搞定了。这些语句也是用一个命令就批量处理了一堆请求。

## 3.2 一次业务操作需要顺序执行数个Redis指令

比如：

好友请求列表的各种操作，A、B两个玩家分别有自己的请求队列，一次操作造成两边各种修改。（以下业务代码可能也是需要优化的。）

```java
        RedisUtil.GAME().zrem(friendApplyRedisKey, rid);
        RedisUtil.GAME().zadd(friendApplyRedisKey, time, rid);
        RedisUtil.GAME().expire(friendApplyRedisKey,
                PublicSynchroData.ADD_FRIEND_REQUEST_REDISKEY_EXPIRE);

        RedisUtil.GAME().lrem(friendRequestRedisKey, 1, targetRid);
        RedisUtil.GAME().lpush(friendRequestRedisKey, targetRid);
        RedisUtil.GAME().expire(friendRequestRedisKey,
                PublicSynchroData.ADD_FRIEND_REQUEST_REDISKEY_EXPIRE);
```

虽然看着不起眼，这一个方法执行下去也是来回走了6遍请求回应了。

当然了，这种情况下用pipelining是否有必要，性能是否会有明显提升，是存疑的，目前来看，这种好友业务请求线上也不会大批量瞬间发生。在压测业务接口出现瓶颈前倒是没必要换实现方式。

## 3.3 异步

基于3.2做一个猜想，游戏出现了一种跨服玩法，所有参与的玩家都会请求，瞬间爆发大量Redis操作，那业务Service自己开一个Pipelining，缓存同时发生的Redis操作，定时发送异步push消息响应，相对于换用异步Redis客户端来说，是否是一个比较便宜的解决方案。

# 4. 性能测试

我在本地简单地写了一个例子，由我的电脑往内网开发机的Redis发1000次set请求。

分别针对Pipeline模式、普通模式和不销毁Jedis的改进普通模式计时输出。

```java
    public static void testRedis() {
        RedisInstance inst = (RedisInstance) RedisUtil.GAME();

        // pipeline模式
        long now = TimeUtils.getCurrentMills();
        Jedis jedis = inst.getClient();
        Pipeline p = jedis.pipelined();
        for (int i = 0; i < 1000; ++i) {
            p.set("S80_test_p_" + i, "a");
        }
        List<Object> results = p.syncAndReturnAll();
        jedis.close();
        long past = TimeUtils.getCurrentMills() - now;
        System.out.println("pipeline:" + past + "ms");
        System.out.println("result size:" + results.size() + " result class:"
                + results.get(0).getClass().getSimpleName());

        // 最普通的for循环模式
        now = TimeUtils.getCurrentMills();
        for (int i = 0; i < 1000; ++i) {
            inst.set("S80_test_n_" + i, "a");
        }
        past = TimeUtils.getCurrentMills() - now;
        System.out.println("normal:" + past + "ms");

        // 改进的for循环模式，jedis拿出来执行1000次再销毁。
        now = TimeUtils.getCurrentMills();
        jedis = inst.getClient();
        for (int i = 0; i < 1000; ++i) {
            jedis.set("S80_test_m_" + i, "a");
        }
        jedis.close();
        past = TimeUtils.getCurrentMills() - now;
        System.out.println("improved normal:" + past + "ms");
    }
```

首先看一下我本地ping服务器的延迟，大概是2.4ms左右。

![IMAGE](https://thespit.github.io/pic/23100E7C8976963028BC530335F4E6E0.jpg)

程序输出结果：

![IMAGE](https://thespit.github.io/pic/7E1BA5751F2D7AA10E81B014E6E84CC4.jpg)

分析：

1. pipeline模式就一次I/O，当然很快。
2. for循环的传统串行方式享受了1000次的网络延迟影响，并且每次JedisClient都会拿出再释放。
3. 改进的for循环方式只拿出一次JedisClient，执行1000遍set之后再销毁。可以看到所花费的时间基本就是网络延时2～3ms乘以1000次的结果。

最后去Redis简单查看结果，pipelining模式确实也能顺利地存入值。

![IMAGE](quiver-image-url/7E1BA5751F2D7AA10E81B014E6E84CC4.jpg =353x266)

# 5. 空间占用问题

那么究竟一次Pipelining合并发送多少笔请求是合理的，是否越多越好。关于这一点，[官网的文章](https://redis.io/topics/pipelining)里有提到。

> IMPORTANT NOTE: While the client sends commands using pipelining, the server will be forced to queue the replies, using memory. So if you need to send a lot of commands with pipelining, it is better to send them as batches having a reasonable number, for instance 10k commands, read the replies, and then send another 10k commands again, and so forth. The speed will be nearly the same, but the additional memory used will be at max the amount needed to queue the replies for this 10k commands.

多少指令一起发，主要考量的是Redis服务器的内存。由于Redis服务器会把Pipelining整合的请求先缓存，然后再一个个执行，因此需要额外的指令存储内存开销。文章也顺便提了10K这么一个消息队列的数量级。

因为Redis指令缓存究竟有多大，暂时还没发现测试方法，所以我对这个事情的看法是这样的：

1. 一个批次发送10K消息应该是远超过我们的需求了，在目前的游戏业务环境下，不太可能一个Game服会瞬间需要发送10K数量级的Redis指令。即使是活跃最顶峰状态，所有玩家都瞬间点击某个热点玩法，单服瞬间Redis请求数都不太会达到10K。
2. 关于额外的内存开销，因为Redis命令都是字符串，首先假设单个指令非常大的情况，比如1Kb，1000个指令一起缓存也就是1Mb。就算Redis变着花样存，估计也不会是惊人的数字。比起Redis内存的额外开销，我倒觉得这种请求瞬间爆发的情况下，Game服的CPU或者网络带宽会先吃紧。
3. 官网的NOTE里也说了，指令太多了（比如超过了10K条）可以拆成两批发，性能不会有太大变化（由一次RTT延迟变成了两次）。如果真的对大批次指令有担忧的话，可以根据业务指令大小定一个较为安全的批次发就行。