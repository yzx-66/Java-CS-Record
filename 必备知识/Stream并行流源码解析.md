#### 并行流

并行流调试代码截图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020091314091064.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



**并行流源码**

核心类图（可以看出Fork-Join）
	![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913140949889.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


**和串行开始有区别的地方**

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913141010946.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)

  


---------------
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913141639677.png#pic_center)





-------------------
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913141149995.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913141825804.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


------------------------------------------------------------
去重：

 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913142240845.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)

----------------
  
 排序：

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913142259517.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


---------------------

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913142315616.png#pic_center)



-------------
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913142332319.png#pic_center)


  ------------------

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913142353991.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)

-------------


![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913142423564.png#pic_center)


--------------------

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913142442225.png#pic_center)

  ------------

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913142457831.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)

--------------
​	

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913142519205.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)







**小结：**

- 并行流对于有状态中间操作，会先设置其CombineFlag
- 然后遍历每个流，如果是有状态的，则调用opEvaluateParallelLazy() 返回该有状态操作执行后的流（因为java doc中说到有状态的操作，没法同时和无状态操作一起并行进行）
- ForkJoin进行任务拆分执行，其实现为ReduceTask，在其compute()进行拆分时，其拆分依据有两个，线程数是否够与迭代器是否还可再拆，可拆分的话会拆成两个子任务，但并不是让两个任务都fork加入任务队列，然后让两个子任务接着递归接着拆分，而是交替着放左半边元素的任务或右半边元素的任务，剩下没放入任务队列的任务，在继续在循环中拆分，这样的话就减少了一半放入任务队列的次数，也就减少了一半的递归次数
- 拆分到不能拆分后，会调用doLeaf()，执行这些元素的所用的操作，即wrapAndCopyInto()，然后把终止操作的Sink保存到自己的LocalResult
- 下面进行合并，最终调用的ReduceTask的OnCompeletion()，会用leftChildTask.combine(RightChildTask)，以此来保证返回的迭代器中元素位置，与之前在sourceSplitator() 执行完有状态操作后返回的迭代器中元素位置一样。
- 最后返回顶层Task的localResult。
