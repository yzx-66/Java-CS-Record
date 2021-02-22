在JAVA的世界中，一个对象A怎么才能调用对象B？通常有以下几种方法：

|   类别   | 描述               | 时间点         |
| :------: | :----------------- | :-------------- |
| 外部传入 | 构造方法传入       |                |
|          | 属性设置传入       | 设置对象状态时 |
|          | 运行时做为参数传入 | 调用时         |
| 内部创建 | 属性中直接创建     | 创建引用对象时 |
|          | 初始化方法创建     | 创建引用对象时 |
|          | 运行时动态创建     | 调用时         |

上表可以看到， 引用一个对象可以在不同地点（其它引用者）、不同时间由不同的方法完成。如果B只是一个非常简单的对象  如直接new B()，怎样都不会觉得复杂，比如你从来不会觉得创建一个String 是一个件复杂的事情。但如果B 是一个有着复杂依赖的Service对象，这时在不同时机引用B将会变得很复杂。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201130212422187.png?)
无时无刻都要维护B的复杂依赖关系，试想B对象如果项目中有上百过，系统复杂度将会成陪数增加。IOC容器的出现正是为解决这一问题，其可以将对象的构建方式统一，并且自动维护对象的依赖关系，从而降低系统的实现成本。

>IOC(Inversion of Control)控制反转：控制反转。就是把原先我们代码里面需要实现的对象创建、依赖的代码，反转给容器来帮忙实现。那么必然的我们需要创建一个容器，同时需要一种描述来让容器知道需要创建的对象与对象的关系。这个描述最具体表现就是我们所看到的配置文件。

## 1.实体Bean的构建

### 1.1 xml 配置形式
xml 配置形式，顾名思义就是在单独的 xml 文件中对 bean 进行配置，常见的有以下几种形式：

**1.ClassName反射构建**

```xml
<!-- 默认构造函数构建 -->
<bean name="hello",class="com.my.spring.HelloSpring"></bean>
```

这是最常规的方法，其原理是在spring底层会基于 Class 属通过反射进行构建。
```java
Object obj = HelloSpring.class.newInstance();
```

>那我们该如何获取IOC容器中的Bean呢？
```java
public static void main(String[] args)    
{    
     // 传入配置文件
     ClassPathXmlApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml"); 
     // getBean 可以传入id、name、Class 来获取唯一个 bean
     // 注：id与name的区别在于，id必须唯一，而name可以不唯一（冲突时后面的对象会覆盖前面的）
  	 System.out.println(ctx.getBean("hello"));  
}
```
**2.指定构造方法构建**

如果需要基于参数进行构建，就采用构造方法构建，其对应属性如下：

* name:构造方法参数变量名称
* type:参数类型（非必须）
* index:参数索引，从0开始
* value:参数值，spring 会自动转换成参数实际类型值
* ref:引用容串的其它对象

```java
public class HelloSpring {
	private String name;
	private int sex;
	
	public HelloSpring(){}
	
	public HelloSpring(String name) {
		this.name = name;
		this.sex = sex;
	}
}
```

```xml
<!-- 相当于是指定构造函数构建，而上面是用默认构造函数构建 -->
<bean class="com.my.spring.HelloSpring">
    <constructor-arg name="name" type="java.lang.String" value="zhangsan"/>
    <constructor-arg index="1" value="1" />
</bean>
```

**3.静态工厂方法创建**

该模式下必须创建一个静态工厂方法，并且方法返回该实例，spring 会调用该静态方法创建对象。
```xml
<!-- 指定Bean的类型，静态工厂方法（未特别指定则在当前类中） -->
<bean class="com.my.spring.HelloSpring" factory-method="build">
	<!-- 给工厂方法传入的参数名称，值。这里是要构造一个B类型的对象 -->
    <constructor-arg name="type" type="java.lang.String" value="B"/>
</bean>
```
```java
// bulid 方法相当于一个静态工厂，返回已经创建好的对象
public static HelloSpring build(String type) {
	// 如果需要的是A类型的Bean
    if (type.equals("A")) {
        return new HelloSpring("luban",1);、
    // 如果需要的是B类型的Bean
    } else if (type.equals("B")) {
        return new HelloSpring("diaocan", 0);
    // 如果既不是A类型也不是B类型
    } else {
        throw new IllegalArgumentException("type must A or B");
    }
}
```
使用场景：如果你正在对一个对象进行A/B测试 ，就可以采用静态工厂方法的方式创建，其于策略创建不同的对像或填充不同的属性。

**4.FactoryBean创建**

指定一个Bean工厂来创建对象，对象构建初始化完全交给该工厂来实现。配置Bean时指定该工厂类的类名。
```xml
<!-- 构建一个工厂Bean  -->
<bean class="com.my.spring.MYFactoryBean"></bean>
```
```java
// 实现FactoryBean接口
public class MYFactoryBean implements FactoryBean {
    @Override
    public Object getObject() throws Exception {
        return new HelloSpring();
    }
    @Override
    public Class<?> getObjectType() {
        return HelloSpring.class;
    }
    @Override
    public boolean isSingleton() {
        return false;
    }
}
```
>FactoryBean：具有工厂生产对象能力的 bean，只能生成特定的对象。bean 必须实现 FactoryBean接口，此接口提供方法 `getObject()` 用于获得特定 bean。所以，使用时先创建FB实例，然后调用 getObject() 方法。
### 1.2 JavaConfig配置形式
JavaConfig 不再是单独的配置文件去配置，而是采用标了 @Configuration 注解的配置类去配置bean：

**1.@Bean（适用于第三方组件）**

* 通过@Bean的形式是使用的话，bean的默认名称是方法名
* 若@Bean(value="bean的名称") 那么bean的名称是指定的 

```java
@Configuration 
public class MainConfig { 
    @Bean     
    public Person person(){  
        return new Person(); 
    } 
}
```
> 如果是注解形式的，那我该如何获取这个bean呢？
```java
public static void main(String[] args) { 
 // 传入配置类                                          
 AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MainConfig.class);
 System.out.println(ctx.getBean("person"));  
}
```

**2.@CompentScan（适用于自定义类）**

在配置类上写@CompentScan注解来进行包扫描。除此之外还有 @Compent,@Controller,@Service,@Repository

@ComponetScan常用的属性字段有 basepackage（包）、excludeFilters（排除）、includeFilters（包含）。
>注意，@CompentScan一般直接写在配置类上。
```java
@Configuration 
// basePackages指定啊哟扫描的包
// 注意：如果要扫描的的包有多个，需要用{}包裹
@ComponentScan(basePackages = {"com.my.testcompentscan"}) 
public class MainConfig { 
}
```
```java
@Configuration 
// excludeFilters 指定排除扫描的类
// 排除@Controller注解的,和TestService的)
@ComponentScan(basePackages = {"com.my.testcompentscan"}, excludeFilters = { 
    @ComponentScan.Filter(type = FilterType.ANNOTATION,value = {Controller.class}), 
    @ComponentScan.Filter(type = FilterType.ASSIGNABLE_TYPE,value = {TestgService.class})
}) 
public class MainConfig { 
}
```
```java
@Configuration 
// 只注入@Controller 和 @Service
// 注意：若使用包含的用法，需要把useDefaultFilters属性设置为false（true表示扫描全部的）
@ComponentScan(basePackages = {"com.my.testcompentscan"},includeFilters = { 
    @ComponentScan.Filter(type = FilterType.ANNOTATION,value = {Controller.class, Service.class}) 
},useDefaultFilters = false) 
public class MainConfig { 
}
```

**3.@Import（适用于特定场景）**

定义在配置类上，value指定导入的类，可多个。也可以实现ImportSelector重写selectImports定义导入类的注解信息

```java
@Configuration
@Import(value = {Person.class, Car.class})
public class MainConfig { 
}
```

```java
public class MYImportSelector implements ImportSelector { 
    // 可以获取导入类的注解信息     
    @Override   
    public String[] selectImports(AnnotationMetadata importingClassMetadata) { 
    	return new String[]{"com.my.testimport.compent.Dog"};   
    } 
} 

@Configuration 
@Import(value = {Person.class, Car.class, MYImportSelector.class})
public class MainConfig { 
}
```

## 2.Bean相关配置

上面我们只是介绍了如何通过配置的形式声明一个bean，但是具体bean中可以配置哪些信息还不知道，下面我们就一起来看看。

### 2.1 作用域

|     类别      |                             说明                             |
|  :-----------: | :----------------------------------------------------------: |
|   singleton   |            Spring IOC 容器中仅存在一份 bean 实例             |
|   prototype   |                      每次都创建新的实例                      |
|    request    | 一次 http 请求创建一个新的 bean，仅适用于 WebApplicationContext |
|    session    | 一个 session 对应一个 bean 实例，仅适用于 WebApplicationContext |
| globalSession | 一般用于 Portlet 应用环境，作用域仅适用于 WebApplicationContext 环境 |

>三点注意：
>* 如果不特别指定作用域，那么就默认是singleton
>* singleton默认采用的是饿汉模式加载(容器启动实例就创建好了)
>* prototype默认采用的是懒汉模式加载（IOC容器启动的时候，并不会创建对象，而是 在第一次使用的时候才会创建）

**xml：scope**

```xml
<bean class="com.my.spring.HelloSpring" scope="prototype"></bean>
```

**JavaConfig：@Scope**

```java
@Bean    
@Scope(value = "prototype")   
public Person person() {     
    return new Person();  
} 
```

### 2.2 生命周期（单例可控）

创建 -> 初始化 -> 销毁。由容器管理Bean的生命周期，我们可以自己指定bean的初始化方法和bean的销毁方法。

* 针对**单实例bean**的话，容器启动的时候，bean的对象就创建了，而且容器销毁的时候，也会调用Bean的销毁方法。
* 针对多实例bean的话，容器启动的时候，bean是不会被创建的而是在获取bean的时候被创建，而且bean的销毁不受 IOC容器的管理。

**xml：init-method  destroy-method**

将这两个参数加载要注入的`<bean>`

```xml
<!-- 指定创建和销毁时执行的生命周期方法 -->
<bean class="com.my.spring.HelloSpring" init-method="onInit" destroy-method="onDestroy"></bean>
```
```java
public class HelloSpring {
    public void onInit() {
        System.out.println("onInit");
    }
    public void onDestroy() {
        System.out.println("onDestroy");
    }
}
```

**JavaConfig：@PostConstruct @PreDestory**

将前置与后置方法定义在要加入IOC的Bean内

```java
@Component 
public class Book { 
    
    public Book() {  
        System.out.println("book 的构造方法");   
    } 
    @PostConstruct   
    public void init() {   
        System.out.println("book 的PostConstruct标志的方法");  
    } 
    @PreDestroy   
    public void destory() {     
        System.out.println("book 的PreDestory标注的方法");   
    }
}
```

### 2.3 懒加载（单例）

主要针对单实例的bean 容器启动的时候，不创建对象，在第一次使用的时候才会创建该对象

* 懒加载：容器启动的更快，

* 非懒加载：可以容器启动时更快的发现程序当中的错误 

>选择哪一个就看追求的是启动速度，还是希望更早的发现错误，一般我们会选择后者。

**xml：default-lazy-init**

xml 要配置在最开始的<beans>标签内，里面所有的bean都要懒加载

* true: 懒加载，即延迟加载
* false（默认）:非懒加载，容器启动时即创建对象

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
                           http://www.springframework.org/schema/beans/spring-beans.xsd"
default-lazy-init="true">
```

**JavaConfig：@Lazy**

定义在JavaConfig类中，只指定当前bean

```java
@Bean    
@Lazy     
public Person person() {  
    return new Person();  
} 
```





