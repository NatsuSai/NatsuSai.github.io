---
title: 学习JPA笔记——构建复杂查询
date: 2020-10-06 09:25:19
categories: 
    - Java
tags:
    - Java
    - ORM
    - JPA
    - Hibernate
comments: true
index_img: /gallery/learning-jpa-02.jpg
banner_img: /gallery/learning-jpa-02.jpg
---
封面：同上篇，这次前景就是最高机密 Viper Zero。~~嗯，没啥问题，每集一张，只是PS了~~
<!--more-->
---
这里介绍两种 JPA 做复杂查询的方法，一个是用 SpringDataJPA 实现， 一个是用 Java EE 实现。 

这里还是稍微提一下这几个东西之间的关系。
首先JPA是一种规范，Java EE 中有把这种规范抽象出来的接口，具体实现是看用的什么框架，可以是 Hibernate 或者是 EclipseLink 等。
而这之中 Spring 又对 Java EE 中的接口再次封装，以更好的整合进 Spring 体系当中，但 SpringDataJPA 仍然是个抽象，具体实现仍然是看选型的框架。
但日常中，由于 SpringDataJPA 默认是 Hibernate 实现，所以一般场合基本相当于 Hibernate。

## SpringDataJPA
SpringDataJPA 的复杂查询除了直接写 sql，按照规则定义 Repository 接口方法以外，还可以使用Specification做查询。

### Specification
这是SpringDataJPA抽象出的一个接口，故并不一定通用于其他JPA的实现。
该接口重点在于`toPredicate`方法，该方法将创建一个 where 语句对象。
```
public interface Specification<T> extends Serializable {

    ...

    /**
     * Creates a WHERE clause for a query of the referenced entity in form of a  Predicate for the given
     * Root and CriteriaQuery.
     */
    @Nullable
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder);
}
```
这里我只做简要说明，以便于理解，仅供参考。
- root：
    一般指实体类本身，包装成 Root 对象, 可由query.from(Post.class)得到`Root<Post>`，Java EE 会提到。
- query：
    sql语句对象，一般在此方法内部不做调用，Java EE 会给一些用于理解的调用。
- criteriaBuilder：
    用于构建条件语句。

> 个人建议是不要管我说的这些，真要去理解就看源码注释，或者看下面 Java EE 的代码，更能够理解。
    
### Getting Started
首先我们创建一个 Repository 接口，并继承`JpaSpecificationExecutor<T>`以获得复杂查询的能力。
```java
public interface PostRepository extends JpaSpecificationExecutor<Post> {
}
```
然后我们运用 java8 所带来的新特性，使用 lambda 构建一个匿名 Specification 的实现类，并实现 toPredicate 方法。
```
repository.findAll(
    (Specification<Post>) (root, criteriaQuery, builder) -> { 
        // where title like '%Test%';
        return builder.like(root.get(Post_.title), "%Test%")
        // 如果是多个条件，例如 where title like '%Test%' and content like '%Test%' and id < 10;
        return builder.and(builder.like(root.get(Post_.title), "%Test%"), builder.like(root.get(Post_.content), "%Test%"), builder.le(root.get(Post_.id), 10))
        // 而如果我们没有做 Typesafe，那么就会变成这样
        return builder.like(root.get("title"), "%Test%")
    }
);
```
使用起来其实没什么困难，基本举一反三，其他的复杂查询我暂时没研究，主要是觉得可以避开用别的方法操作，或者提到程序中操作。

## Java EE
实际上，SpringDataJPA 是对 Java EE 原本的 JPA 抽象再次包装了一层，所以这个可以说是原汁原味了。

### Getting Stated
由于我主要使用 SpringDataJPA 所以摘抄了一段代码，[出处](https://github.com/hantsy/helidon-sample/blob/master/mp-jpa/src/main/java/com/example/PostRepository.java)。
```
//需要对此进行注入
private final EntityManage entityManager;

String q;
int offset, limit;

CriteriaBuilder cb = this.entityManager.getCriteriaBuilder();
// create query
CriteriaQuery<Post> query = cb.createQuery(Post.class);
// set the root class
Root<Post> root = query.from(Post.class);

// if keyword is provided
if (q != null && !q.trim().isEmpty()) {
    // 这里其实就是上面 toPredicate 返回的对象作为参数传入where方法当中，所以里面就和上面的实现没有什么太大区别。
    query.where(
            cb.or(
                    cb.like(root.get(Post_.title), "%" + q + "%"),
                    cb.like(root.get(Post_.content), "%" + q + "%")
            )
    );
}
//perform query
return this.entityManager.createQuery(query)
        .setFirstResult(offset)
        .setMaxResults(limit)
        .getResultList();
```

- EntityManager
    实体管理类，用于与持久化上下文进行互动，核心类。
    
其他几个 Root、CriteriaQuery、CriteriaBuilder 作用同上，毕竟spring只是做了封装。
看完上面代码，大致就能够了解清楚这几个类分别是怎么使用的了，总体来说其实比上面spring的实现所接触到的东西更加全面一些，也能够理解这几个类互相是怎么作用的了。

## Reference
[helidon-sample](https://github.com/hantsy/helidon-sample/blob/master/mp-jpa/src/main/java/com/example/PostRepository.java) @ [hantsy](https://github.com/hantsy)
