# Spring IOC 和 AOP #
## 参考资料 ##
[https://juejin.im/post/5b040cf66fb9a07ab7748c8b](https://juejin.im/post/5b040cf66fb9a07ab7748c8b)
## IOC ##
控制反转(Inversion of Control)是 OOP中的一种设计原则,也是 Spring框架的核心.大多数应用程序的业务逻辑代码都需要两个或多个类进行合作完成的,通过IoC则可以减少它们之间的耦合度.
### IOC容器原理 ###
IOC容器其实就是一个大工厂,它用来管理我们所有的对象以及依赖关系.  

- 原理就是通过Java的反射技术来实现的.通过反射我们可以获取类的所有信息(成员变量,类名等等).
- 通过配置文件(xml)或者注解来描述类与类之间的关系
- 我们就可以通过这些配置信息和反射技术来构建出对应的对象和依赖关系.

### IOC容器分类 ###
- BeanFactory  
默认lazy-load，所以启动较快常用的一个实现类是DefaultListableBeanFactory，该类还实现了BeanDefinitionRegistry接口，该接口担当Bean注册管理的角色registerBeanDefinition#BeanDefinitionRegistry
bean会以beanDefinition的形式注册在BeanDefinitionRegistry中BeanFactory接口中只定义了一些关于bean的查询方法，而真正的工作需要其子类实现的其他接口定义

- ApplicationContext  
构建于BeanFactory基础之上，非lazy，启动时间较长，除了BeanFactory的基础功能还提供了一些额外的功能（事件发布、国际化信息支持等）

几乎所有的应用场合都是使用ApplicationContext！

- 两者不同之处
	1. ApplicationContext会利用Java反射机制自动识别出配置文件中定义的BeanPostProcessor、 InstantiationAwareBeanPostProcesso 和BeanFactoryPostProcessor后置器，并自动将它们注册到应用上下文中。而BeanFactory需要在代码中通过手工调用addBeanPostProcessor()方法进行注册
	2. ApplicationContext在初始化应用上下文的时候就实例化所有单实例的Bean。而BeanFactory在初始化容器的时候并未实例化Bean，直到第一次访问某个Bean时才实例化目标Bean.

### IOC容器装配Bean ###
1. 装配Bean方式
- xml配置
- 注解
- JavaConfig
- 基于Groovy DSL配置(这种很少见)

2. 依赖注入(DI) 
- 基于构造函数 : 实现特定参数的构造函数,在创建对象时来让 IoC容器注入所依赖类型的对象.
- 基于set方法 : 实现特定属性的public set()方法,来让IoC容器调用注入所依赖类型的对象.
- 基于接口 : 实现特定接口以供 IoC容器注入所依赖类型的对象.
- 基于注解 : 通过 Java的注解机制来让 IoC容器注入所依赖类型的对象,例如 Spring框架中的@Autowired.

### spring bean容器的生命周期 ###
1. Spring 容器根据配置中的 bean 定义中实例化 bean。
2. Spring 使用依赖注入填充所有属性，如 bean 中所定义的配置。
3. 如果 bean 实现 BeanNameAware 接口，则工厂通过传递 bean 的 ID 来调用 setBeanName()。
4. 如果 bean 实现 BeanFactoryAware 接口，工厂通过传递自身的实例来调用 setBeanFactory()。
5. 如果存在与 bean 关联的任何 BeanPostProcessors，则调用 preProcessBeforeInitialization() 方法。
6. 如果为 bean 指定了 init 方法（<bean> 的 init-method 属性），那么将调用它。
7. 最后，如果存在与 bean 关联的任何 BeanPostProcessors，则将调用 postProcessAfterInitialization() 方法。
8. 如果 bean 实现 DisposableBean 接口，当 spring 容器关闭时，会调用 destory()。
9. 如果为 bean 指定了 destroy 方法（<bean> 的 destroy-method 属性），那么将调用它。
 
## AOP ##
AOP(Aspect-OrientedProgramming)即面向方面编程.它是一种在运行时,动态地将代码切入到类的指定方法、指定位置上的编程思想.用于切入到指定类指定方法的代码片段叫做切面,而切入到哪些类中的哪些方法叫做切入点.    
AOP是OOP的有益补充,OOP从横向上区分出了一个个类, AOP则从纵向上向指定类的指定方法中动态地切入代码.它使 OOP变得更加立体. 
### 参考资料 ###
[https://www.jianshu.com/p/fe2a1eebfd17](https://www.jianshu.com/p/fe2a1eebfd17)
### 相关概念 ###   
- Joinpoint(连接点)   
	- 所谓连接点是指那些被拦截到的点。在spring中,这些点指的是方法,因为spring只支持方法类型的连接点
- Pointcut(切入点)  
	- 所谓切入点是指我们要对哪些Joinpoint进行拦截的定义
- Advice(通知/增强)
	- 所谓通知是指拦截到Joinpoint之后所要做的事情就是通知.通知分为前置通知,后置通知,异常通知,最终通知,环绕通知(切面要完成的功能)
- Introduction(引介)  
	- 引介是一种特殊的通知在不修改类代码的前提下, Introduction可以在运行期为类动态地添加一些方法或Field
- Target(目标对象)
	- 代理的目标对象
- Weaving(织入)
	- 是指把增强应用到目标对象来创建新的代理对象的过程
- Proxy（代理）
	- 一个类被AOP织入增强后，就产生一个结果代理类
- Aspect(切面)
	- 是切入点和通知的结合，以后咱们自己来编写和配置的  
		

### AOP的通知类型 ###
- 前置通知:在目标类的方法执行之前执行。
	- 配置文件信息：<aop:after method="before" pointcut-ref="myPointcut3"/>
	- 应用：可以对方法的参数来做校验
- 最终通知:在目标类的方法执行之后执行，如果程序出现了异常，最终通知也会执行。
	- 在配置文件中编写具体的配置：<aop:after method="after" pointcut-ref="myPointcut3"/>
	- 应用：例如像释放资源
- 后置通知:方法正常执行后的通知.
	- 在配置文件中编写具体的配置：<aop:after-returning method="afterReturning" pointcut-ref="myPointcut2"/>
	- 应用：可以修改方法的返回值
- 异常抛出通知:在抛出异常后通知
	- 在配置文件中编写具体的配置：<aop:after-throwing method="afterThorwing" pointcut-ref="myPointcut3"/>
	- 应用：包装异常的信息
- 环绕通知:方法的执行前后执行。
	- 在配置文件中编写具体的配置：<aop:around method="around" pointcut-ref="myPointcut2"/>
	- 要注意：目标的方法默认不执行，需要使用ProceedingJoinPoint对来让目标对象的方法执行。
