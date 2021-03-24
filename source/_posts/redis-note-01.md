---
title: Redis笔记01
date: 2021-03-24 09:18:07
categories:
    - Redis
hide: false
comments: true
---
使用Redis时有时候会碰上一些并发的问题，这里来说一下分布式锁以及加锁超时等问题。
<!--more-->
### 0x00
一般情况下，会对那些变化不大但是访问量比较大的请求做缓存处理，但缓存会在某个时刻失效，
此时就会有大量的请求涌向数据库从而容易引发雪崩效果。
### 0x01
为了避免雪崩，我们可以在更新缓存的代码块（即访问数据库）中加入分布式锁，只能够让一个线程去做缓存更新:
```
if (redis.setIfAbsent(lock, 1)) { 
    // 更新缓存逻辑
    redis.del(lock);
}
```
但是上面如果更新逻辑出现卡死现象就会造成死锁，所以我们还需要给锁设置超时时间，
但是设置超时时间一般的工具例如spring的RedisTemplate是没有把检查存在并设置值和设置超时时间
作为一个请求发送的，则无法保证是原子操作，这样也会发生在设置超时时间的时候没有成功导致死锁。  

由于redis新版本支持了setNX和EX复合指令的原子操作（即判断存在和设置超时时间），
所以我们可以去扩展RedisTemplate或者当前使用的工具类；另一种方法则是用lua脚本，执行lua脚本也是一次原子操作。
> redis在4.0之前整体都是单线程的，4.0后开始加入多线程，但也仅是部分操作，
> 所以就算非复合指令也可以通过一个请求发送多个指令的方式来达成原子操作的目的。
```
if (redis.setAndExpireIfAbsent(lock, 1, 10000)) { 
    // 更新缓存逻辑
    redis.del(lock);
}
```
> 注意   
> redis 2.6.12 之前，set返回永远为ok，之后则设置成功时返回ok，
> 加入条件参数不成立则返回空
### 0x02
但我们可能还会发现另一个问题，更新逻辑太久了超过了超时时间，此时锁已经被解除了，
这就会执行导致途中另一个线程获取到了锁，导致后面删除的时候是删除的另一个线程加的锁。  

所以我们需要引入一个随机id，作为当前线程加锁的标识，若后面发现不是相同id则不做删除。
```
long random = SnowFlakeGenerator.getInstant().nextId();
if (redis.setAndExpireIfAbsent(lock, random, 10000)) { 
    // 更新缓存逻辑
    if (redis.get(lock) == random) {
        redis.del(lock);
    }
}
```
Ok，到了这里你可能也发现了，这个删除是不是也要做原子操作比较好点(  
没错...  
若是不做原子操作，那可能就会虽然拿到锁的值能够匹配上，但是下一个瞬间就因为超时而被别的其他线程获取到锁
从而又引发了上面的问题，删错了别的线程的锁。
### 0x03
类似的由于超时导致的问题还有一些情况就是计数器
```
synchronized (LOCK) { //只是为了排除多线程情况，这里只想讨论超时问题，实际情况还要具体分析
    if(redis.exists(userId)) {
        redis.incr();
        if (redis.get(userId) > maxAllowedTimes) {
            return false;
        }
        return true;
    } else {
        redis.set(userId, 1);
        redis.setExpire(60000);
        return ture;
    }
}
```
上面是一段限制用户一分钟内可访问次数的redis计数器。  
这里面如果判断到存在后的下一个瞬间恰好超时，此时incr方法在redis的行为就是先创建并设置值为0，然后加1，
而没有设置过期时间。导致后面永远被限制访问。  
```
synchronized (LOCK) {
 
    if(redis.setAndExpireIfAbsent(userId, 1, 60000)) {
        return true;
    } else {
        long times = redis.get(userId);
        if(times < maxAllowedTimes) {
            redis.setIfPresent(userId, ++times);
            return true;
        }
        return false;
    }
}
```
  
  