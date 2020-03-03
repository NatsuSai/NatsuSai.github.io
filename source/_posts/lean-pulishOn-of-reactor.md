---
title: Reactor处理阻塞问题笔记
date: 2020-03-03 09:53:22
categories: 
    - Java
tags: 
    - Java
    - Reactor
comments: true
thumbnail: /gallery/machi.png
---
其实本来想要记录问题的过程，但奈何自己也说不太好XD
<!--more-->
## What

由于接触Vert.x以及阅读其文档后，了解到异步编程下是不能够阻塞主线程的，不然异步将失去意义。

我们需要做的是将这些阻塞线程移到其他线程进行处理。

## How

利用Mono或是Flux的PublishOn方法将之后调用的方法都移动到其他线程进行处理。

- `publishOn`

  ```java
  public final Mono<T> publishOn(Scheduler scheduler);
  public final Flux<T> publishOn(Scheduler scheduler);
  ```

  其中`Scheduler`可用`Schedulers.parallel()`或`Schedulers.single()`进项创建或是其他方法，其中`single`和`parallel`是有一些区别的。

  - `single`

    这一条调用链下不会同时执行，并且只有这条调用链执行完成后才会再次被调用

  - `parallel`

    与上面相反，调用链会在同时执行

  下面是测试代码

  ~~本人只是刚开始玩reactor，程序写的很蹩脚XD~~

  ```java
  @Test
    public void test() throws IOException {
      AtomicReference<Employee> employeeAR = new AtomicReference<>(); //<1>
      Scheduler scheduler = Schedulers.single(); //<2>
      for (int i = 0; i < 5; i++) {
        int finalI = i;
        Mono.just(1)
            .publishOn(scheduler)
            .map(x -> {
              try {
                Thread.sleep(1000);
                System.out.println(finalI + "-" + Thread.currentThread()
                    .getName() + "-A"); //<3>
              } catch (InterruptedException e) {
                e.printStackTrace();
              }
              Employee emp = buildEmployee();
              employeeAR.set(emp); //<1>
              return emp;
            })
            .map(x -> {
              try {
                System.out.println(finalI + "-" + Thread.currentThread()
                    .getName() + "-B"); //<3>
                Thread.sleep(1000);
              } catch (InterruptedException e) {
                e.printStackTrace();
              }
              return x;
            })
            .map(x -> {
              System.out.println(finalI + "-" + Thread.currentThread()
                  .getName() + "-C"); //<3>
              return employeeAR.get().getAccountId(); //<1>
            })
            .subscribe(System.out::println);
  
        System.out.println(Thread.currentThread()
            .getName());  //<3>
      }
      char c = (char) System.in.read();
      System.out.println("your char is: " + c);
    }
  ```

  - <1> 因为涉及到匿名方法中的变量的再次调用，所以用到AtomicReference进行储存。

  下面是`single`的运行结果

  ```
  main
  main
  main
  main
  main
  0-single-1-A
  0-single-1-B
  0-single-1-C
  123
  1-single-1-A
  1-single-1-B
  1-single-1-C
  123
  2-single-1-A
  2-single-1-B
  2-single-1-C
  123
  3-single-1-A
  3-single-1-B
  3-single-1-C
  123
  4-single-1-A
  4-single-1-B
  4-single-1-C
  123
  ```

  将<2>中`single`改为`parallel`

  ```
  main
  main
  main
  main
  main
  0-parallel-1-A
  3-parallel-4-A
  2-parallel-3-A
  4-parallel-5-A
  0-parallel-1-B
  3-parallel-4-B
  2-parallel-3-B
  1-parallel-2-A
  1-parallel-2-B
  4-parallel-5-B
  0-parallel-1-C
  1-parallel-2-C
  3-parallel-4-C
  4-parallel-5-C
  2-parallel-3-C
  123
  123
  123
  123
  123
  ```

  观察代码中<3>，我们可以发现调用链当中是按照顺序执行的（我最开始以为会平行执行调用链中的方法，但并不是），而且主线程也没有被阻塞，能够快速输出当前线程名称，由此可见已经达到我们最初的目的了——不阻塞主线程。

## Why

在这里我用的是Spring WebFlux，而其中会用到netty，其中有一个Eventloop模块，这是由单个线程运行的模块，这个单线程就是由我们程序所运行的主线程来担当。

Eventloop会重复检查当前有没有事件产生，若有则会接收该事件并运行相应的事件响应，也就是发布订阅模式，而如果我们在其中一个调用该事件的响应方法中等待（阻塞）过久，就会导致我们无法快速处理后续产生的事件，只能够加多线程进行快速处理，这就又回到了非异步编程当中去了。

所以能够快速响应才能够体现出异步编程的优势。

## Reference
- [Is there a standard way to solve blocking that must happen.](https://github.com/reactor/reactor-core/issues/1756)
- [How to handle blocking calls when using reactor in a JAX-RS-powered server?](https://stackoverflow.com/questions/56706308/how-to-handle-blocking-calls-when-using-reactor-in-a-jax-rs-powered-server)

## TODO

- [ ] 测试嵌套调用publishiOn是什么情况  
- [ ] 是否是调用一次publishOn后，后面的链式调用都是在另一条线程，是否需要再次调用一次pubulishOn保证之后的一次阻塞操作也不在主线程当中  
- [ ] 补充详细Evenloop  
- [ ] 寻找更加优雅的方式，或者看看这种链式调用是不是也是一个不太好的地方  


