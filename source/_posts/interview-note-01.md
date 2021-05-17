---
title: 面试题笔记01
date: 2021-05-17 09:18:07
categories:
    - 编程
tags:
    - MQ
    - Micro Service
    - Redis
hide: false
comments: true
index_img: /gallery/Arknights01.jpg
banner_img: /gallery/Arknights01.jpg
---
记录一下看过的一些面试题。
<!--more-->
> 封面  
> 这位画师的脸以及上色很有特点，稍微留意一下很容易就能够辨别出来了，只是表情总给人一种同样的感觉w
> https://twitter.com/__LM7__/status/1379416474293465090

## MQ接收到消息后无法查询到或者是旧的状态

原因
- 数据库回滚  
- 数据库事务未提交

解决办法
- 可以尝试将mq放在数据库事务之后执行。
- 可以利用@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)监听spring事务afterCommit阶段。
  
    ```java
    @Transactional
    public boolean saveFoo(FooEntity fooEntity) throws InterruptedException {
        log.error("start insert foo");
        fooRepository.save(fooEntity);
        publisher.publishEvent(new MyTransactionEvent(fooEntity.getFooName()));
        log.error("end insert foo");
        Thread.currentThread().sleep(2000);
        log.error("to commit insert");
        return true;
    }
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void afterCommit(MyTransactionEvent event) {
        log.error("after commit then send event {}", event);
        log.error("after commit then send event {}", event.getName());
    }
    ```
    发送的spring事件会在监听到AFTER_COMMIT时执行  
    参考：https://blog.csdn.net/little_kelvin/article/details/111330768

## MQ偶然接收到重复消息
接收端消费了消息但是没有向broker发送ack或者broker没有接收到ack，导致broker将消息再次入队被其他接收端或同一个接收端消费，造成了重复消费消息

接收端方法做幂等：
新增，则可以在消息做一个唯一主键，重复了则会异常，保证数据库没有脏数据。
修改，一般都为幂等，修改多少次一般都是一样的结果。

如果还是比较困难，则用redis记录每次消费并生成全局唯一键<id,message>，每次消费查询一次redis，存在消费记录则不进行消费（处理）
### 熔断与服务降级 
当请求某个服务超时或是响应过慢，并在一定时间内次数达到一定阈值，为了防止调用链路响应过长而引发的服务雪崩，我们将暂时不去请求这个服务， 而是调用降级方法，并下线该服务。期间一般会在过去一定时间后，尝试再次请求该服务，获得响应后才会上线，不然都将是调用降级方法。

例如Hystrix，它有一个滑动时间窗的概念，在这个滑动的时间窗内（默认20s），错误率达到阈值（默认50%）则将打开熔断器，并经过一段时间之后（默认5s）再次执行一次检测是否应该打开熔断器。
熔断器在打开期间，调用此服务将直接返回失败（服务降级），不再远程调用。

> ps: 服务雪崩是指由于调用链路当中，下游的服务响应太慢或者超时导致上游服务的请求得不到释放， 逐渐导致连接数达到上限从而又对上游服务造成影响 ，如此往复直到最上游服务难以承受压力（超过最大连接数等）


  
  