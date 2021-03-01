# 自动装配
**自动装配功能总体来说由 @EnableXXX注解 + @Import**
* 再配合@Conditional注解可以实现条件自动装配
* 在SpringBoot中核心注解为@EnableAutoConfiguration

## @EnableAutoConfiguration
通常情况下，springBoot应用启动类不会直接标注此注解，而是通过@SpringBootApplication注解来实现：

![img](https://img-blog.csdnimg.cn/img_convert/7d7166a1ab858e79dd2bcd745844be7e.png)

 发现 @SpringBootApplication中包含了 @SpringBootConfiguration（等同于@Configuration）、@EnableAutoConfiguration、@ComponentScan 注解。

> **总结：在启动类上加上 @EnableAutoConfiguration 注解 或者@SpringBootApplication即可实现自动装配，推荐使用 @SpringBootApplication这个组合注解。**

## @EnableAutoConfiguration 注解

依照 @EnableXXX的驱动设计
* @EnableAutoConfiguration 必然也是按照 @Import 配合 importSelector 或者 ImportBeandefinetionRegistrar 接口编程的套路

查看@EnableAutoConfiguration注解源码：

  ![img](https://img-blog.csdnimg.cn/img_convert/4338594ff2e607fd3885cda674dd5c8b.png)
果不其然，再进一步验证：  ![img](https://img-blog.csdnimg.cn/img_convert/d3014de6fca5d620b84eec93f45e10df.png)
关于 ImportSelector 的回调可以参考
* <a href = 'https://blog.csdn.net/weixin_43934607/article/details/114198142?spm=1001.2014.3001.5501 ' >Spring 注解驱动原理（三）：使用 annotatedClass 构造之注册配置类 </a>
* <a href = 'https://blog.csdn.net/weixin_43934607/article/details/114236207?spm=1001.2014.3001.5501 ' >Spring 注解驱动原理（四）：使用 annotatedClass 构造之 ConfigurationClassPostProcessor </a>


- 此时相信读者已经知道大致的脉络了，那么我们就重点分析一下 **AutoConfigurationImportSelector** 这个 ImportSelector实现。

## 回调逻辑
### DeferredImportSelector
正常情况下，若类实现了 ImportSelector接口，则会回调其相对于的 selectImports方法，但是我们通类的关系图发现 AutoConfigurationImportSelector 直接实现的是 DeferredImportSelector，而这个 ImportSelector 如下： 

 ![img](https://img-blog.csdnimg.cn/img_convert/325ff21d2b1158def76ca11d0b6d8db6.png)

  是在 Spring 4.0之后新增的延迟ImportSelector，且处理逻辑跟普通的 ImportSelector不同的是当前接口新定义了 Group的概念。

  ![img](https://img-blog.csdnimg.cn/img_convert/904b9b5b85697bbafa9e92d69b81dc03.png)

### ConfigartionClassParser.parse
**追踪 process 方法如下：**
![img](https://img-blog.csdnimg.cn/img_convert/4378a5453250d16614f4e5f080001d5a.png)
**processGroupImports**
![img](https://img-blog.csdnimg.cn/img_convert/2d2ab86f0482aee17b2e0e4960f42cfd.png)
**grouping.getImports()**

重点在于此处的 grouping.getImports()，我们发现是 ConfigurationClassParser的内部静态类 DeferredImportSelectorGrouping：
![img](https://img-blog.csdnimg.cn/img_convert/ff7f1298f47506c9269d9e011610d878.png)

  此类中的两个处理方法正正是关键的步骤，而这两个方法正是 DeferredImportSelector 中的内部接口 Group的实现去执行的。然后我们发现Group的方法默认实现是AutoConfigurationImportSelector的内部静态类AutoConfigurationGroup，如下：

  ![img](https://img-blog.csdnimg.cn/img_convert/46786cfb845ea83e3acf55a497edb2a3.png)
至此就调用到了 AutoConfigrationImportSelector 的 selectImport
