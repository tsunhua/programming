---
title: Spring
date: '2018-01-05T11:10:00.000Z'
tags:
  - Java
  - Spring
comments: true
---

# Spring & Spring Boot

## 概述

轻量级控制反转和面向切面的容器框架。

功能：

1. 使用基本的JavaBean代替EJB（Enterprise JavaBean）

优点：

1. 低侵入性
2. 高服用性
3. DI有效降低耦合度
4. AOP提供了通用任务的集中管理
5. ORM（对象实体映射）和DAO简化对数据库的访问
6. 高开放性，并不强制

核心：

![](https://www.ibm.com/developerworks/cn/java/wa-spring1/spring_framework.gif)

### IoC

Inversion of Control，控制反转，其另一个名字是依赖注入（Dependency Injection），就是有Ioc容器在运行期间，动态地将某种依赖关系注入到对象之中。

所以，DI和IoC是从不同角度描述的同一件事情，就是指通过引入IoC容器，利用依赖关系注入的方式，实现对象的解耦。

粘合剂，将对象之间的耦合交给IoC容器进行，从而达到解耦的目的。

### AOP

面向切面编程，从动态角度研究。

功能：将系统级别处理（日志管理、调试管理、事务管理、缓存等等）分离出来，模块化，而不影响业务逻辑。

专门用于处理系统中分布于各个模块中的交叉关注点的问题。 AOP 代理其实是由AOP框架动态生成的一个对象，该对象可作为目标对象使用。

AOP使用：

1. 定义普通业务组件
2. 定义切入点
3. 定义增强处理

代理对象的方法 = 被代理对象的方法 + 增强处理

AOP关键概念：

1. 横切性关注点（Cross Cut Concern）：一个独立的服务，不与其他业务逻辑耦合。可能遍布在系统的各个角落，也可能遍布在系统的处理流程之中。
2. 切面（Aspect）：一个关注点的模块化，生成对应的类，这个类就叫切面。切面可能会横切多个对象。
3. 通知（Advice）：对横切性关注点的具体实现。通知类型有：before、throw和after等。多数使用拦截器作为通知模型
4. 连接点（Joi Point）：程序执行中的某个特定点，比如某方法调用或处理异常时，也就是Advice在应用程序上执行的点或时机。
5. 切入点（Point Cut）：可以设定具体的方法上是否需要横切性关注点的实现。
6. 织入（Weave）：把切面连接到其他的应用程序类型或者对象上。
7. 目标对象（Target Object）：被一个或多个切面所通知的对象。
8. AOP代理（AOP Proxy）：AOP框架创建的对象，用来实现切面契约。
9. 引入（Introduction）：也被称为内容类型声明。声明额外的方法或者某个类型的字段。

## 开发环境搭建

直接通过IDEA新建一个Spring项目。

### Ioc注入

1. Spring通过`<null/>`注入null值
2. 通过`ref="xxBean"` 进行bean注入，通过`value="xxValue"` 进行值注入
3. 通过`<constructor-arg>` 注入构造参数值
4. 通过`<property>` 注入属性值
5. 通过`<list>`注入list数据，通过`<array>` 注入array数据，通过`<map>` 注入map数据，通过`<set>` 注入set数据
6. 设置`lazy-init="true"` 进行延迟初始化
7. bean的作用域有两种，分别是 `singleton` 和 `prototype`。singleton是默认的scope，其实现方式是通过注册表方式注册道单例缓存池。设置为prototype则每次向Spring容器请求获取Bean都返回一个全新的Bean。

### 基于注解的配置

开启方式：在spring的config文件的`<beans>` 下增加

```markup
<context:annotation-config/> 
<!--设置对指定的包名下的Spring注解生效-->
<context:component-scan base-package=”com.demo”>
```

使用：

| 注解 | 描述 | 应用域 |
| :--- | :--- | :--- |
| @Required | 进行依赖检查，只判断字段是否使用了setter注入，若无则抛出异常 | bean属性的setter方法 |
| @Autowired | 自动装配，默认按类型装配 | bean属性的setter方法、非setter方法、构造函数、属性 |
| @Qualifier | 实现类的类名 | 作为@Autowired的补充，当对应的属性有多个实现时，声明@Qualifier帮助其自动装配 |
| @Resource | 默认按名称装配，当找不到与名称匹配的bean才会按类型装配 | 属性（属于JSR-250的注解） |
| @Component、@Service、@Controller、@Repository | @Component用于说明一个类是Spring容器管理的类，@Service、@Controller、@Repository是@Component的细化，分别代表服务层、控制层和持久层 | 类 |

## Spring Boot

1. 推出时间：从Maven仓库的时间看是2016.7.28
2. 目的：摆脱大量的XML配置文件以及复杂的Bean依赖关系，快速、敏捷地开发新一代基于Spring框架的应用程序
3. 思想：约定优于配置（convention over configuration）
4. 框架功能：集成大量常用第三库配置（Jackson、JDBC、Mongo、Redis、Mail等）

### 创建定时任务

1. 在Spring Boot 的主类加入 `@EnableScheduling` 注解，启用定时任务的配置；

   ```text
   @SpringBootApplication
   @EnableScheduling
   public class Application {
     public static void main(String[] args) {
       SpringApplication.run(Application.class, args);
     }
   }
   ```

2. 使用 `@Scheduled(fixedRate = 1000)` 注解实现类需要定时执行的方法。

   ```text
   @Component
   public class ScheduledTasks implements Runnable{
       private static final SimpleDateFormat dateFormat = new SimpleDateFormat("HH:mm:ss");
   ​
       @Override
       @Scheduled(fixedRate = 1000)
       public void run() {
           System.out.println("Current: " + dateFormat.format(new Date()));
       }
   }
   ```

## Q&A

### 如何将配置 SpringApplication 的配置分离到单独的配置类

一般情况下，我们使用如下方式启动一个 Spring。

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
       SpringApplication.run(Application.class);
    }
}
```

但是，如果要把配置分离呢？

（1）编写 Spring 配置类，注意要指定组件扫描位置，因为通常配置类的位置跟 Application 类的位置不一致，而**默认情况下Spring 只扫描配置类所在包及其子包的组件**。

```java
@SpringBootApplication
@ComponentScan(basePackages = {"com.example"})
public class SpringConfig {
}
```

（2）修改之前的 Application，移除注解，传入 SpringConfig。

```java
public class Application {
    public static void main(String[] args) {
       SpringApplication.run(SpringConfig.class);
    }
}
```

分离后的好处是，可以将配置统一管理，以及更灵活地在启动 Spring，特别是有必须在 Spring 启动前 Ready 的依赖组件。

## 参考

1. [springboot快速入门及@SpringBootApplication注解分析](https://www.jianshu.com/p/4e1cab2d8431)

