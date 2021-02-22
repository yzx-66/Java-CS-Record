AOP 是OOP 的延续，是AspectOrientedProgramming的缩写，意思是面向切面编程。可以通过预编译方式和运行期动态代理实现在不修改源代码的情况下给程序动态统一添加功能的一种技术。AOP设计模式孜孜不倦追求的是调用者和被调用者之间的解耦，AOP可以说也是这种目标的一种实现。

我们现在做的一些非业务，如：日志、事务、安全等都会写在业务代码中(也即是说，这些非业务类横切于业务类)，但这些代码往往是重复，复制——粘贴式的代码会给程序的维护带来不便，AOP 就实现了把这些业务需求与系统需求分开来做。这种解决的方式也称代理机制。 

## 1.AOP简介

定义：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。


优点：利用AOP可以对业务逻辑的各个部分进行隔离，从而使得业务逻辑各部分之间的耦合度降低，提高程序的可重用性，同时提高了开发的效率。

原理：aop底层将采用代理机制进行实现

- 接口 + 实现类：spring 默认采用 jdk 的动态代理 Proxy。
- 实现类：spring 默认采用 cglib 字节码增强。

**几个基本概念**：

1. target：目标类，需要被代理的类。例如：UserService

2. Joinpoint：连接点，是指那些可能被拦截到的方法们。例如：UserService中 m1(), m2(), m3()....等待选方法们

3. Aspect: 切面，切入点（pointcut)  + 通知（advice) = **指定切点的非业务逻辑类**。例如：LogAspect

   >这正是**面向切面**的意义（解耦非业务逻辑，如 LogAspect ）

4. advice： 通知/增强，增强代码。例如：after、before
   * 前置通知（before）
     * 在方法执行前执行，如果通知抛出异常，阻止方法运行
     * 应用：各种校验
   * 后置通知 （after）
     * 方法执行完毕后执行，无论方法中是否出现异常
     * 应用：清理现场
   * 后置通知 （afterReturning）
     * 方法正常返回后执行，如果方法中抛出异常，通知无法执行必须在方法执行后才执行，所以可以获得方法的返回值。
     * 应用：常规数据处理
   * 环绕通知 （around）
     * 方法执行前后分别执行，可以阻止方法的执行必须手动执行目标方法
     * 应用：十分强大，可以做任何事情
   * 异常抛出通知 （afterThrowing）
     *  方法抛出异常后执行，如果方法没有抛出异常，无法执行
     * 应用：包装异常信息
   * 引介通知 `org.springframework.aop.IntroductionInterceptor`：在目标类中添加一些新的方法和属性
5. PointCut：切入点，已经被增强的连接点，PointCut属于JoinCut。例如：m1()，已经选定的方法；
6. Weaving：**织入**，是指把 advice 应用到 target 来创建代理对象的过程。
7. Proxy：代理类。在 Spring AOP 中有两种代理方式，JDK动态代理和 CGLib代理。默认情况下，TargetObject 实现了接口时，则采用 JDK动态代理，例如，AServiceImpl；反之，采用CGLib代理，例如，BServiceImpl。强制使用CGLib代理需要将 `<aop:config>`的 proxy-target-class属性设为true。

## 2.Spring AOP 编程

使用SpringAOP 可以基于两种方式，一种是比较方便和强大的注解方式，另一种则是中规中矩的xml配置方式。先说注解，使用注解配置 SpringAOP 总体分为两步：

### 2.1 基于注解

>AspectJ 是一个基于 Java 语言的 AOP 框架，Spring2.0 以后新增了对 AspectJ 切点表达式支持。 `@AspectJ` 是 AspectJ1.5 新增功能，通过 JDK5 注解技术，允许直接在 Bean 类中定义切面。新版本 Spring 框架，建议使用 AspectJ 方式来开发 AOP，主要用途是自定义开发。

第一步：在 xml文件中声明激活自动扫描组件功能，同时激活自动代理功能

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop" xsi:schemaLocation="
        http://www.springframework.org/schema/beans 
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/aop 
        	http://www.springframework.org/schema/aop/spring-aop.xsd">
	
    <!-- 扫描com.yzh包下的所有bean，交给IOC容器管理 -->
    <context:component-scan base-package="com.yzh"/>
    <!-- 开启主机 -->
    <context:annotation-config />
    <!-- 添加aop注解驱动（AspectJ） -->
    <aop:aspectj-autoproxy proxy-target-class="true"/>
</beans>
```

第二步：为Aspect切面类添加注解

```java
// 声明这是一个组件
@Component
// 声明这是一个切面Bean，AnnotaionAspect是一个面，由框架实现的
@Aspect
public class AnnotaionAspect {

   private final static Logger log = Logger.getLogger(AnnotaionAspect.class);
   
   // 配置切入点,该方法无方法体,主要为方便同类中其他方法使用此处配置的切入点切点的集合，
   // 这个表达式所描述的是一个虚拟面（规则）,就是为了Annotation扫描时能够拿到注解中的内容
   @Pointcut("execution(* com.yzh.aop.service..*(..))")
   public void aspect(){}
   
   /*
    * 配置前置通知,使用在方法aspect()上注册的切入点
    * 同时接受JoinPoint切入点对象,可以没有该参数
    */
   @Before("aspect()")
   public void before(JoinPoint joinPoint){
      log.info("before " + joinPoint);
   }
   
   // 配置后置通知,使用在方法aspect()上注册的切入点
   @After("aspect()")
   public void after(JoinPoint joinPoint){
      log.info("after " + joinPoint);
   }
   
   // 配置环绕通知,使用在方法aspect()上注册的切入点
   @Around("aspect()")
   public void around(JoinPoint joinPoint){
      long start = System.currentTimeMillis();
      try {
         ((ProceedingJoinPoint) joinPoint).proceed();
         long end = System.currentTimeMillis();
         log.info("around " + joinPoint + "\tUse time : " + (end - start) + " ms!");
      } catch (Throwable e) {
         long end = System.currentTimeMillis();
         log.info("around " + joinPoint + "\tUse time : " + (end - start) + " ms with exception : " + e.getMessage());
      }
   }
   
   // 配置后置返回通知,使用在方法aspect()上注册的切入点
   @AfterReturning("aspect()")
   public void afterReturn(JoinPoint joinPoint){
      log.info("afterReturn " + joinPoint);
   }
   
   // 配置抛出异常后通知,使用在方法aspect()上注册的切入点
   @AfterThrowing(pointcut="aspect()", throwing="ex")
   public void afterThrow(JoinPoint joinPoint, Exception ex){
      log.info("afterThrow " + joinPoint + "\t" + ex.getMessage());
   }
   
}
```
测试代码：正常调用相应代码
```java
@ContextConfiguration(locations = {"classpath*:application-context.xml"}) // 把配置文件加载进来
@RunWith(SpringJUnit4ClassRunner.class)
public class AnnotationTest {
	@Autowired
	MemberService memberService;
	
	@Test
	public void test(){
		System.out.println("=====这是一条华丽的分割线======");
		
		memberService.save(new Member());

		System.out.println("=====这是一条华丽的分割线======");
		try {
			memberService.delete(1L);
		} catch (Exception e) {
			// e.printStackTrace();
		}
	}
}
```
MemberService 代码如下：
```java
@Service
public class MemberService {

   private final static Logger log = Logger.getLogger(AnnotaionAspect.class);
   
   public Member get(long id){
      log.info("getMemberById method . . .");
      return new Member();
   }
   
   
   public Member get(){
      log.info("getMember method . . .");
      return new Member();
   }
   
   public void save(Member member){
      log.info("save member method . . .");
   }
   
   public boolean delete(long id) throws Exception{
      log.info("delete method . . .");
      throw new Exception("spring aop ThrowAdvice演示");
   }
   
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20201203215831553.png?)
可以看到，正如我们预期的那样，虽然我们并没有对MemberService 类包括其调用方式做任何改变，但是Spring仍然拦截到了其中方法的调用，或许这正是AOP 的魔力所在。

### 2.2 xml配置形式
再简单说一下xml配置方式，其实也一样简单，也是分为两步：

```xml
<!-- 将切面Bean交给IOC容器 -->
<bean id="xmlAspect" class="com.yzh.aop.aspect.XmlAspect"></bean>
<!-- AOP配置-->
<aop:config>
   <!--声明一个切面,并注入切面Bean,相当于@Aspect -->
   <aop:aspect ref="xmlAspect" >
      <!--配置一个切入点,相当于@Pointcut -->
      <aop:pointcut expression="execution(* com.yzh.aop.service..*(..))" id="simplePointcut"/>
      <!--配置通知,相当于@Before、@After、@AfterReturn、@Around、@AfterThrowing -->
      <aop:before pointcut-ref="simplePointcut" method="before"/>
      <aop:after pointcut-ref="simplePointcut" method="after"/>
      <aop:after-returning pointcut-ref="simplePointcut" method="afterReturn"/>
      <aop:after-throwing pointcut-ref="simplePointcut" method="afterThrow" throwing="ex"/>
      <aop:around pointcut-ref="simplePointcut"  method="around"/>
   </aop:aspect>
</aop:config>
```
```java
/**
 * XML版Aspect切面Bean(理解为TrsactionManager)
 */
public class XmlAspect {

   private final static Logger log = Logger.getLogger(XmlAspect.class);
   
   /*
    * 配置前置通知,使用在方法aspect()上注册的切入点
    * 同时接受JoinPoint切入点对象,可以没有该参数
    */
   public void before(JoinPoint joinPoint){
//    System.out.println(joinPoint.getArgs()); //获取实参列表
//    System.out.println(joinPoint.getKind());   //连接点类型，如method-execution
//    System.out.println(joinPoint.getSignature()); //获取被调用的切点
//    System.out.println(joinPoint.getTarget()); //获取目标对象
//    System.out.println(joinPoint.getThis());   //获取this的值
      
      log.info("before " + joinPoint);
   }
   
   // 配置后置通知,使用在方法aspect()上注册的切入点
   public void after(JoinPoint joinPoint){
      log.info("after " + joinPoint);
   }
   
   // 配置环绕通知,使用在方法aspect()上注册的切入点
   public void around(JoinPoint joinPoint){
      long start = System.currentTimeMillis();
      try {
         ((ProceedingJoinPoint) joinPoint).proceed();
         long end = System.currentTimeMillis();
         log.info("around " + joinPoint + "\tUse time : " + (end - start) + " ms!");
      } catch (Throwable e) {
         long end = System.currentTimeMillis();
         log.info("around " + joinPoint + "\tUse time : " + (end - start) + " ms with exception : " + e.getMessage());
      }
   }
   
   // 配置后置返回通知,使用在方法aspect()上注册的切入点
   public void afterReturn(JoinPoint joinPoint){
      log.info("afterReturn " + joinPoint);
   }
   
   // 配置抛出异常后通知,使用在方法aspect()上注册的切入点
   public void afterThrow(JoinPoint joinPoint, Exception ex){
      log.info("afterThrow " + joinPoint + "\t" + ex.getMessage());
   }
   
}
```
### 2.3 注解和xml配置对比
个人觉得不如注解灵活和强大，你可以不同意这个观点，但是不知道如下的代码会不会让你的想法有所改善：

```java
// 声明这是一个组件
@Component
// 声明这是一个切面Bean
@Aspect
public class ArgsAspect {

   private final static Logger log = Logger.getLogger(ArgsAspect.class);
   
   // 配置切入点,该方法无方法体,主要为方便同类中其他方法使用此处配置的切入点
   @Pointcut("execution(* com.yzh..aop.service..*(..))")
   public void aspect(){  }
   
   // 配置前置通知,拦截返回值为com.yzh..model.Member的方法
   @Before("execution(com.yzh.model.Member com.yzh..aop.service..*(..))")
   public void beforeReturnUser(JoinPoint joinPoint){
      log.info("beforeReturnUser " + joinPoint);
   }

   // 配置前置通知,拦截参数为com.yzh..model.Member的方法
   @Before("execution(* com.yzh..aop.service..*(com.yzh.model.Member))")
   public void beforeArgUser(JoinPoint joinPoint){
      log.info("beforeArgUser " + joinPoint);
   }

   // 配置前置通知,拦截含有long类型参数的方法,并将参数值注入到当前方法的形参id中
   @Before("aspect()&&args(id)")
   public void beforeArgId(JoinPoint joinPoint, long id){
      log.info("beforeArgId " + joinPoint + "\tID:" + id);
   }
   
}
```

---
应该说学习 Spring AOP 有两个难点，第一点在于理解 AOP 的理念和相关概念，第二点在于灵活掌握和使用切入点表达式。概念的理解通常不在一朝一夕，慢慢浸泡的时间长了，自然就明白了，下面我们简单地介绍一下切入点表达式的配置规则。通常情况下，表达式中使用”execution“就可以满足大部分的要求。表达式格式如下：
```
execution(modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern) throws-pattern?
```
* modifiers-pattern：方法的操作权限
* ret-type-pattern：返回值
* declaring-type-pattern：方法所在的包
* name-pattern：方法名
* parm-pattern：参数名
* throws-pattern：异常

其中，除 ret-type-pattern 和 name-pattern 之外，其他都是可选的。上例中，`execution(* com.yzh.aop.service..*(..))`表示com.yzh.aop.service 包下，返回值为任意类型；方法名任意；参数不作限制的所有方法。

最后说一下通知参数，可以通过args来绑定参数，这样就可以在通知（Advice）中访问具体参数了。例如，<aop:aspect>配置如下：
```xml
<aop:config> 
	<aop:aspect ref="xmlAspect">
	 <aop:pointcut id="simplePointcut" expression="execution(* com.my.aop.service..*(..)) and args(msg,..)" /> 
	 <aop:after pointcut-ref="simplePointcut" Method="after"/>
	</aop:aspect>
</aop:config> 
```
上面的代码 args(msg,..)是指将切入点方法上的第一个 String 类型参数添加到参数名为 msg 的通知的入参上，这样就可以直接使用该参数啦。

在上面的Aspect切面Bean中已经看到了，每个通知方法第一个参数都是 JoinPoint。其实，在Spring中，任何通知（Advice）方法都可以将第一个参数定义为 org.aspectj.lang.JoinPoint 类型用以接受当前连接点对象。JoinPoint 接口提供了一系列有用的方法， 比如 getArgs() （返回方法参数）、getThis() （返回代理对象）、getTarget() （返回目标）、getSignature() （返回正在被通知的方法相关信息）和 toString() （打印出正在被通知的方法的有用信息）。 


