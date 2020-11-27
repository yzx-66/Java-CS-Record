### 基本SQL语言
SQL语言是集DDL、DML和DCL于一体的数据库语言

SQL语言主要由以下9个单词引导的操作语句来构成，但每一种语句都能表达复杂的操作请求

* DDL语句引导词：**Create**(建立),**Alter**(修改),**Drop**(撤消)
  * 模式的定义和删除，包括定义Database,Table,View,Index,完整性约束条件等，也包括定义对象(RowType行对象,Type列对象)
* DML语句引导词：**Insert** ,**Delete**,**Update**,**Select**
  * 各种方式的更新与检索操作，如直接输入记录，从其他Table(由SubQuery建立)输入
  * 各种复杂条件的检索，如连接查找，模糊查找，分组查找，嵌套查找等
  * 各种聚集操作，求平均、求和、...等，分组聚集，分组过滤等
* DCL语句引导词：**Grant**,**Revoke**
  * 安全性控制：授权和撤消授权


#### DDL

DDL:  Data  Definition Language（通常由DBA来使用，也有经DBA授权后由应用程序员来使用）

* 创建数据库(DB)—Create Database
* 创建DB中的Table(定义关系模式)---Create Table
* 定义Table及其各个属性的约束条件(定义完整性约束)---Create Trigger
* 定义View (定义外模式及E-C映像)---Create View
* 定义Index、Tablespace...  ...等(定义物理存储参数)---Create Index 、...
* 上述各种定义的撤消与修正---Alert、Drop



创建Database

* 数据库(Database)是若干具有相互关联关系的Table/Relation的集合

* 数据库可以看作是一个集中存放若干Table的大型文件

* create database的简单语法形式：

  ```sql
  create database 数据库名;
  ```

  



创建Table

* create table简单语法形式：

  ```sql
  Create tabletable 表名 ( 列名 数据类型 [Primary key |Unique] [Not null] 
                         [,列名 数据类型[Not null] , ... ]) ;
  ```

  * “[ ] ”表示其括起的内容可以省略，“| ”表示其隔开的两项可取其一
  * Primary key:主键约束。每个表只能创建一个主键约束。
  * Unique:唯一性约束(即候选键)。可以有多个唯一性约束。
  * Not null: 非空约束。是指该列允许不允许有空值出现，如选择了Not null表明该列不允许有空值出现。
  * 语法中的数据类型在SQL标准中有定义
    * char (n) :固定长度的字符串varchar (n) :可变长字符串
    * int:整数  //  有时不同系统也写作integer
    * numeric (p，q) :固定精度数字，小数点左边p位，右边p-q位
    * real  :浮点精度数字 //有时不同系统也写作float(n)，小数点后保留n位
    * date:日期 (如 2003-09-12)
    * time  : 时间 (如 23:15:003)



修正

* 修正数据库：修正数据库的定义，主要是修正表的定义

* 修正基本表的定义

  ```sql
  alter   table tablename
  [add  {colname  datatype, ...}]    // 增加新列
  [modify {colname  datatype, ...}]  // 修改列定义
  [drop {完整性约束名}]                // 删除完整性约束
  ```



撤销

* 撤消基本表：drop   table表名

* 撤消数据库：drop   database数据库名;





#### DML

DML:  Data  Manipulation Language（通常由用户或应用程序员使用，访问经授权的数据库）

* 向Table中追加新的元组：Insert
* 修改Table中某些元组中的某些属性的值: Update
* 删除Table中的某些元组: Delete
* 对Table中的数据进行各种条件的检索: Select



检索语句Select

* Select的简单语法形式：

  ```sql
  Select 列名[[, 列名] ... ] From 表名[Where 检索条件] ;
  ```

  * 语义：从表名所给出的表中，查询出满足检索条件的元组，并按给定的列名及顺序进行投影显示。
  * 相当于： $\prod_{列名, ... , 列名}$($\sigma_{检索条件}$(表名))
  * Select语句中的 select ... , from... , where...,  等被称为子句，在以上基本形式基础上会增加许多构成要素，也会增加许多新的子句，满足不同的需求。

  



* 结果唯一

  * 关系模型不允许出现重复元组。但现实DBMS，却允许出现重复元组，但也允许无重复元组。

  * 在Table中存在的元组无重复是通过定义Primary key或Unique来保证的；而在检索结果中要求无重复元组, 是通过DISTINCT保留字的使用来实现的。

    ```sql
    SELECT DISTINCT FROM-WHERE
    ```



* 模糊查询

  * 含有like运算符的表达式列名

    ```sql
    列名 [not] like “字符串”
    ```

    * 找出匹配给定字符串的字符串。其中给定字符串中可以出现%, \_等匹配符.
    * 匹配规则：
      * “%”匹配零个或多个字符
      * “\_”匹配任意单个字符
      * “\”转义字符，用于去掉一些特殊字符的特定含义，使其被作为普通字符看待, 如用“\%”去匹配字符%，用"\\_" 去匹配字符\_



* 连接

  * 直接在 From 后写多个表，然后在 select 中写连接条件即可

  ```sql
  SELECT  FROM tb1 tb2 .. tbn WHERE ...
  ```

  * 和 join-on 的 inner join 等价的，on 也会被 dbms 转为基本操作 $\sigma$，join on 的目的主要是为了提供 outer join

    * Inner Join:  即关系代数中的$\theta$-连接运算
    * Left Outer Join, Right  Outer Join, Full  Outer  Join: 即关系代数中的外连接运算

    ![在这里插入图片描述](https://img-blog.csdnimg.cn/20201126235833990.png#pic_center)



* 重命名

  * 连接运算涉及到重名的问题，如两个表中的属性重名，连接的两个表重名(同一表的连接)等，因此需要使用别名以便区分

  * select中采用别名的方式

    ```sql
    Select 列名 as 列别名[ [, 列名 as 列别名] ... ]
    From   表名1  as  表别名1, 表名2  as  表别名2,   ...
    Where   检索条件;
    ```

    * 上述定义中的 as 可以省略
    * 当定义了别名后，在检索条件中可以使用别名来限定属性





* 结果排序

  * DBMS可以对检索结果进行排序，可以升序排列，也可以降序排列。

  * Select语句中结果排序是通过增加order by子句实现的

    ```sql
    order by 列名 [asc |desc]
    ```

    * 意义为检索结果按指定列名进行排序，若后跟asc或省略，则为升序；若后跟desc, 则为降序。



* 空值处理

  * 除is[not]null之外，空值不满足任何查找条件
  * 如果null参与算术运算，则该算术表达式的值为null
  * 如果null参与比较运算，则结果可视为 false。在SQL-92中可看成 unknown
  * 如果null参与聚集运算，则除count(*)之外其它聚集函数都忽略null

  * 如果要查询null，使用 IS [not] NULL



追加元组

* insert into简单语法形式：

  ```sql
  insert into 表名[ (列名[, 列名]... ] values (值[, 值] , ...) ;
  ```

  * values后面值的排列，须与into子句后面的列名排列一致
  * 若表名后的所有列名省略，则values后的值的排列，须与该表存储中的列名排列一致

* 元组新增Insert命令有两种形式

  * 单一元组新增命令形式：插入一条指定元组值的元组插入一条指定元组值的元组

    ```sql
    insert  into 表名[(列名[，列名]...)] values (值[，值]...)；
    ```


  * 批数据新增命令形式：插入子查询结果中的若干条元组。待插入的元组由子查询给出。

    ```sql
    insert  into 表名[(列名[，列名]...)] 子查询;
    ```

    * 子查询可以返回一行或者多行，但是每一列的域必须相同





元组删除

* 元组删除Delete命令: 删除满足指定条件的元组删除满足指定条件的元组

  ```sql
  Delete   From 表名 [ Where 条件表达式] ;
  ```

  * 如果Where条件省略，则删除所有的元组。



元组更新

* 元组更新Update命令: 用指定要求的值更新指定表中满足指定条件的用指定要求的值

  ```sql
  Update 表名 Set 列名 = 表达式|(子查询)  [ [ ,  列名= 表达式|  (子查询) ] ... ][ Where 条件表达式] ;
  ```

  * 如果Where条件省略，则更新所有的元组。

  * 子查询只能返回一行，并且域值必须符合要求
