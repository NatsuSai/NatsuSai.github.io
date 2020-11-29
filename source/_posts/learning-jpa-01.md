---
title: 学习JPA笔记——使用MetaModel实现Typesafe
date: 2020-10-06 07:25:51
categories: 
    - Java
tags:
    - Java
    - ORM
    - JPA
    - Hibernate
comments: true
index_img: /gallery/learning-jpa-01.jpg
banner_img: /gallery/learning-jpa-01.jpg
---
本篇主要说 Typesafe，下一篇讲一下 JPA 构建复杂查询。项目~~极简~~代码[在这](https://github.com/NatsuSai/spring-data-jpa-demo)
<!--more-->
> 封面：动画「ガーリー・エアフォース」#08的 EDCard，算是原作插画师[@遠坂あさぎ](https://twitter.com/asagi_0398)的贺图，每集一张。
> 有趣的是因为最高机密的特性导致男主看到的样子和幼馴染一样，而此时他的幼馴染正生着气，所以标题叫不高兴的最高机密w

最近看了[@hantsy](https://github.com/hantsy/helidon-sample)大大在V站的帖子（[这里](https://www.v2ex.com/t/688051#reply70)、[这里](https://www.v2ex.com/t/688051#reply70)），
就开始心血来潮想要看看 JPA 怎么玩，另外就是大大所说的 Typesafe 要这么实现。

> 注意：这里不会对JPA大多的基础知识进行说明，文章本意是做一次笔记，必要时请充分发挥自主能动性进行查找学习

## Typesafe
我的理解是不要那种无法编译时无法检验出错误或者 IDE 无法帮助我们检验错误的字符串，而这里比较突出的就是字段名。

我司其实也是内部写了一套 [orm](https://github.com/tanqimin/MyFavsORM)，只是基本不在意 Typesafe，而更加注重方便直接编写复杂 sql 而已。

而没有编译时的检测或者是 IDE 的检测，就难免出现 Typo，更加糟糕的是后期维护时的修改会造成一种我还有哪里用到了这个字段的尴尬状况，一旦遗漏就只能等运行时才可以检测出了。

为了解决这一状况，我们可以用到 MetaModel 生成器，例如 Hibernate 就有相应的生成器`jpamodelgen`（这类 MetaModel 生成器是为了实现 JPA2.0 标准的，具体我没有细查）

除了 Hibernate 以外，EclipseLink 也有这类生成器，各位可以自己去玩玩。我由于直接用 SpringDataJpa，而 spring 默认使用 Hibernate，所以就没有折腾别的了。

### Getting Started
下面是我整合了Lombok生成器的配置，仅供参考。也可以看看[这篇文章](https://docs.jboss.org/hibernate/stable/jpamodelgen/reference/en-US/html_single/)，有多种玩法。
```xml
<project>
    ...
    <dependencies>
        ...
        <dependency>
          <groupId>org.hibernate</groupId>
          <artifactId>hibernate-jpamodelgen</artifactId>
          <version>5.4.21.Final</version>
        </dependency>

        <!-- 如果不是用SpringDataJpa的话，需要额外引入下面的依赖 -->
        <dependency>
          <groupId>jakarta.persistence</groupId>
          <artifactId>jakarta.persistence-api</artifactId>
          <version>2.2.3</version>
        </dependency>
    </dependencies>
    ...
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <annotationProcessorPaths>
                        <path>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                            <version>${lombok.version}</version>
                        </path>
                    </annotationProcessorPaths >
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.bsc.maven</groupId>
                <artifactId>maven-processor-plugin</artifactId>
                <version>4.3</version>
                <executions>
                    <execution>
                        <id>process</id>
                        <goals>
                            <goal>process</goal>
                        </goals>
                        <phase>generate-sources</phase>
                        <configuration>
                            <processors>
                                <processor>org.hibernate.jpamodelgen.JPAMetaModelEntityProcessor</processor>
                            </processors>
                        </configuration>
                    </execution>
                </executions>
                <dependencies>
                    <dependency>
                        <groupId>org.hibernate</groupId>
                        <artifactId>hibernate-jpamodelgen</artifactId>
                        <version>5.4.21.Final</version>
                    </dependency>
                </dependencies>
            </plugin>
        </plugins>
    </build>
</project>
```
添加完后，在实体类添加@Entity和@Id的注解使生成器生效，例如：
```java
@Entity
@Data
public class Post implements Serializable {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    int id;
    String title;
    String content;
    @Enumerated(EnumType.STRING)
    Status status = Status.DRAFT;
    LocalDateTime createdAt;
    @Transient
    String excludeColumn;

    static enum Status{DRAFT, PUBLISHED}

    public static Post of(String title, String content) {
        Post post = new Post();
        post.setCreatedAt(LocalDateTime.now());
        post.setTitle(title);
        post.setContent(content);

        return post;
    }
}
```
添加完后对代码进行编译，生成器则会找到该注解的类生成这样的类：
```java
@Generated(value = "org.hibernate.jpamodelgen.JPAMetaModelEntityProcessor")
@StaticMetamodel(Post.class)
public abstract class Post_ {

	public static volatile SingularAttribute<Post, LocalDateTime> createdAt;
	public static volatile SingularAttribute<Post, Integer> id;
	public static volatile SingularAttribute<Post, String> title;
	public static volatile SingularAttribute<Post, String> content;
	public static volatile SingularAttribute<Post, Status> status;

	public static final String CREATED_AT = "createdAt";
	public static final String ID = "id";
	public static final String TITLE = "title";
	public static final String CONTENT = "content";
	public static final String STATUS = "status";

}
```
而当我们运行程序时，SingularAttribute 类型的对象则会被自动赋值，之后在调用 JPA 的 API 时则可以作为参数传入，而不是传入字符串了。
调用时看起来是这样子的：
```
repository.findAll((Specification<Post>) (root, criteriaQuery, builder) -> builder.like(root.get(Post_.title), "%Test%"))
```
~~我说什么来着，不用字符串~~
其实这里就不太需要追求这些，而字段这些是带关联性的，会在好多个地方出现，有必要对其进行检测。对于人类而言，检测总是会犯错，所以这些最好是交由机器来帮忙，也能够让我们更加关注业务。

## Reference
[helidon-sample](https://github.com/hantsy/helidon-sample/blob/master/mp-jpa/src/main/java/com/example/PostRepository.java) @ [hantsy](https://github.com/hantsy)
[Hibernate JPA 2 Metamodel Generator](https://docs.jboss.org/hibernate/stable/jpamodelgen/reference/en-US/html_single/)
