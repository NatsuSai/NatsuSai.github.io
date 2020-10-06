---
title: Reactor中的上下文
date: 2020-09-08 22:53:22
categories: 
    - Java
    - Reactor
tags: 
    - Java
    - Reactor
    - Webflux
    - Spring
comments: true
thumbnail: /gallery/machi.png
---
由于参与的项目当中决定用 [spring-cloud-gateway](https://github.com/spring-cloud/spring-cloud-gateway) 作为网关，所以自然就碰上了Reactor当中获取上下文的问题。    

这里我是遇到了 I18N 需要获取上下文中 Locale 的问题，所以我就以这个问题举例，提供其中一种上下文的解决方法。  
<!--more-->

## What happened
在使用 I18N 进行国际化时，Servlet 框架下我们通常会使用 ThreadLocal 进行储存解析请求得到上下文的 Locale 对象，以判断当前请求需要的是什么语言。  
但在 Reactor 框架下，由于一个请求整个流程下来，线程都在不断的进行切换，所以 ThreadLocal 自然也就失去上下文储存对象的能力。

经过简单的谷歌查询，我只能想两种处理方法：  

1. 从 controller 获取 Locale 或者任意能够得到 Exchange（类似 Request）的方法开始一路传递下去。
2. 直接将 I18N 的 key 传入自定义异常对象当中，然后在统一异常处理类当中获取 Exchange 再做进一步的国际化操作。
    
但他们都不够优雅，而且其中弊端也很明显，一个是需要无限传递变量，一个是只能限制再异常处理类进行处理，理所当然就无法添加国际化的变量。

## How should I do
### 如何取值
令人喜悦的是，Spring 其实已经写了一个获取上下文的例子了，它就是`ReactiveSecurityContextHolder`，对应 Servlet 当中的`SecurityContextHolder`。

ReactiveSecurityContextHolder#getContext
```
private static final Class<?> SECURITY_CONTEXT_KEY = SecurityContext.class;

public static Mono<SecurityContext> getContext() {
        return Mono.subscriberContext()
            .filter( c -> c.hasKey(SECURITY_CONTEXT_KEY))       
            .flatMap( c-> c.<Mono<SecurityContext>>get(SECURITY_CONTEXT_KEY));
}
```
可以看到，通过`Mono.subscriberContext()`，我们可以得到一个上下文对象，然后他先判断是否包含给出的key值，包含则获取值。  
这里需要注意，如果直接获取值会抛出异常。   


### 如何赋值    
当我查看`withSecurityContext`方法时，其注释告诉我是用来创建一个包含`SecurityContext`的 Reactor 上下文对象（Context）并可被用于与其他上下文对象（Context）进行合并。  

所以当我们查看有什么方法调用它时，就会发现`ReactorContextWebFilter`这个过滤器。
```
public class ReactorContextWebFilter implements WebFilter {
    private final ServerSecurityContextRepository repository;

    public ReactorContextWebFilter(ServerSecurityContextRepository repository) {
        Assert.notNull(repository, "repository cannot be null");
        this.repository = repository;
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        return chain.filter(exchange)
            .subscriberContext(c -> c.hasKey(SecurityContext.class) ? c :
                withSecurityContext(c, exchange)
            );
    }

    private Context withSecurityContext(Context mainContext, ServerWebExchange exchange) {
        return mainContext.putAll(this.repository.load(exchange)
            .as(ReactiveSecurityContextHolder::withSecurityContext));
    }
}
```
ServerSecurityContextRepository#load
```
public Mono<SecurityContext> load(ServerWebExchange exchange) {
    return exchange.getSession()
        .map(WebSession::getAttributes)
        .flatMap( attrs -> {
            SecurityContext context = (SecurityContext) attrs.get(this.springSecurityContextAttrName);
            return Mono.justOrEmpty(context);
        });
}
```
ReactiveSecurityContextHolder#withSecurityContext
```
public static Context withSecurityContext(Mono<? extends SecurityContext> securityContext) {
    return Context.of(SECURITY_CONTEXT_KEY, securityContext);
}
```
通过阅读上面的代码可得知，上面的过滤器通过`ServerSecurityContextRepository`解析请求中的 Security 上下文，通过`ReactiveSecurityContextHolder`生成 Securtiy 上下文对象并返回。

## Finally
我们可以仿照着写一个从请求对象当中解析出一个 Locale 对象并放入上下文当中（不确定是不是上下文，但是Reactor确实是用这种办法实现了上下文的功能）。  
```
// 将 Locale 放到上下文中
public class ReactorLocaleContextWebFilter implements WebFilter {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        return chain.filter(exchange)
            .subscriberContext(c -> c.hasKey(Locale.class) ? c :
                withLocaleContext(c, exchange)
            );
    }

    private Context withLocaleContext(Context mainContext, ServerWebExchange exchange) {
        //解析请求获取 Locale 
        Locale locale = getLocale(exchcange);
        return mainContext.putAll(Context.of(Locale.class, Mono.justOrEmpty(locale)));
    }
}

//获取 Locale
Mono.subscriberContext()
            .filter( c -> c.hasKey(Locale.class))       
            .flatMap( c-> c.<Mono<Locale>>get(Locale.class));
```


具体实现我日后会写一个demo。~~咕咕咕~~
    
