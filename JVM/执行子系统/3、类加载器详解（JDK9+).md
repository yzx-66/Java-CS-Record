#### 模块化

##### 如何向前兼容

* 具名模块(Named Module)

  具名模块也称为应用模块(Application Module)，通常在模块根目录下有 module-info.java 文件的话，那么该模块就称为具名模块，我们编写的模块一般都属于这种类型。

* 无名模块(Unnamed Module)：不分模块的 jar 包，放到 不分模块的路径（即这个项目类路径下）

  无名模块指的就是不包含 module-info.java 的 jar 包，通常这些 jar 包都是 Java 9 之前构建的。无名模块可以读取到其他所有的模块，并且也会将自己包下的所有类都暴露给外界。

  * 需要注意的是无名模块导出了所有的包，但并不意味着任何具名的模块可以读取无名模块，因为具名模块在 module-info.java 中无法声明对无名模块的依赖，无名模块导出所有包的目的在于让其他无名模块可以加载这些类。

* 自动模块(Automatic Module)：不分模块的 jar 包，方法 分模块的路径（即某个模块下）

  任何无名模块(没有 module-info.java 的模块)放到模块路径(module path)上会自动变为自动模块，允许 Java 9 模块能引用到这些模块中的类。自动模块对外暴露所有的包并且能引用到其他所有模块的类，其他模块也能引用到自动模块的类。

  * 由于自动模块并不能声明模块名，那么 JDK 会根据 jar 包名来自动生成一个模块名以允许其他模块来引用。生成的模块名按以下规则生成：首先会移除文件扩展名以及版本号，然后使用".“替换所有非字母字符。例如 spring-core-4.3.12.jar 生成的模块名为 spring.core，那么其他模块就可以通过 requires spring.core 来引用其中的类。



##### JDK 9类加载器

扩展类加载器（Extension Class Loader）被平台类加载器（Platform Class Loader）取代。

* 整个JDK都基于模块化进行构建（原来的rt.jar和tools.jar被拆分成数十个JMOD文件），其中的Java类库就已天然地满足了可扩展的需求，因为分成了更小颗粒，可以对 moudle 进行组合，而并非都是固定某个 jar，那自然无须再保留<JAVA_HOME>\lib\ext目录，此前使用这个目录或者java.ext.dirs系统变量来扩展JDK功能的机制已经没有继续存在的价值了，用来加载这部分类库的扩展类加载器也完成了它的历史使命。
* 类似地，在新版的JDK中也取消了<JAVA_HOME>\jre目录，因为随时可以组合构建出程序运行所需的JRE来，譬如假设我们只使用java.base模块中的类型，那么随时可以通过以下命令打包出一个“JRE”：`jlink -p $JAVA_HOME/jmods --add-modules java.base --output jre`



jdk 8 和 jdk 9 后默认类加载比较

* BootClassLoader（jdk 9 之后有了BootClassLoader 的Java类）
  * 从下面的 loadClassOrNull 方法可以看出，其会调用 native 方法查找启动类加载器加载了哪些类
  * 同时为了兼容 java 9 之前的版本，BootClassLoader 在 ClassLoader#loadClass 时仍然是 null）
  * ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026152413542.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


-------------------

* ExtClassLoader -> PlatformClassLoader
  * classpath 为 null，这个是和 AppClassLoader 的唯一区别，所以只能加载 jvm 内部指定的 moudle 和 package，不可以加载用户指定的相关类。
  * ​	![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026152428918.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


------------------------

* AppClassLoader
  * ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026152449186.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




* 结论就是
  * jdk 8 时，拓展类加载器和用户类加载都是继承的 UrlClassLoader，jdk 11 之后，三个默认的累加载器都继承了 BuiltinClassLoader
  * BuiltinClassLoader 和 UrlClassLoader  对比
    * 原理差不多，都是基于 UrlClassPath 来实现查找的。
    * 但 BuiltinClassLoader  支持从 moudle 加载 class。
    * 还有和通常的双亲委派不同，如果一个 class 属于某个 moudle 那么会直接调用该 moudle 的类加载器去加载，而不是说直接用当前类加载器的双亲委派模型去加载。但是找到这个 class 对应的类加载器后，还是会按照双亲委派去加载。





#### 类加载器

类加载器可以说是Java语言的一项创新，让应用程序自己决定如何去获取所需的类的二进制字节流



##### 类与类加载器

对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在Java虚拟机中的唯一性。

* 比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个Java虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等。

* 这里所指的“相等”，包括代表类的Class对象的equals()方法、isAssignableFrom()方法、isInstance()方法的返回结果，也包括了使用instanceof关键字做对象所属关系判定等各种情况。

  ·

  

##### 双亲委派模型

* 类加载器的双亲委派模型在JDK 1.2时期被引入，并被广泛应用于此后几乎所有的Java程序中，但它**并不是一个具有强制性约束力的模型**，而是Java设计者们推荐给开发者的一种类加载器实现的最佳实践。



启动类加载器（Bootstrap Class Loader）

* jdk 9 之后有了BootClassLoader 的Java类，和平台类加载与应用类加载器一样看，都继承了BuitingClassLooader，这是为了 java 9 模块化而产生的一个类加载器，虽然和之前继承 UrlClassLoader 一样，都是基于 UrlClassPath 的，但拓展了可以加载模块，还有重写了 loadClass 破坏了双亲委派模型。
* 对于 BootClassLoader ，其重写了 loadClassOrNull 方法，通过调用 native 方法去实现类加载，是一个媒介的作用，本身并不实现类加载，docs 上说主要作用是查找启动类加载器加载了哪些类，和 jdk 8 时的 BootSClassPathHolder 一个功能。
* 同时为了兼容 java 9 之前的版本，BootClassLoader 在 ClassLoader#loadClass 时仍然是 null。





扩展类加载器（PlatformClassLoader）

* jdk9 之后用来代替　ExtClassLoader 的加载器，用来加载 jdk 中的非核心模块类。

* 由Java代码实现的，但是 PlatformClassLoader 的 classpath 为 null，所以只能加载jdk 指定部分的 package 和 module。

  



应用程序类加载器（Application Class Loader）

* 它负责**加载用户类路径（ClassPath）上所有的类库**，开发者同样可以直接在代码中使用这个类加载器。如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器。
* new 一个 ClassLoader对象时，如果不指定 parent，那么其parent 就是 getSystemClassLoader() 的返回值，该值默认就是 AppClassLoader





JDK 9 及以后的类加载器委派关系



![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026152507491.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)

* JDK 9中虽然仍然维持着三层类加载器和双亲委派的架构
* 但当平台及应用程序类加载器收到类加载请求，在委派给父加载器加载前，要先判断该类是否能够归属到某一个系统模块中，如果可以找到这样的归属关系，就要优先委派给负责那个模块的加载器完成加载，所以说就多了 平台类加载器对于应用类加载的委派，和应用类加载器对启动类加载器的委派





原理

* 核心 ClassLoder#loadClass

  ```java
  // name 必须要是全限定类名，既要加上包名，不能只是类名
  protected synchronized Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException{
      // 首先，检查请求的类是否已经被加载过了
      // 存在要满足 同一个 classLoader 对象，且 是同一个全限定类名（name）
      Class c = findLoadedClass(name);
      if (c == null) {
          try {
              if (parent != null) {
                  c = parent.loadClass(name, false);
              } else {
                  c = findBootstrapClassOrNull(name);
              }
          } catch (ClassNotFoundException e) {
              // 如果父类加载器抛出ClassNotFoundException
              // 说明父类加载器无法完成加载请求
          }
          if (c == null) {
              // 在父类加载器无法加载时,再调用本身的findClass方法来进行类加载
              // 这个方法一般是留给自定义类加载器来重写的，在这个方法获得 class 文件的字节码数组之后，调用 defineClass 加载
              c = findClass(name);
          }
      }
      if (resolve) {
      	resolveClass(c);
     	}
      return c;
  }
  ```

  

* 下面讲解一下 UrlClassLoader（所有代码都省略了安全校验）

  ```java
  public URLClassLoader(URL[] urls, ClassLoader parent) {
      // 指定父加载器
      super(parent); 
      this.acc = AccessController.getContext();
      // 构造 URLClassPath（核心）
      this.ucp = new URLClassPath(urls, acc);
  }
  
  public URLClassLoader(URL[] urls) {
      // 没指定默认会在 ClassLoder#ClassLoder() 构造中调用 getSystemClassLoader() 获得 parent
      super();
      this.acc = AccessController.getContext();
      // 构造 URLClassPath（核心）
      this.ucp = new URLClassPath(urls, acc);
  }
  
  // 自定类加载重写了 ClassLoader 的 findClass()，不破坏 loadClass 的规则。
  protected Class<?> findClass(final String name) throws ClassNotFoundException{
      final Class<?> result;
      // 先获得path
      String path = name.replace('.', '/').concat(".class");
      
      // 这个是 UrlClassPath 是 BuiltinClassLoader（是 Boot 和 UrlClassLoader 的核心，要获得class文件就是通过这个 UrlClassPath
      // 一句话总结：就是 urlClassPath 里面维护了 loaders 集合，每个 loader 对应一个构造时传入的正确的 url，
      //  loader 有三种 FileLoader、JarLoader、Loader（对应 protocol 不是 file 也不是 jar）
      //  每次在 ucp 查找时就是遍历 loaders，依次调用每个 Loader#getResource，如果 Loader 对应的绝对 path 下存在要查找的 
      //  相对 path 那么就返回该相对路径对应的 Resource，否则返回 null。
      Resource res = ucp.getResource(path, false);
      
      // 下面省略了AccessController的校验
      if (res != null) {       
          try {        
              // ClassLoader#defineClass，加载class文件的字节数组到 jvm
              return defineClass(name, res);            
          } catch (IOException e) {            
              throw new ClassNotFoundException(name, e);              
          }        
      } else {                  
          return null;                  
      }
     
      if (result == null) {      
          throw new ClassNotFoundException(name)       
      }      
      return result;  
  }
  ```

  

破坏双亲委派的方式

* loadclass 被重写了，正常情况下是重写 findClass，这样会在父类没有加载该类时，调用当前类加载器的 findClass
* jdk 9 的 BuiltinClassLoader，加载任务委派给模块的加载器，而不是父加载器。
* 加载器内部使用线程上下文加载器，即 父加载器 把加载任务委派给了 子加载器。
  * 线程上下文加载器是为了解决父加载器，想要使用子加载器的场景，因为要获取子加载器对应classpath下的文件时，只有获取到子加载器。
  * 通过java.lang.Thread类的setContext-ClassLoader()方法进行设置，如果创建线程时还未设置，它将会从父线程中继承一个，如果在应用程序的全局范围内都没有设置过的话，那这个类加载器默认就是应用程序类加载器。

* OSGi 热部署，不再双亲委派模型推荐的树状结构（一个加载器只有一个父加载器），而是进一步发展为更加复杂的网状结构（一个加载器可以有多个父加载器），按照一定规则决定委派给哪个加载器





##### 案例

* 核心原理就是双亲委派，因此在当前类的 classLoader 在 loadClass 的时候，只可以使用 当前类加载器 和 所有父类加载器  classpath 下的类，所以不在这一路径上的其他 classLoader 加载的 classpath 是无法被被使用的，比如 子 classloader 或者 同级 classLoader 或者 父的同级 classLoader。
* 如果要让父类加载器 classpath 里的类，使用子类加载器 classpath 中的类，那么只有破坏双亲委派，或者使用线程上下文加载器。



###### Tomcat

* 放置在/common目录中。类库可被Tomcat和所有的Web应用程序共同使用。
* 放置在/server目录中。类库可被Tomcat使用，对所有的Web应用程序都不可见。
* 放置在/shared目录中。类库可被所有的Web应用程序共同使用，但对Tomcat自己不可见。
* 放置在/WebApp/WEB-INF目录中。类库仅仅可以被该Web应用程序使用，对Tomcat和其他Web应用程序都不可见。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026152520457.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




当app下面的class文件修改的时候，Tomcat更新步骤：

1. ContextWrapper 会有专门线程监控app下面的类的修改情况。 
2. 如果发现有类被修改了。那么调用 Context.reload()。清楚一系列相关的引用和资源。 
3. 然后创新创建一个WebappClassLoader实例，重新加载app下面需要的class。



当Jsp文件修改的时候，Tomcat更新步骤：

1. 当访问.jsp文件的时候，.jsp 的包装类 JspServletWrapper 会去比较 .jsp 文件最新修改时间和上次的修改时间，以此判断 .jsp是否修改过。 
2. .jsp修改过的话，那么 jspservletWrapper 会清除相关引用，包括 jsp编译后的 servlet 实例和加载这个 servlet 的 JasperLoader 实例。 
3. 重新创建一个 JasperLoader 实例，重新加载修改过后的.jsp，重新生成一个Servlet实例。 



###### OSGi

假设：

* Bundle A：声明发布了 packageA，依赖了 java.*的包；
* Bundle B：声明依赖了packageA 和 packageC，同时也依赖了 java.*的包；
* Bundle C：声明发布了 packageC，依赖了 packageA。



 继承关系：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026152533128.png#pic_center)




委托规则：

* 以java.*开头的类，委派给父类加载器加载。
* 否则，委派列表名单内的类，委派给父类加载器加载。
* 否则，Import列表中的类，委派给Export这个类的Bundle的类加载器加载。
* 否则，查找当前Bundle的Classpath，使用自己的类加载器加载。
* 否则，查找是否在自己的Fragment Bundle中，如果是则委派给Fragment Bundle的类加载器加载。
* 否则，查找Dynamic Import列表的Bundle，委派给对应Bundle的类加载器加载。
* 否则，类查找失败。
