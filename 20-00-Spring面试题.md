[TOC]

## 1.知识点

### 1.1 aware

Spring提供的Aware接口能让Bean感知Spring容器的存在，即让Bean在初始化就可以使用Spring容器所提供的资源。

| 接口                           |                  作用                   | 备注 |
| ------------------------------ | :-------------------------------------: | ---: |
| ApplicationContextAware        | 能获取Application Context调用容器的服务 | 常用 |
| ApplicationEventPublisherAware |    应用事件发布器，可以用来发布事件     |      |
| BeanClassLoaderAware           |      能获取加载当前Bean的类加载器       |      |
| BeanFactoryAware               |    能获取Bean Factory调用容器的服务     |      |
| BeanNameAware                  |          能获取当前Bean的名称           |      |
| EnvironmentAware               |      能获取当前容器的环境属性信息       | 常用 |
| MessageSourceAware             |          能获取国际化文本信息           | 常用 |
| ResourceLoaderAware            |       获取资源加载器读取资源文件        |      |



### 1.2 Spring ： ApplicationContext和BeanFactory

首先要明白两点：

1.  `BeanFactory`和`ApplicationContext`都是容器，也就是放置所有`Java Bean`对象的地方，而且它们的关系是`ApplicationContext`继承自 `BeanFactory`。
2.  `BeanFactory`的最重要的一个方法是`getBean()`，调用这个方法会返回给你一个已经完全初始化好的对应的`bean`对象，不需要你自己去硬编码对象的创建逻辑和创建过程，这样做的一个好处是一个类就能完全专注于自己的业务逻辑，而不用操心其它的“杂事”。还有一个好处是可以在一个类不知情的情况下把它的依赖类换掉，而不用修改它的代码，这样可以使编程更加简单、不易出错。

至于它们之间的区别其实只有一句话：`ApplicationContext`是`BeanFactory`的加强版，它提供了许多自动化的功能，这样你就不用在编写程序时自己去实现，比如说`AOP`。

1.  `ApplicationContext`在创建时会把配置文件中的或者扫描到的`bean`已经创建完成，而`BeanFactory`在创建时只会生成`BeanDefinition`，然后在你手动调用`getBean()`函数的时候才会根据`BeanDefinition`来创建这个`bean`。其实里面也没有什么黑科技，可以看做是`ApplicationContext`对于所有的`bean`都先帮你调用了一下`getBean()`函数，当然有`@Lazy`注解或者非单例的除外。
2.  在1的基础上，`ApplicationContext`会自动注入几个和环境变量有关的`bean`，比如`environment`。`ApplicationContext`还会自动创建并注册`BeanPostProcessor`，以便在调用`getBean()`的时候对目标`bean`进行处理。这是一个重要的功能，`AOP`就是在`BeanPostProcessor`的函数里面实现了创建代理的功能。
3.  `ApplicationContext`提供了发布事件的功能，可以调用`ApplicationContext`提供的接口来发布事件并监听，同样地，`ApplicationContext`也会自动创建并注册监听器。

简而言之，`ApplicationContext`就是为了实现`BeanFactory`的功能，所以需要处理各种各样的情况，以`AnnotationConfigApplicationContext`为例，需要先注册配置类，然后从配置类的注解中找有没有`@Import`了其他的类，然后再进行相应的处理。比如`@EnableAspectJAutoProxy`注解的定义上有一行代码是`@Import(AspectJAutoProxyRegistrar.class)`，所以就需要找到`AspectJAutoProxyRegistrar`类，由于它实现了`ImportBeanDefinitionRegistrar`接口，就调用它的`registerBeanDefinitions`方法把`org.springframework.aop.config.internalAutoProxyCreator`类作为一个`BeanDefinition`也注册到容器中去，接下来根据这个`BeanDefinition`创建一个`AnnotationAwareAspectJAutoProxyCreator`类的对象来实现`AOP`的功能。

### 1.3 事务传播表

传播特性名称	说明
PROPAGATION_REQUIRED	如果当前没有事物，则新建一个事物；如果已经存在一个事物，则加入到这个事物中
PROPAGATION_SUPPORTS	支持当前事物，如果当前没有事物，则以非事物方式执行
PROPAGATION_MANDATORY	使用当前事物，如果当前没有事物，则抛出异常
PROPAGATION_REQUIRES_NEW	新建事物，如果当前已经存在事物，则挂起当前事物
PROPAGATION_NOT_SUPPORTED	以非事物方式执行，如果当前存在事物，则挂起当前事物
PROPAGATION_NEVER	以非事物方式执行，如果当前存在事物，则抛出异常
PROPAGATION_NESTED	如果当前存在事物，则在嵌套事物内执行；如果当前没有事物，则与PROPAGATION_REQUIRED传播特性相同

### 1.4 cglb代理

`CGLIB`是针对类来实现代理的，`原理是对指定的业务类生成一个子类，并覆盖其中业务方法实现代理`。因为采用的是继承，所以不能对final修饰的类进行代理。 在使用的时候需要引入`cglib`和`asm`的`jar`包，并实现`MethodInterceptor`接口。

### 1.5 JDK代理

JDK动态代理所用到的`代理类`在程序调用到`代理类对象`时才由JVM真正创建，`JVM根据传进来的业务实现类对象以及方法名 ，动态地创建了一个代理类的class文件并被字节码引擎执行，然后通过该代理类对象进行方法调用`。

### 1.6 Spring解决bean之间的循环依赖

循环依赖种类

1. 构造函数的循环依赖。这种依赖显然是解决不了的。在 A 的构造方法中依赖 B，在 B 的构造方法中依赖 A 是不行的。
2. 非单例Bean的循环依赖。这种依赖也是解决不了的。
3. 单例Bean的循环依赖。本文介绍的就是如何解决单例Bean的循环依赖的问题。

Spring 在处理属性循环依赖的情况时主要是通过延迟设置来解决的，当bean被实例化后，此时还没有进行依赖注入，当进行依赖注入的时候，发现依赖的bean已经处于创建中了，那么通过可以设置依赖，虽然依赖的bean没有初始化完成，但是这个可以后面再进行设置，至少得先建立bean之间的引用。

Spring为了解决单例的循环依赖问题，使用了三级缓存。

### 1.7 IOC

依赖注入（`Dependecy Injection`）和控制反转（`Inversion of Control`）是同一个概念，具体的讲：当某个角色需要另外一个角色协助的时候，在传统的程序设计过程中，通常由调用者来创建被调用者的实例。

但在spring中 创建被调用者的工作不再由调用者来完成，因此称为控制反转。创建被调用者的工作由spring来完成，然后注入调用者 ，因此也称为依赖注入。

IoC又叫依赖注入（DI）。它描述了对象的定义和依赖的一个过程，也就是说，依赖的对象通过构造参数、工厂方法参数或者属性注入，当对象实例化后依赖的对象才被创建，当创建bean后容器注入这些依赖对象。这个过程基本上是反向的，因此命名为控制反转（IoC），它通过直接使用构造类来控制实例化，或者定义它们之间的依赖关系，或者类似于服务定位模式的一种机制。

### 1.8 BeanFactory

BeanFactory，以Factory结尾，表示它是一个工厂类(接口)，用于管理Bean的一个工厂。在Spring中，BeanFactory是IOC容器的核心接口，它的职责包括：实例化、定位、配置应用程序中的对象及建立这些对象间的依赖。

BeanFactory定义了 IOC 容器的最基本形式，并提供了 IOC 容器应遵守的的最基本的接口，也就是 Spring IOC 所遵守的最底层和最基本的编程规范。在 Spring 代码中，BeanFactory 只是个接口，并不是 IOC 容器的具体实现，但是 Spring 容器给出了很多种实现，如 DefaultListableBeanFactory 、XmlBeanFactory 、 ApplicationContext、ClassPathXmlApplicationContext 等，都是附加了某种功能的实现。

在Spring中，所有的Bean都是由BeanFactory(也就是IOC容器)来进行管理的。但对FactoryBean而言，这个Bean不是简单的Bean，而是一个能生产或者修饰对象生成的工厂Bean,它的实现与设计模式中的工厂模式和修饰器模式类似

### 1.9 FactoryBean

FactoryBean以Bean结尾，表示它是一个Bean，不同于普通Bean的是：它是实现了FactoryBean接口的Bean，根据该Bean的ID从BeanFactory中获取的实际上是FactoryBean的getObject()返回的对象，而不是FactoryBean本身，如果要获取FactoryBean对象，请在id前面加一个&符号来获取。

FactoryBean是一个接口，当在IOC容器中的Bean实现了FactoryBean后，通过getBean(String BeanName)获取到的Bean对象并不是FactoryBean的实现类对象，而是这个实现类中的getObject()方法返回的对象。要想获取FactoryBean的实现类，就要getBean(&BeanName)，在BeanName之前加上&。

这是个特殊的 Bean 他是个工厂 Bean，可以产生 Bean 的 Bean

### 1.10 监听器

1. 拦截器(interceptor)：依赖于web框架，基于Java的反射机制，属于AOP的一种应用。一个拦截器实例在一个controller生命周期内可以多次调用。只能拦截Controller的请求。
2. 过滤器(Filter)：依赖于Servlet容器，基于函数回掉，可以对几乎所有请求过滤，一个过滤器实例只能在容器初使化调用一次。
3. 监听器(Listener)：web监听器是Servlet中的特殊的类，用于监听web的特定事件，随web应用启动而启动，只初始化一次。

**有什么用**

1. 拦截器(interceptor)：在一个请求进行中的时候，你想干预它的进展，甚至控制是否终止。这是拦截器做的事。
2. 过滤器(Filter)：当有一堆东西，只希望选择符合的东西。定义这些要求的工具，就是过滤器。
3. 监听器(Listener)：一个事件发生后，只希望获取这些事个事件发生的细节，而不去干预这个事件的执行过程，这就用到监听器

### 1.11 Spring boot 源码：Bean的Scope

Scope描述的是Spring容器如何创建新Bean的实例。Spring的Scope有以下几种，通过@Scope注解来实现。

1. Singleton：一个Spring容器中只有一个Bean的实例，此为Spring的默认配置，全容器共享一个实例。
2. Prototype：每次调用新建一个Bean实例。
3. Request：Web项目中，给每一个HTTP Request新建一个Bean实例。
4. Session：Web项目中，给每一个HTTP Session新建一个Bean实例。
5. GlobalSession：这个只在portal应用中有用，给每一个global http session新建一个Bean实例。

### 1.12 Spring 框架中都用到了哪些设计模式？

（1）工厂模式：BeanFactory就是简单工厂模式的体现，用来创建对象的实例；

（2）单例模式：Bean默认为单例模式。

（3）代理模式：Spring的AOP功能用到了JDK的动态代理和CGLIB字节码生成技术；

（4）模板方法：用来解决代码重复的问题。比如. RestTemplate, JmsTemplate, JpaTemplate。

（5）观察者模式：定义对象键一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都会得到通知被制动更新，如Spring中listener的实现--ApplicationListener。

### 1.13 Spring事务的实现方式和实现原理

Spring事务的本质其实就是数据库对事务的支持，没有数据库的事务支持，spring是无法提供事务功能的。真正的数据库层的事务提交和回滚是通过binlog或者redo log实现的。

### 1.14 Spring框架中有哪些不同类型的事件？

Spring 提供了以下5种标准的事件：

（1）上下文更新事件（ContextRefreshedEvent）：在调用ConfigurableApplicationContext 接口中的refresh方法时被触发。

（2）上下文开始事件（ContextStartedEvent）：当容器调用ConfigurableApplicationContext的Start方法开始/重新开始容器时触发该事件。

（3）上下文停止事件（ContextStoppedEvent）：当容器调用ConfigurableApplicationContext的Stop方法停止容器时触发该事件。

（4）上下文关闭事件（ContextClosedEvent）：当ApplicationContext被关闭时触发该事件。容器被关闭时，其管理的所有单例Bean都被销毁。

（5）请求处理事件（RequestHandledEvent）：在Web应用中，当一个http请求（request）结束触发该事件。

### 1.15 Spring 事务实现方式

- 编程式事务管理：这意味着你可以通过编程的方式管理事务，这种方式带来了很大的灵活性，但很难维护。
- 声明式事务管理：这种方式意味着你可以将事务管理和业务代码分离。你只需要通过注解或者XML配置管理事务。

### 1.16 BeanFactory和ApplicationContext有什么区别？

ApplicationContext提供了一种解决文档信息的方法，一种加载文件资源的方式(如图片)，他们可以向监听他们的beans发送消息。另外，容器或者容器中beans的操作，这些必须以bean工厂的编程方式处理的操作可以在应用上下文中以声明的方式处理。应用上下文实现了MessageSource，该接口用于获取本地消息，实际的实现是可选的。

相同点：两者都是通过xml配置文件加载bean,ApplicationContext和BeanFacotry相比,提供了更多的扩展功能。

不同点：BeanFactory是延迟加载,如果Bean的某一个属性没有注入，BeanFacotry加载后，直至第一次使用调用getBean方法才会抛出异常；而ApplicationContext则在初始化自身是检验，这样有利于检查所依赖属性是否注入；所以通常情况下我们选择使用ApplicationContext。

### 1.17 什么是Spring Beans？

Spring Beans是构成Spring应用核心的Java对象。这些对象由Spring IOC容器实例化、组装、管理。这些对象通过容器中配置的元数据创建，例如，使用XML文件中定义的创建。

在Spring中创建的beans都是单例的beans。在bean标签中有一个属性为”singleton”,如果设为true，该bean是单例的，如果设为false，该bean是原型bean。Singleton属性默认设置为true。因此，spring框架中所有的bean都默认为单例bean。

### 1.18 解释自动装配的各种模式？

自动装配提供五种不同的模式供Spring容器用来自动装配beans之间的依赖注入:

1. no：默认的方式是不进行自动装配，通过手工设置ref 属性来进行装配bean。
2. byName：通过参数名自动装配，Spring容器查找beans的属性，这些beans在XML配置文件中被设置为byName。之后容器试图匹配、装配和该bean的属性具有相同名字的bean。
3. byType：通过参数的数据类型自动自动装配，Spring容器查找beans的属性，这些beans在XML配置文件中被设置为byType。之后容器试图匹配和装配和该bean的属性类型一样的bean。如果有多个bean符合条件，则抛出错误。
4. constructor：这个同byType类似，不过是应用于构造函数的参数。如果在BeanFactory中不是恰好有一个bean与构造函数参数相同类型，则抛出一个严重的错误。
5. autodetect：自动探测，如果有默认的构造方法，通过 construct的方式自动装配，否则使用 byType的方式自动装配。



### 1.19 有哪些不同类型的IOC(依赖注入)？

构造器依赖注入：构造器依赖注入在容器触发构造器的时候完成，该构造器有一系列的参数，每个参数代表注入的对象。

Setter方法依赖注入：首先容器会触发一个无参构造函数或无参静态工厂方法实例化对象，之后容器调用bean中的setter方法完成Setter方法依赖注入。

你推荐哪种依赖注入？构造器依赖注入还是Setter方法依赖注入？

你可以同时使用两种方式的依赖注入，最好的选择是使用构造器参数实现强制依赖注入，使用setter方法实现可选的依赖关系。

### 1.20 **你可以在Spring中注入一个null 和一个空字符串吗？**

**可以。**

### 1.21 **Aspect 切面**

AOP核心就是切面，它将多个类的通用行为封装成可重用的模块，该模块含有一组API提供横切功能。比如，一个日志模块可以被称作日志的AOP切面。根据需求的不同，一个应用程序可以有若干切面。在Spring AOP中，切面通过带有@Aspect注解的类实现。

### 1.22 Spring AOP 和 AspectJ AOP 有什么区别？

Spring AOP 属于运行时增强，而 AspectJ 是编译时增强。 Spring AOP 基于代理(Proxying)，而 AspectJ 基于字节码操作(Bytecode Manipulation)。

Spring AOP 已经集成了 AspectJ ，AspectJ 应该算的上是 Java 生态系统中最完整的 AOP 框架了。AspectJ 相比于 Spring AOP 功能更加强大，但是 Spring AOP 相对来说更简单，

如果我们的切面比较少，那么两者性能差异不大。但是，当切面太多的话，最好选择 AspectJ ，它比Spring AOP 快很多。

### 1.23 **Spring框架中的单例Beans是线程安全的么？**

 Spring框架并没有对单例bean进行任何多线程的封装处理。关于单例bean的线程安全和并发问题需要开发者自行去搞定。但实际上，大部分的Spring bean并没有可变的状态(比如Serview类和DAO类)，所以在某种程度上说Spring的单例bean是线程安全的。如果你的bean有多种状态的话（比如 View Model 对象），就需要自行保证线程安全。最浅显的解决办法就是将多态bean的作用域由“singleton”变更为“prototype”。


大部分时候我们并没有在系统中使用多线程，所以很少有人会关注这个问题。单例 bean 存在线程问题，主要是因为当多个线程操作同一个对象的时候，对这个对象的非静态成员变量的写操作会存在线程安全问题。

常见的有两种解决办法：

1. 在Bean对象中尽量避免定义可变的成员变量（不太现实）。
2. 在类中定义一个ThreadLocal成员变量，将需要的可变成员变量保存在 ThreadLocal 中（推荐的一种方式）。

### 1.24 **Spring如何处理线程并发问题？**

在一般情况下，只有无状态的Bean才可以在多线程环境下共享，在Spring中，绝大部分Bean都可以声明为singleton作用域，因为Spring对一些Bean中非线程安全状态采用ThreadLocal进行处理，解决线程安全问题。

ThreadLocal和线程同步机制都是为了解决多线程中相同变量的访问冲突问题。同步机制采用了“时间换空间”的方式，仅提供一份变量，不同的线程在访问前需要获取锁，没获得锁的线程则需要排队。而ThreadLocal采用了“空间换时间”的方式。

ThreadLocal会为每一个线程提供一个独立的变量副本，从而隔离了多个线程对数据的访问冲突。因为每一个线程都拥有自己的变量副本，从而也就没有必要对该变量进行同步了。ThreadLocal提供了线程安全的共享对象，在编写多线程代码时，可以把不安全的变量封装进ThreadLocal。



### 1.25 **Spring IoC容器是什么？**

Spring IOC负责创建对象、管理对象(通过依赖注入)、整合对象、配置对象以及管理这些对象的生命周期。



### 1.26 **Spring基于xml注入bean的几种方式**

（1）Set方法注入；

（2）构造器注入：①通过index设置参数的位置；②通过type设置参数类型；

（3）静态工厂注入；

（4）实例工厂；

### 1.27 @Autowired和@Resource之间的区别

1. @Autowired默认是按照类型装配注入的，默认情况下它要求依赖对象必须存在（可以设置它required属性为false）。

2. @Resource默认是按照名称来装配注入的，只有当找不到与名称匹配的bean才会按照类型来装配注入。



### 1.28 **Spring框架中有哪些不同类型的事件？**

Spring 提供了以下5种标准的事件：

1. 上下文更新事件（ContextRefreshedEvent）：在调用ConfigurableApplicationContext 接口中的refresh()方法时被触发
2. 上下文开始事件（ContextStartedEvent）：当容器调用ConfigurableApplicationContext的Start()方法开始/重新开始容器时触发该事件。
3. 上下文停止事件（ContextStoppedEvent）：当容器调用ConfigurableApplicationContext的Stop()方法停止容器时触发该事件。
4. 上下文关闭事件（ContextClosedEvent）：当ApplicationContext被关闭时触发该事件。容器被关闭时，其管理的所有单例Bean都被销毁。
5. 请求处理事件（RequestHandledEvent）：在Web应用中，当一个http请求（request）结束触发该事件。

如果一个bean实现了ApplicationListener接口，当一个ApplicationEvent 被发布以后，bean会自动被通知。

### 1.29 **解释一下Spring AOP里面的几个名词：**

1. 切面（Aspect）：被抽取的公共模块，可能会横切多个对象。 在Spring AOP中，切面可以使用通用类（基于模式的风格） 或者在普通类中以 @AspectJ 注解来实现。
2. 连接点（Join point）：指方法，在Spring AOP中，一个连接点 总是 代表一个方法的执行。 
3. 通知（Advice）：在切面的某个特定的连接点（Join point）上执行的动作。通知有各种类型，其中包括“around”、“before”和“after”等通知。许多AOP框架，包括Spring，都是以拦截器做通知模型， 并维护一个以连接点为中心的拦截器链。
4. 切入点（Pointcut）：切入点是指 我们要对哪些Join point进行拦截的定义。通过切入点表达式，指定拦截的方法，比如指定拦截add*、search*。
5. 引入（Introduction）：（也被称为内部类型声明（inter-type declaration））。声明额外的方法或者某个类型的字段。Spring允许引入新的接口（以及一个对应的实现）到任何被代理的对象。例如，你可以使用一个引入来使bean实现 IsModified 接口，以便简化缓存机制。
6. 目标对象（Target Object）： 被一个或者多个切面（aspect）所通知（advise）的对象。也有人把它叫做 被通知（adviced） 对象。 既然Spring AOP是通过运行时代理实现的，这个对象永远是一个 被代理（proxied） 对象。
7. 织入（Weaving）：指把增强应用到目标对象来创建新的代理对象的过程。Spring是在运行时完成织入。

### 1.30 **Spring通知有哪些类型？**

1. 前置通知（Before advice）：在某连接点（join point）之前执行的通知，但这个通知不能阻止连接点前的执行（除非它抛出一个异常）。
2. 返回后通知（After returning advice）：在某连接点（join point）正常完成后执行的通知：例如，一个方法没有抛出任何异常，正常返回。 
3. 抛出异常后通知（After throwing advice）：在方法抛出异常退出时执行的通知。 
4. 后通知（After (finally) advice）：当某连接点退出的时候执行的通知（不论是正常返回还是异常退出）。 
5. 环绕通知（Around Advice）：包围一个连接点（join point）的通知，如方法调用。这是最强大的一种通知类型。 环绕通知可以在方法调用前后完成自定义的行为。它也会选择是否继续执行连接点或直接返回它们自己的返回值或抛出异常来结束执行。 环绕通知是最常用的一种通知类型。大部分基于拦截的AOP框架，例如Nanning和JBoss4，都只提供环绕通知。 

### 1.31 ApplicationContext通常的实现是什么？

1. FileSystemXmlApplicationContext ：此容器从一个XML文件中加载beans的定义，XML Bean 配置文件的全路径名必须提供给它的构造函数。

2. ClassPathXmlApplicationContext：此容器也从一个XML文件中加载beans的定义，这里，你需要正确设置classpath因为这个容器将在classpath里找bean配置。

3. WebXmlApplicationContext：此容器加载一个XML文件，此文件定义了一个WEB应用的所有bean。

### 1.32 构造方法注入和setter注入之间的区别吗?

有以下几点明显的差异:

1. 在Setter注入,可以将依赖项部分注入,构造方法注入不能部分注入，因为调用构造方法如果传入所有的参数就会报错。
2. 如果我们为同一属性提供Setter和构造方法注入，Setter注入将覆盖构造方法注入。但是构造方法注入不能覆盖setter注入值。显然，构造方法注入被称为创建实例的第一选项。
3. 使用setter注入你不能保证所有的依赖都被注入,这意味着你可以有一个对象依赖没有被注入。在另一方面构造方法注入直到你所有的依赖都注入后才开始创建实例。
4. 在构造函数注入,如果A和B对象相互依赖：A依赖于B,B也依赖于A,此时在创建对象的A或者B时，Spring抛出`ObjectCurrentlyInCreationException`。所以Spring可以通过setter注入,从而解决循环依赖的问题。



