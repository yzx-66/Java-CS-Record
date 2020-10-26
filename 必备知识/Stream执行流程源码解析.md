##### Stream

使用

- Stream中的操作可以分为两大类：中间操作与结束操作，中间操作只是对操作进行了记录，只有结束操作才会触发实际的计算（即惰性求值），这也是Stream在迭代大集合时高效的原因之一。

	![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913134035731.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)

- 注意事项

  - 不要使用基本类型的数组获取流，因为得到的流还是数组
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913135128818.png#pic_center)


  

  - 流的执行顺序是一个元素会把流执行到底，并不是一个环节全部收集后再一起执行下一环节

  	![在这里插入图片描述](https://img-blog.csdnimg.cn/2020091313405747.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




核心类图

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913134122438.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


上面截图代码执行源码：

- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913134152254.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913134332910.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913134348287.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



- ![](https://img-blog.csdnimg.cn/20200913134403975.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




- ![](https://img-blog.csdnimg.cn/2020091313442737.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




- ![](https://img-blog.csdnimg.cn/2020091313444353.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




- ![](https://img-blog.csdnimg.cn/20200913134502280.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




- ![](https://img-blog.csdnimg.cn/20200913134623760.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913140228224.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)





- ![](https://img-blog.csdnimg.cn/20200913134913328.png#pic_center)




- ![](https://img-blog.csdnimg.cn/20200913134930114.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




- ![在这里插入图片描述](https://img-blog.csdnimg.cn/2020091313495886.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




- ![](https://img-blog.csdnimg.cn/20200913135012560.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913135030182.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




- ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200913135043990.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



总结：

- 获取一个流或者进行一个中间操作，都会获取一个AbstractPipeline的节点，一般都是ReferencePipeline。然后必须要实现ReferencePipeline的抽象方法onWrapSink()，返回一个Sink。返回的Sink必须针对不同的Stream有不同的是实现，其中包括begin()、accept()，但是end()一般都空实现
- 在进行结束操作的时候，不会再创建一个Stream，而是创建一个TerminalOP，其包含一个makeSink()可以返回终止操作的Sink。然后回调用TerminalOP的evaluate()，并把最后一个Stream向上转型为PipelineHelper作为参数传入
- 调用传入的helper的warpAndCopyInto()，在这个方法又会调用先调用wrapSink() 将所有sink连起来，通过在最后一个Stream向前进行遍历AbstractPipeline的链表，并依次调用每个pipeline的onWrapSink()，并把返回的sink作为下一个遍历到Stream的onWrapSink()的downStream参数，同时又因为第一个传入的参数是终止操作的sink，所以就形成了终止操作的Sink在最里层Sink链。然后warpAndCopyInto()会调用copyInto()，该方法会先执行sink链的start()，然后遍历元素的迭代器，依次给每个元素调用sink链的accept()，最后调用sink链的end()。
- 在copyInto()方法中，在调用到终止操作的sink时，会把每个元素的执行结果保存到state属性中，最后只要返回该sink的state就是最终结果。
