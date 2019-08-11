---
title: Re：从零开始的领域驱动设计
date: 2019/8/11
categories: 
    - Coding
tags: 
    - DDD
    - CQRS
    - Event Sourcing
comments: true 
thumbnail: /gallery/FINAL FANTASY XIV SHADOWBRINGERS.png
---
领域驱动设计(Domain-driven design)，缩写为DDD。以领域设计为驱动，构建整一个系统。

这个设计思想是在微服务开始流行时逐渐变得火爆的，因为其设计理念非常适合分布式的微服务拆分。
<!--more-->

> 我声明一点，本文章其实都是东拼西凑的，里面所表达的仅仅是个人的理解（我没有读完ddd那本书）

# 层结构(Layered Architecture)
![Layered Architecture](p29.png)
- User Interface  
负责向用户展现信息，并且会解析用户行为，即常说的展现层。
- Application Layer  
应用层没有任何的业务逻辑代码，它很简单，它主要为程序提供任务处理。
- Domain Layer  
这一层包含有关领域的信息，是业务的核心，领域模型的状态都直接或间接（持久化至数据库）存储在这一层。
- Infrastructure Layer  
为其他层提供底层依赖操作。

# 模型关系图(Model-Driven Design)
![Model-Driven Design](p28.png)
## 服务(Services)
当我们在分析某一领域时，一直在尝试如何将信息转化为领域模型，但并非所有的点我们都能用Model来涵盖。对象应当有属性，状态和行为，但有时领域中有一些行为是无法映射到具体的对象中的，我们也不能强行将其放入在某一个模型对象中，而将其单独作为一个方法又没有地方，此时就需要服务

### 引用书上的例子
> 预定一艘船在一次航程中要运载的货物  
> 应用程序的任务：将每个货物(Cargo)与航程(Voyage)关联起来，记录并跟踪这种关系

那么它可能含有类似以下的代码：

```
//预定
public int makeBooking(Cargo cargo, Voyage voyage){
    int confirmation = orderConfirmationSequence.next();//订单确认序列
    voyage.addCargo(cargo, confirmation);
    return confirmation;
}
```

因为总有人临时取消订单，所以一般来说航运业的做法都是接受比运载能力多一点的货物，即“超订”。这是一个基本政策。

所以需求文档还有个要求：**允许10%超订**

```
//预定
public int makeBooking(Cargo cargo, Voyage voyage){
    //超订
    double maxBooking = voyage.capacity() * 1.1;
    if(voyage.bookedCargoSize() + cargo.size() > maxBooking) 
        return -1;
    
    int confirmation = orderConfirmationSequence.next();//订单确认序列
    voyage.addCargo(cargo, confirmation);
    return confirmation;
}
```
上面这段代码隐含了超订的规则，但是却跟预定绑在一起了，，同时也会让别人难以理解到其中的含义，当这个规则开始复杂化后则会让预定变得难以处理

我们可以尝试应策略模式改进:
```
//预定
public int makeBooking(Cargo cargo, Voyage voyage){
    //超订
    if(overbookingpolicy.isAllowed(cargo, voyage)) 
        return -1;
    
    int confirmation = orderConfirmationSequence.next();//订单确认序列
    voyage.addCargo(cargo, confirmation);
    return confirmation;
}
```
超订政策类(OverbookingPolicy)里面应该包含以下方法：
```
public boolean isAllowed(Cargo cargo, Voyage voyage){
    return (cargo.size + voyage.bookedCargoSize()) <= (voyage.capality() * 1.1);
}
```

这样我们就能够很明确的知道，超订是一项独特的政策，他的实现非常明确且独立。

## 工厂(Factories)  
在大型系统中，实体和聚合通常是很复杂的，这就导致了很难去通过构造器来创建对象。工厂就决解了这个问题，它把创建对象的细节封装起来，巧妙的实现了依赖反转。当然对聚合也适用（当建立了聚合根时，其他对象可以自动创建）

## 仓库(Repository)  
仓库封装了获取对象的逻辑，领域对象无须和底层数据库交互，它只需要从仓库中获取对象即可。仓库可以存储对象的引用，当一个对象被创建后，它可能会被存储到仓库中，那么下次就可以从仓库取。如果用户请求的数据没在仓库中，则会从数据库里取，这就减少了底层交互的次数

## 边界上下文(Bounded Context)
简单来说就是定义该领域模型的适用范围以及使用场景。

可以这样理解：
- 边界(Bounded)
即有边界的，表示领域模型有边界；这个边界定义了模型的适用范围，以便让负责该模型的团队知道什么该在模型中实现，什么不该；

- 上下文(Context)
即领域模型的产生是在某个上下文中产生的；上下文是一个和环境相关的概念。比如一次头脑风暴会议大家达成了一个模型，那这次会议的讨论就是该模型的上下文；比如某本书中谈到了某个东西，那这个东西的上下文就是那本书，那个东西要有意义的前提离不开那本书这个上下文；所以，上下文是模型有意义的前提；

## 实体(Entity) 和 值对象(ValueObject)    
一言蔽之，实体大致可以理解为我们传统开发的实体，但是他具有自己的行为，而不是POJO(只具有简单的getter,setter)；值对象是指描述一个实体某个属性的对象。
当然，这些都是需要在上面所说的BoundedContext被指定的前提下讨论。

举个例子：
在电商系统我们现在分成两个模块，一个商品模块，一个订单模块
订单对象中有收货地址(address)
```
Order {
    int id;
    String address; 
}
```
我们把address扩展开来
```
Order {
    int id;
    Address address; 
}

Address {
    String province;//省
    String city;//市
    String street;//街道
}
```
现在Address是一个对象了，但是我们不会认为他是一个实体，因为在这个订单模块中它只是描述了订单中的收货地址而已，仅仅只是order上的一个值，几个内部的值所组合出的抽象，你完全可以把它理解为是一个Map:
```
Order {
    int id;
    Map<String, String> address;
    
    //address Map{"province":"","city":"","street":""} 
}
```

这跟java中String对象非常类似，String对象是不会进行修改的，如果你将新的一串字符串重新赋值给一个String对象，实际上等于new了一个String，地址是变化了的，不再是同一个对象。

所以ValueObject有这样几个特点:
- 没有标识(唯一标识)
- 不可变(只读)
- 不具备生命周期
    
## 聚合(Aggregates) 和 聚合(Aggregate Root)  
`聚合`可以看作是多个实体之间的组合，而每个聚合都有一个根实体，叫`聚合根`。

在DDD当中，聚合外部想要访问聚合内的信息，必须通过`聚合根`进行访问。

- 如何识别聚合和聚合根？
首先一个边界上下文(Bounded Context)可能包含多个聚合，每个聚合都有一个聚合根。
    1. 找出哪些实体可能是聚合根
    2. 逐个分析每个聚合根的边界，即该聚合根应该聚合哪些实体或值对象
    3. 划分边界上下文

- 如何确定聚合边界？
边界的确定法则是根据不变性约束规则（Invariant）:
    - 聚合边界内必须具有哪些信息，如果没有这些信息就不能称为一个有效的聚合
    - 聚合内的某些对象的状态必须满足某个业务规则 

- 如何找到聚合根？
如果存在一个业务操作是完全面向某个实体，那么这个实体就可能是一个聚合根

- 例子分析
> Order（一 个订单）必须有对应的客户信息，否则就不能称为一个有效的Order；同理，Order对OrderLineItem有不变性约束，Order也必须至少有一个OrderLineItem(一条订单明细)，否 则就不能称为一个有效的Order；另外，Order中的任何OrderLineItem的数量都不能为0，否则认为该OrderLineItem是无效 的，同时可以推理出Order也可能是无效的。因为如果允许一个OrderLineItem的数量为0的话，就意味着可能会出现所有 OrderLineItem的数量都为0，这就导致整个Order的总价为0，这是没有任何意义的，是不允许的，从而导致Order无效；所以，必须要求 Order中所有的OrderLineItem的数量都不能为0；那么现在可以确定的是Order必须包含一些OrderLineItem，那么应该是通 过引用的方式还是ID关联的方式来表达这种包含关系呢？这就需要引出另外一个问题，那就是先要分析出是OrderLineItem是否是一个独立的聚合 根。回答了这个问题，那么根据上面的规则就知道应该用对象引用还是用ID关联了。那么OrderLineItem是否是一个独立的聚合根呢？因为聚合根意 味着是某个聚合的根，而聚合有代表着某个上下文边界，而一个上下文边界又代表着某个独立的业务场景，这个业务场景操作的唯一对象总是该上下文边界内的聚合 根。想到这里，我们就可以想想，有没有什么场景是会绕开订单直接对某个订单明细进行操作的。也就是在这种情况下，我们 是以OrderLineItem为主体，完全是在面向OrderLineItem在做业务操作。有这种业务场景吗？没有，我们对 OrderLineItem的所有的操作都是以Order为出发点，我们总是会面向整个Order在做业务操作，比如向Order中增加明细，修改 Order的某个明细对应的商品的购买数量，从Order中移除某个明细，等等类似操作，我们从来不会从OrderlineItem为出发点去执行一些业 务操作；另外，从生命周期的角度去理解，那么OrderLineItem离开Order没有任何存在的意义，也就是说OrderLineItem的生命周 期是从属于Order的。所以，我们可以很确信的回答，OrderLineItem是一个实体。

# CQRS 和 Event Souring
//TODO

![Event Souring](v2-7c6a1b0c101d8f0cf5e89716bfb4d6a1_hd.jpg)
![Event Souring + CQRS](v2-35249fb2693f44bbe4bf48ea6755c55c_hd.jpg)


