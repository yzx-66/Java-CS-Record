DI(DependencyInjection)依赖注入：就是指对象是被动接受依赖类而不是自己主动去找，换句话说就。是指对象不是从容器中查找它依赖的类，而是在容器实例化对象的时候主动将它依赖的类注入给它。

### 1.set方法注入

```java
public class Student {
    private String name;
    private Teacher teacher; 
    
    public String getName() {return name;} 
    public void setName(String name) { this.name = name;}
	public Teacher getTeacher() { return teacher; }
    public void setTeacher(Teacher teacher) { this.teacher = teacher; }
}

public class Teacher {
    private String name;
    
    public String getName() { return name; } 
    public void setName(String name) { this.name = name; }
}
```

**1.xml形式**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
                           http://www.springframework.org/schema/beans/spring-beans.xsd">
 	
    <!-- 创建一个id=teacher的Teacher -->
    <bean id="teacher" class="test.Teacher">
        <property name="name" value="李四"/>
    </bean>
    
    <!-- 创建一个id=student的Student -->
    <bean id="student" class="test.Student">
        <property name="name" value="张三"/>
        <!-- 将teacher注给student -->
        <property name="teacher" ref="teacher"/>
    </bean>
    
 </beans>
```

**2.注解**

```java
@Configuration
public class BeansConfiguration {
 
    @Bean
    // 这个bean的name=student
    public Student student(){
        Student student=new Student();
        student.setName("张三");
        student.setTeacher(teacher());
        return student;
    }
 
    @Bean
    // name=student
    public Teacher teacher(){
        Teacher teacher=new Teacher();
        teacher.setName("李四");
        return teacher;
    }
}
```

```java
public class Main {
 
    public static void main(String args[]){
        FileSystemXmlApplicationContext context=new 
            FileSystemXmlApplicationContext("applicationContext.xml的绝对路径");
        // 容器中拿出来的student张三就是被注入过student的
        Student student= (Student) context.getBean("student");
        Teacher teacher= (Teacher) context.getBean("teacher");
        
        System.out.println("学生的姓名："+student.getName()+"。老师
                           是"+student.getTeacher().getName());
        System.out.println("老师的姓名："+teacher.getName());
    }
}
```



### 2.构造方法注入

```java
public class Student {
    private String name;
    private Teacher teacher; 
    
    public Student(String name,Teacher teacher) {
        this.name = name;
        this.teacher = teacher;
    }
}

public class Teacher {
    private String name;
    
    public String getName() { return name; } 
    public void setName(String name) { this.name = name; }
}
```

**1.xml形式**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">

<!-- 创建Teacher实例teacher -->
<bean name="teacher" class="test.Teacher">
	<property name="name" value="李四"></property>
</bean>

<bean name="student" class="test.Student">
  <!-- 注入 teacher 这个 bean -->
  <constructor-arg ref="teacher" />
</bean>
    
</beans>
```

**2.注解**

```java
@Configuration
public class MainConfig {
    
    // 这个bean没必要注入IOC容器
    public Teacher teacher() {
        return new Teacher();
        teacher.setName = "李四";
    }
    
    // 创建一个bean student
    @Bean
    public Student student() {
        return new Student(teacher());
    }
}
```

### 3.自动注入

自动注入就是根据当前对象中定义的实例变量名进行注入

**1.xml形式：byName + byType**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
 	
    <!-- 创建Teacher实例teacher -->
    <bean id="teacher" class="test.Teacher">
        <property name="name" value="李四"/>
    </bean>
    
    <!-- 创建Student实例student -->
    <!-- byName 会自动给未初始化实例变量找能匹配的bean注入进来 -->
    <bean id="student" class="test.Student" autowire="byName">
        <property name="name" value="张三"/>
    </bean>  
 
</beans>
```

**2.注解形式：@Autowired + @Value**

@Autowired 
  * 自动装配首先时按照类型进行装配，若在IOC容器中发现多个相同类型的组件，那么就按照属性名称来进行装配
  * @Qualifier("name")：可以在容器中有多个同一类的bean时指定name。比如 Teacher有teacher1，teacher2，那么我们就可以 @Autowired @Qualifier("teacher1") 指定teacher1注入。

> @Autowired 和 @Resource：
> * 共同点：@Resource和@Autowired都可以作为注入属性的修饰，在接口仅有单一实现类时，两个注解的修饰效果相同，可以互相替换，不影响使用。
> * 不同点：
> 	* @Autowired是spring的注解，是spring2.5版本引入的，Autowired只根据type进行注入，不会去匹配name。如果涉及到type无法辨别注入对象时，那需要依赖@Qualifier或@Primary注解一起来修饰。
> 	* @Resource 是JDK1.6支持的注解， 默认按照名称进行装配，名称可以通过name属性进行指定，如果没有指定name属性，当注解写在字段上时，默认取字段名，按照名称查找。如果注解写在setter方法上默认取属性名进行装配。
>**当找不到与名称匹配的bean时才按照类型进行装配**。但是需要注意的是，如果name属性一旦指定，就只会按照名称进行装配。
>
>一般推荐使用 @Autowired，当需要指定 beanName 时再用 @Resource。

@Value

  * @Value(value)：给当前变量直接注入指定值value，类型要对应
  * @Value("#{configProperties['key']}")：表示SpEl表达式通常用来获取bean的属性，或者调用bean的某个方法。当然还有可以表示常量
  * @Value("${key}")：可以获取对应属性文件中定义的属性值。

```java
// 将Teacher的实例teacher注入IOC容器
@Component("teacher") 
public class Teacher {
 
    @Value("李四")  // 为name注入String值，李四
    private String name;
 
    public String getName() {
        return name;
    }
 
    public void setName(String name) {
        this.name = name;
    }
}

// 将Student实例student注入IOC容器
@Component("student") 
public class Student {
 	
    @Value("张三") 
    private String name;  // name = 张三
 
    @Resource  
    private Teacher teacher; // 通过名字 teacher 去寻找bean，找到了注入进去
 
    public String getName() {
        return name;
    }
 
    public void setName(String name) {
        this.name = name;
    }
 
    public Teacher getTeacher() {
        return teacher;
    }
 
    public void setTeacher(Teacher teacher) {
        this.teacher = teacher;
    }
}
```

### 4.依赖方法注入(lookup-method)
当一个单例的Bean，依赖于一个多例的Bean，用常规方法只会被注入一次，如果每次都想要获取一个全新实例就可以采用lookup-method 方法来实现。该操作的原理是基于动态代理技术，重新生成一个继承至目标类，然后重写抽像方法到达注入目的。

>前面说所单例Bean依赖多例Bean这种情况也可以通过实现 ApplicationContextAware 、BeanFactoryAware 接口来获取BeanFactory 实例，从而可以直接调用getBean方法获取新实例，推荐使用该方法，相比lookup-method语义逻辑更清楚一些。

```xml
<bean id="MethodInject" class="com.my.spring.MethodInject">
    <lookup-method name="getFine" bean="fine"></lookup-method>
</bean>
```

```java
// 编写一个抽像类
public abstract class MethodInject {
    public void handlerRequest() {
      	// 通过对该抽像方法的调用获取最新实例
        getFine();
    }
    // 编写一个抽像方法
    public abstract FineSpring getFine();
}
// 设定抽像方法实现
```

