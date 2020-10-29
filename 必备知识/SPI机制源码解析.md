**概念**

​	SPI 全称为 (Service Provider Interface) ，是JDK内置的一种服务提供发现机制。SPI是一种动态替换发现的机制， 比如有个接口，想运行时动态的给它添加实现，你只需要添加一个实现。

​	当服务的提供者提供了一种接口的实现之后，需要在classpath下的`META-INF/services/`目录里创建一个以服务接口命名的文件，这个文件里的内容就是这个接口的具体的实现类。


**没有jdk spi机制的不足：**

* 需要遍历所有的实现，并实例化，然后才能在循环中才能找到需要的实现，做不到按需加载 。
* 配置文件中只是简单的列出了所有的扩展实现，而没有给它们命名。导致在程序中很难去准确的引用它们，dubbo spi使用别名方便查找。
* 扩展如果依赖其他的扩展，做不到自动注入和装配。
* 不提供类似于Spring的IOC和AOP功能。
* 扩展很难和其他的框架集成，比如扩展里面依赖了一个Spring bean，原生的Java SPI不支持

使用：

- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913143121501.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




创建源码：

- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913143140925.png#pic_center)


--------------------------------

- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913143155973.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




获取源码（说白了就还是调用的 UrlClassPath 的 findResources，然后在外层进行遍历，只要搞懂 UrlClassPath#findResources 那么 spi 就懂了） ：

- 时序图：

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913143218791.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


- 核心类UrlClassPath概览：

	![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913143236656.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)






源码：

- 
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913145748973.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)

--------------------------------



- ![在这里插入图片描述](https://img-blog.csdnimg.cn/202009131433415.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)
--------------------------------
- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913145158184.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



--------------------------------

- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913143412287.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)
![](https://img-blog.csdnimg.cn/20200913143444288.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



--------------------------------
- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913143605432.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



--------------------------------
- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913143622286.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



--------------------------------
- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913143637291.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)
-------------------------------- 


*  <img src = 'https://img-blog.csdnimg.cn/2020091314365129.png' align = 'left' />
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913144535919.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)
