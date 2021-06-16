---
title: Java并发编程——双检锁单例模式
date: 2021-06-16 11:19:12
categories:
    - Java并发编程
tags:
    - Java
    - J.U.C
comments: true
index_img: /gallery/64535234_p0.png
banner_img: /gallery/64535234_p0.png
---
记录一下双检锁单例模式是怎么一回事
<!--more-->
> 封面 pixiv id 64535234

## 0x01
懒汉单例模式是无法保证线程安全的
```java
public class Singleton {
    
    private static Singleton singleton;

    private Singleton() {
    }

    public static Singleton getInstance() {
        if (singleton == null) { 
            //多个线程同时通过if判断新建实例，违反单例的初衷
            singleton = new Singleton();
        }

        return singleton;
    }
}
```
即当线程A通过if判断，但还未创建实例，此时线程B也能够通过if判断，那么就会对重复创建实例违反单例的初衷。

## 0x02
那么我们为了保证线程安全，引入synchronized进行同步处理
```java
public class Singleton {
    
    private static Singleton singleton;

    private Singleton() {
    }

    public static synchronized Singleton getInstance() {
        if (singleton == null) {
            //此时最多只会有一个线程进入if判断中
            singleton = new Singleton();
        }
        return singleton;
    }
}
```
此时就只有一个线程能够获取锁进入到if判断创建实例了。

---
> - synchronized 偏向锁，自旋锁，轻量级锁，重量级锁
> 
> 通过 synchronized 加锁，第一个线程获取的锁为偏向锁，这时有其他线程参与锁竞争，升级为轻量级锁，其他线程通过循环的方式尝试获得锁，称自旋锁。若果自旋的次数达到一定的阈值，则升级为重量级锁。
> 
> 需要注意的是，在第二个线程获取锁时，会先判断第一个线程是否仍然存活，如果不存活，不会升级为轻量级锁。

## 0x03
可是现在由于同步处理导致每次获取实例都需要竞争获取锁导致效率非常低下，
所以我们应该在最外面做一次if判断来让大多数时候直接return实例而不是进行锁的竞争。
```java
public class Singleton {
    
    private static Singleton singleton;

    private Singleton() {
    }

    public static  Singleton getInstance() {
        if (singleton == null) { //为了快速返回而不进入锁的竞争
            synchronized (Singleton.class) { //以当前类作为锁
                if (singleton == null) {
                    //此时最多只会有一个线程进入if判断中
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
而这就是双检锁（双重检查锁，Double check lock，DCL）。

## 0x04
但这里其实还是没有彻底解决多线程的问题，因为new Object分为三个步骤：
- 分配内存空间 
- 初始化对象信息
- 将内存空间引用赋值给变量

如果这当中指令重排了，在还没有初始化对象的时候就把地址赋值给了变量，此时在最外层的if判断变量不为空，因为有地址， 
这时候就会拿到一个未经初始化的变量。  
所以我们还需要用volatile修饰变量
```java
public class Singleton {
    
    private static volatile Singleton singleton; //避免指令重排序

    private Singleton() {
    }

    public static  Singleton getInstance() {
        if (singleton == null) { //为了快速返回而不进入锁的竞争
            synchronized (Singleton.class) { //以当前类作为锁
                if (singleton == null) {
                    //此时最多只会有一个线程进入if判断中
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```
---
> - volatile  
> 
>![volatile](volatile.png)
> 
> 在多线程环境下，保证变量的可见性。使用了 volatile 修饰变量后，在变量修改后会立即同步到主存中，每次用这个变量前会从主存刷新。  
> 
> 禁止 JVM 指令重排序。
