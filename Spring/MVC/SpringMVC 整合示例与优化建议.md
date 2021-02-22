### 1.web.xml中添加spring相关配置
在web.xml中需要配置spring上下文监听器和springmvc的servlet，并且指定spring上下文配置文件和springmvc配置文件，具体配置如下：

```xml
<!--spring监听器，指定spring配置文件-->
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring-context.xml</param-value>
</context-param>
<listener>
    <listener-class>com.yzh.tally.manager.listener.WebContextLoaderListener</listener-class>
</listener>

<!--spring mvc拦截器-->
<servlet>
   <servlet-name>DispatcherServlet</servlet-name>
   <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
   <init-param>
       <param-name>contextConfigLocation</param-name>
       <!--指定mvc配置文件-->
       <param-value>classpath:spring-mvc.xml</param-value>
   </init-param>
   <load-on-startup>1</load-on-startup>
</servlet>

<!--DispatcherServlet 拦截器路径-->
<servlet-mapping>
     <servlet-name>DispatcherServlet</servlet-name>
     <url-pattern>/*</url-pattern>
 </servlet-mapping>
```

如上所示，配置好Servlet拦截器以后，该web应用下的所有请求都会经过DispatcherServlet进行处理，这个时候你就会发现 js、css、图片等一系列静态资源就无法访问了，这可如何是好呢？不用紧张，其实只需要再添加默认的servlet进行拦截就ok了。

```xml
<!--静态资源-->
 <servlet-mapping>
     <servlet-name>default</servlet-name>
     <url-pattern>/js/*</url-pattern>
 </servlet-mapping>

 <servlet-mapping>
     <servlet-name>default</servlet-name>
     <url-pattern>/images/*</url-pattern>
 </servlet-mapping>
 <servlet-mapping>
     <servlet-name>default</servlet-name>
     <url-pattern>/css/*</url-pattern>
 </servlet-mapping>
```

### 2.配置 spring-context.xml
在spring上下文配置中，主要配置properties资源文件，数据访问，如下配置所示：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:context="http://www.springframework.org/schema/context"
      xsi:schemaLocation="http://www.springframework.org/schema/beans 
      http://www.springframework.org/schema/beans/spring-beans.xsd
      http://www.springframework.org/schema/context
      http://www.springframework.org/schema/context/spring-context.xsd">
   <!--自动扫描-->
   <context:annotation-config/>
   <!--配置properties资源文件-->
   <context:property-placeholder location="classpath:config.properties"/>
   <!--配置bean-->
   <bean class="com.yzh.tally.manager.config.DevplatformCfg"></bean>
   <import resource="spring-jdbc.xml"/>
</beans>
```

spring-jdbc.xml的配置如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:tx="http://www.springframework.org/schema/tx"
      xmlns:context="http://www.springframework.org/schema/context"
      xsi:schemaLocation="
       http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/tx
       http://www.springframework.org/schema/tx/spring-tx.xsd
       http://www.springframework.org/schema/context
       http://www.springframework.org/schema/context/spring-context.xsd">

   <context:component-scan base-
   package="com.yzh.tally.sdk.service,com.yzh.tally.sdk.dao,com.yzh.tally.sdk.impl"/>

   <bean id="jdbcHelper" class="com.yzh.tally.sdk.spjdbc.impl.MySqlJdbcHelper">
       <property name="dataSource" ref="dataSource"></property>
   </bean>

   <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource" destroy-method="close">
       <property name="driverClass" value="${jdbc.driverClassName}"/>
       <property name="jdbcUrl" value="${jdbc.url}"/>
       <property name="user" value="${jdbc.username}"/>
       <property name="password" value="${jdbc.password}"/>
       <property name="maxPoolSize" value="${jdbc.maxPoolSize}"/>
       <property name="minPoolSize" value="${jdbc.minPoolSize}"/>
       <property name="initialPoolSize" value="${jdbc.initialPoolSize}"/>
       <property name="maxIdleTime" value="${jdbc.maxIdleTime}"/>
       <property name="preferredTestQuery" value="select 1"/>
   </bean>

   <!--事物控制-->
   <tx:annotation-driven/>

   <bean id="transactionManager" 
   class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
       <property name="dataSource" ref="dataSource"/>
   </bean>
</beans>
```

这里我用的是注解方式的事物管理，所以没有配aop。

### 3.配置spring-mvc.xml

在mvc配置文件中配置视图解析器、类型转换支持、拦截器、文件上传限制等。我用的视图层是velocity，你可以根据自己的需求配置为framemaker或jsp。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:mvc="http://www.springframework.org/schema/mvc"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xmlns:context="http://www.springframework.org/schema/context"
      xsi:schemaLocation="http://www.springframework.org/schema/beans 
                          http://www.springframework.org/schema/beans/spring-beans.xsd
      http://www.springframework.org/schema/mvc
      http://www.springframework.org/schema/mvc/spring-mvc.xsd 
                          http://www.springframework.org/schema/context 
                          http://www.springframework.org/schema/context/spring-context.xsd
      http://www.springframework.org/schema/context
      http://www.springframework.org/schema/context/spring-context.xsd">

   <context:annotation-config/>
   <context:component-scan base-package="com.yzh.tally.manager.controller"/>

   <bean id="conversionService" 
         class="org.springframework.context.support.ConversionServiceFactoryBean">
       <property name="converters">
           <set>
               <bean class="com.yzh.common.CommonConvert"/>
               <bean class="com.yzh.common.StringToDateConvert"/>
           </set>
       </property>
   </bean>
   <mvc:annotation-driven conversion-service="conversionService">
       <mvc:message-converters register-defaults="false">
           <bean class="org.springframework.http.converter.StringHttpMessageConverter">
               <property name="supportedMediaTypes">
                   <list>
                       <value>text/plain;charset=UTF-8</value>
                       <value>text/html;charset=UTF-8</value>
                   </list>
               </property>
           </bean>
           <bean 
              class="org.springframework.http.converter.json.MappingJackson2HttpMessageConverter">
               <property name="objectMapper">
                   <bean class="com.fasterxml.jackson.databind.ObjectMapper">
                       <property name="serializationInclusion" value="NON_NULL"/>
                   </bean>
               </property>
               <property name="supportedMediaTypes">
                   <list>
                       <value>application/json;charset=UTF-8</value>
                       <value>application/x-www-form-urlencoded;charset=UTF-8</value>
                   </list>
               </property>
           </bean>
       </mvc:message-converters>
   </mvc:annotation-driven>

   <!--velocity视图解析器-->
   <bean id="velocityConfig" 
         class="org.springframework.web.servlet.view.velocity.VelocityConfigurer">
       <property name="resourceLoaderPath" value="/WEB-INF/html/"></property>
       <property name="configLocation" value="/WEB-INF/velocity.properties"></property>
   </bean>
   <bean id="velocityViewResolver" 
         class="org.springframework.web.servlet.view.velocity.VelocityViewResolver">
       <property name="suffix" value=".vm"></property>
       <property name="prefix" value=""></property>
       <property name="cache" value="true"></property>
       <property name="exposeSpringMacroHelpers" value="true"/>
       <property name="exposeRequestAttributes" value="true"/>
       <property name="exposeSessionAttributes" value="true"/>
       <property name="contentType" value="text/html;charset=UTF-8"/>
       <property name="toolboxConfigLocation" value="/WEB-INF/toolbox.xml"></property>
   </bean>

   <!--文件上传表单的视图解析器-->
   <bean id="multipartResolver" 
         class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
       <property name="defaultEncoding" value="UTF-8"></property>
       <property name="maxUploadSize" value="209715200"></property><!--文件上传大小限制 200m-->
   </bean>


   <import resource="spring-interceptor.xml"/>
</beans>
```

另外，spring-interceptor.xml的配置

这里配置的就是拦截器了，拦截器一般都是用作登录校验，权限检查等的拦截。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:mvc="http://www.springframework.org/schema/mvc"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="
       http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/mvc
       http://www.springframework.org/schema/mvc/spring-mvc.xsd">
   <mvc:interceptors>
       <mvc:interceptor>
           <mvc:mapping path="/**"/>
           <mvc:exclude-mapping path="/"/>
           <mvc:exclude-mapping path="/index"/>
           <mvc:exclude-mapping path="/login"/>
           <mvc:exclude-mapping path="/dologin"/>
           <mvc:exclude-mapping path="/vcode"/>
           <mvc:exclude-mapping path="/material/**"/>
           <mvc:exclude-mapping path="/article/**"/>
           <mvc:exclude-mapping path="/wx/**"/>
           <bean class="com.yzh.tally.manager.interceptor.LoginInterceptor"></bean>
       </mvc:interceptor>
   </mvc:interceptors>
</beans>
```

到此为止一个完整的web应用就搭建完成了。

### Spring MVC使用优化建议

1.Controller如果能保持**单例**

尽量使用单例这样可以减少创建对象和回收对象的开销。也就是说，如果Controller的类变量和实例变量可以以方法形参声明的尽量以方法的形参声明，不要以类变量和实例变量声明，这样可以避免**线程安全问题**。 

2.@RequestParam注解

处理Request的方法中的形参务必加上@RequestParam注解，这样可以**避免Spring MVC使用asm框架读取class文件获取方法参数名的过程**。即便Spring MVC对读取出的方法参数名进行了缓存，如果不要读取class文件当然是更好。

```java
@RequestMapping("/query")
public void query(@RequestParam("id") Integer id){}
public void query(Integer id){} // 避免这样写，要加上@RequestParam注解
```

3.缓存URL

Spring MVC 在源码中并没有对处理 url 的方法进行缓存，也就是说每次都要根据请求**url去匹配Controller中的方法url**，如果把url和Method的关系缓存起来，会不会带来性能上的提升呢？

有点恶心的是，负责解析url和Method对应关系的 ServletHandlerMethodResolver 是一个 private的内部类，不能直接继承该类增强代码，必须要该代码后重新编译。当然，如果缓存起来，必须要考虑缓存的线程安全问题。
