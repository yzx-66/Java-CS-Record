### IDEF1x工程化方法

* IDEF1x是将E-R模型扩充语义含义而形成的, 或者说，IDEF1x是E-R图的细化

* IDEF1x是一种进行数据建模或数据库设计的工程化的方法 



实体(Entity)

* 独立标识符实体/独立实体(Identifier-IndependentEntity)--强实体
* 从属标识符实体/从属实体(Identifier-dependentEntity)--弱实体



联系(Relationship)

* 可标定连接联系(IdentifyingConnectionRelationship)

* 非标定连接联系(Non-IdentifyingConnectionRelationship)

* 分类联系(CategorizationRelationship)

* 非确定联系(Non-SpecificRelationship)

  

属性/关键字(Attribute/Key)

* 属性(Attribute)主关键字/主码(PrimaryKeys)--主属性
* 次关键字/候选码(AlternateKeys)
* 外来关键字/外来码(ForeignKeys)--外来属性



示例

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201127090013903.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




#### 实体与属性

实体(Entity):一个“实体”表示一个现实和抽象事物的集合，这些事物必须具有相同的属性和特征。这个集合的一个元素就是该实体的一个实例。

* 实体被区分为独立实体和从属实体；

  * 独立实体：一个实体的实例都被唯一的标识而不决定于它与其他实体的联系
  * 从属实体：一个实体的实例的唯一标识需要依赖于该实体与其他实体的联系
    * 从属实体需要从其他实体继承属性作为关键字（主码）的一部分
    * 主关键字包含了外来属性的实体为从属实体

* 在扩展E-R图中，独立实体又称强实体，从属实体又称弱实体。

  * 实体用实体名/实体号标识
  * 独立实体用直角方形框，从属实体用圆角方形框表示
  * 独立实体的主关键字没有外键，从属实体的主关键字含有外键
  * 从属实体的实例依赖于独立实体实例存在而存在

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201127090027431.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)



* 工程化的要求
  * 每一个实体必须使用唯一的实体名，相同的含义总是用于同一实体名，相同的含义不能用于不同的实体名
  * 一个实体应有一个或多个能唯一标识实体每一个实例的属性,即应有一个主关键字及若干次关键字(0或多个)
  * 一个实体可以有一个或多个属性，这些属性可以是其自身所具有的，也可以是通过一个联系而继承得到的
  * 任意实体都可与模型中任意其他的实体有任何联系
  * 如果一个完全外来关键字是一个实体主关键字的全部或部分,那么该实体就是从属实体。相反，如果仅一部分或根本没有外来关键字属性用作一个实体的主关键字，那么，这个实体就是独立实体



属性和关键字

* 属性：表示一类现实或抽象事物的一种特征或性质。
* 关键字：能唯一确定实体每一个实例的属性或属性组。



* 关键字，被区分为主关键字和次关键字

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201127090045110.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)

* 外来关键字：是其他实体的关键字

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/202011270901005.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  

* 属性的工程化要求

  * 每个属性都必须有一个唯一的名称，且相同的名字必须总是描述相同的含义。因此相同的含义不可能对应于不同的名字(别名除外)
  * 每个实体可以具有任意个属性，一个属性只能归属于一个实体，这一规则称“单主规则”
  * 一个实体可有任意个继承属性，而每个继承属性都必须是某个相关的父亲实体或一般实体主关键字的一部分（为了无损连接）。
  * 实体的每一个实例，对每一个属性都必须具有一个值。这一规则称为“非空规则”
  * 对于同某实体相关的属性而言，该实体没有一个实例可能具有一个以上的值。这一规则称为“非重复规则”

* 主关键字和次关键字的工程化要求

  * 每个实体必须有一个主关键字，可有任意个次关键字
  * 主关键字和次关键字可由单个或多个属性组成
  * 个别属性可以是多个关键字的一部分
  * 构成主关键字或次关键字的属性可以是实体自身所具有的或由某些联系继承得到的属性
  * 主关键字和次关键字必须仅包含有助于唯一标识实体的那些属性。也就是说，如果主关键字或次关键字中去掉任一部分属性，那么都无法唯一确定实体的实例。此规则称“最小关键字规则”
  * 如主关键字是由多个属性组成，那么每个非键属性的值必须完全函数依赖于主关键字，也就是，如果主关键字的一部分属性被确定了，那么非键属性的值无法唯一确定。此规则称“完全函数依赖规则”
  * 每个非键属性必须是仅仅函数依赖于主关键字和次关键字，也就是，没有一个非键属性的值能够由其他非键属性的值所确定。此规则称“非传递依赖规则”

* 外来关键字的工程化要求

  * 在确定连接联系或分类联系中的儿子实体或分类实体时必须包含一个外来关键字
  * 一般实体的主关键字必须遗传为每一个分类实体的主关键字
  * 存在一个联系，只能有一个外来关键字
  * 被继承属性只能是主关键字所包含的属性
  * 分配给继承属性的每一个作用名(RoleName，即什么联系)都必须是唯一的，同时同一含义必须应用于同一作用名
  * 如果在某实体的任一给定实例中，对于两个外来关键字而言，单一遗传属性总是具有相同值，那么，该属性可以是多个外来关键字的部分





#### 联系

联系有连接联系、分类联系、和不确定性联系

* 连接联系，又称父子联系或依存联系，又可进一步区分为标定联系和非标定联系



连接联系

* 标定联系

  * 子实体的实例都是由它与父实体的联系而确定。父实体的主关键字是子实体主关键字的一部分
  * 标定联系用实直线表示，在子实体一侧有圆圈，联系名标注在直线旁

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201127090117633.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




* 非标定联系

  * 子实体的实例能够被唯一标识而无需依赖与其实体的联系。父实体的主关键字不是子实体的主关键字。
  * 非标定联系用虚直线表示,在子实体一侧有圆圈，联系名标注在直线旁

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/2020112709013519.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




* 工程化要求 
  * 一个确定性连接联系总是存在于两个实体之间，一个作为父实体，另一个作为子实体
  * 子实体的一个实例必须且总是恰好地与父实体的一个实例相联系
  * 父实体一个实例可与子实体的0个、1个或多个实例相联系，具体情况由特定基数而定。在子实体端标注P(1或大于1)/Z(0或1)/n(确定数目)/<省略>(0,1或大于1)
  * 在标定联系中的子实体总是一个从属标识符实体。
  * 一个实体可以与任意多个其他实体相联系，可以在不同的联系中充当不同的角色，如在一些联系中充当父实体，而在另外一些联系中充当子实体。



非确定联系

* 即实体之间的多对多的联系，非确定联系必须分解为若干个一对多的联系来表达。
* 非确定联系通过引入相交实体(Intersection Entity)或者称相关实体(Associative Entity)来分解为若干个一对多的联系来表达
  * 确定性联系通过属性继承实现两实体之间的联系
  * 非确定性联系通过引入相交实体实现两实体的联系

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201127090151322.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)




* 工程化要求
  * 一个非确定联系总是存在于两个实体之间，而不是三个或更多个实体之间
  * 两个实体中,任意一个实体的实例可以与另一实体的0,1或多个实例相关联，具体情况要视情况而定，在图中标出其基数
  * 为了完全地设计出一个模型，非确定联系必须由确定联系来替代



分类联系

* 一个实体实例是由一个一般实体实例及多个分类实体实例构成的

  * 一个一般实体是若干具体实体(分类实体)的类
  * 分类实体与一般实体具有相同的主关键字
  * 不同分类实体除具有一般实体特征外，各自还可能具有不同的属性特征

* 具体化

  * 实体的实例集中，某些实例子集具有区别于该实例集内其它实例的特性，可以根据这些差异特性对该实例集进行分组/分类，这一分组/分类的过程称作具体化
  * 自顶向下、逐步求精
  * (面向对象中的)父类--子类
  * 子类=\=特例 ==更小的实例集合=\=更多的属性
  * 示例：学生可以有研究生、本科生。研究生有“论文”属性，而本科生有“军训”属性。

* 泛化

  * 若干个实体根据共有的性质，可以合成一个较高层的实体。泛化是一个高层实体与若干个低层实体之间的包含关系
  * 自底向上、逐步合成
  * 泛化与具体化是个互逆的过程
  * 具体化强调同一实体不同实例之间的差异属性，泛化强调不同实体之间的相似属性
  * 反映了数据库设计或数据库抽象的不同思路或方法：自底向上或者自顶向下

* 具体化和泛化在E-R图中用标记为标记为ISAISA的三角形的三角形来表示

  * ISA=“is-a”,表示高层实体和低层实体之间的“父类－子类”联系
  * 在IDEF1X中具体化和泛化表征的就是一种分类联系

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201127090207179.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


* 属性继承

  * 高层实体的属性被低层实体自动继承
  * 低层实体特有的性质仅适用于某个特定的低层实例
  * 如“Dissertation”属性只适用于“研究生”实例

* IDEF1X表示

  * 一圆圈带两横线：完全分类联系

  * 一圆圈带一横线：非完全分类联系

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201127090220391.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  * 示例：

  ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201127090234343.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  

* 工程化要求

  * 一个分类实体只能有一个对应的一般实体，即对一分类联系而言，它只能是一个分类集的成员
  * 一个分类联系中的一个分类实体可以是一个其他分类联系中的一般实体
  * 一个实体可以具有任意个分类联系，在这些分类联系中，这个实体作为一般实体。例如“雇员”实体可分类为“计时雇员”和“月薪雇员”，也可分类为“普通雇员”和“高级雇员”
  * 一个分类实体不能是可标定联系中的子实体
  * 分类实体的主关键字属性必须和一般实体主关键字属性相同。
  * 一个分类实体的全部实例都具有相同的“鉴别器值”，并且不同分类实体的实例都具有不同的鉴别器值

* 最后强调

  * **分类联系 =\\= 分类，分类实体必须有特有的属性，否则分类没有意**
