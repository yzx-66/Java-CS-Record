### 嵌入式SQL

#### 概述

嵌入式SQL语言

* 将SQL语言嵌入到某一种高级语言中使用
* 这种高级语言，如C/C++, Java, PowerBuilder等，又称宿主语言(Host Language)
* 嵌入在宿主语言中的SQL与前面介绍的交互式SQL有一些不同的操作方式



下面以嵌入 C 语言为例，exec 关键字是为了让 C 语言编译器识别。



#### 变量声明

在嵌入式SQL语句中可以出现宿主语言语句所使用的变量：

```c
exec sql begin declare section;
	char vSname[10], specName[10]=“张三”;
	int vSage;
exec sql end declare section;

// 对 vSname 和 vSage 变量进行赋值（通过输入或直接赋值）

exec sql  select Sname, Sage  into:vSname, :vSage from Student  
    where Sname= :specName;
```

* 注意：
  * 宿主程序的字符串变量长度应比字符型字段的长度多1个。因宿主程序的字符串尾部多一个终止符为‘\0‘，而程序中用双引号来描述。
  * 宿主程序变量类型与数据库字段类型之间有些是有差异的,有些DBMS可支持自动转换，有些不能。



#### 数据集与游标

单行结果处理与多行结果处理的差异

* 检索单行结果可将结果直接传送到宿主程序的变量中（Into）

  ```
  EXEC  SQL  SELECT[ALL| DISTINCT] expression [, expression...]
  INTO host-variable , [host-variable, ...]
  FROM  tableref  [corr_name] [ , tableref  [corr_name] ...]
  WHERE  search_condition;
  ```

  示例

  ```
  exec sql  selectSname, Sage  into:vSname, :vSagefromStudent  where Sname = :specName ;
  ```

* 检索多行结果，则需使用游标（fetch-into)

  * 游标是指向某检索记录集的指针
  * 通过这个指针的移动，每次读一行，处理一行，再读一行... , 直至处理完毕

  


定义

* 声明游标

  ```
  declare cursor
  EXEC  SQL  DECLARE  cursor_name  CURSOR   FOR 
  	Subquery
  	[ORDER  BY  result_column  [ASC| DESC][, result_column ...]
  	[FOR  [ READ  ONLY |  UPDATE [OF  columnname [, columnname...]]]];
  ```

  示例

  ```
  exec sql  declare  cur_studentcursor  for 
  select Sno, Sname, Sclass from Student  where Sclass= :vClassorder by  Sno
  for read only ;
  ```

* 打开和关闭

  ```
  EXEC  SQL  OPEN  cursor_name;
  EXEC  SQL  CLOSE  cursor_name;
  ```

* 数据读取

  ```
  EXEC  SQL  FETCH  cursor_name INTO  host-variable , [host-variable, ...];
  ```

  

使用步骤

* 游标(Cursor)的使用需要先定义、再打开(执行)、接着一条接一条处理，最后再关闭

  * 读一行操作是通过Fetch...into语句实现的：每一次Fetch, 都是先向下移动指针，然后再读取

  * 记录集有结束标识EOF, 用来标记后面已没有记录了

    ```
    exec sql  declare  cur_student cursor  for 
    	select Sno, Sname, Sclass from Student  where Sclass=‘035101’;
    exec sql  open   cur_student;
    exec sql  fetch  cur_student into:vSno, :vSname, :vSclass;
    ......
    exec sql  close cur_student;
    ```




可滚动游标

* 定义

  ```
  EXEC  SQL  FETCH 
  [ NEXT| PRIOR | FIRST | LAST | [ABSOLUTE  | RELATIVE]  value_spec ]  
  FROM  cursor_name  INTO  host-variable [, host-variable ...];
  ```

  * NEXT向结束方向移动一条；
  * PRIOR向开始方向移动一条；
  * FIRST回到第一条；
  * LAST移动到最后一条；
  * ABSOLUTvalue_spec定向检索指定位置的行,value_spec由1至当前记录集最大值；
  * RELATIVEvalue_spec相对当前记录向前或向后移动，value_spec为正数向结束方向移动，为负数向开始方向移动



定位操作

* 定位删除

  ```
  exec sql  declare delcust cursor  for
  	select cid  from customers  c  wherec.city =‘harbin’and 
  	not exists ( select* fromorders  o  whereo.cid = c.cid)
  	for  update  of  cid;
  exec sql  open  delcust
  While (TRUE) {
  	exec  sql  fetch  delcustinto  :cust_id; 
  	exec  sql  delete  from  customers where  current  of  delcust; // 删除 delcust 游标的 current 位置元素
  }
  ```

* 定位更新

  ```
  exec  sql  update  studentset custername = ‘cname’ where  current  of  custcur ; // 更新 custcur 游标的 current 位置元素
  ```

  

#### 动态构造

指根据输入，动态拼接 SQL，这也是嵌入式 SQL 的核心优势。



#### 执行方式

* 动态SQL的两种执行方式



如SQL语句已经被构造在host-variable字符串变量中,则：

* 立即执行语句: 运行时编译并执行

  ```
  EXEC  SQL  EXECUTE  IMMEDIATE  :host-variable; // host-variable 指 sql 字符串
  ```

  

* Prepare-Execute-Using语句:

  * PREPARE语句先编译，编译后的SQL语句允许动态参数，
  * EXECUTE语句执行，用USING语句将动态参数值传送给编译好的SQL语句

  ```
  EXEC  SQL  PREPARE  sql_temp  FROM  :host-variable; // host-variable 指 sql 字符串
  ......
  EXEC  SQL  EXECUTE  sql_temp  USING  :cond-variable
  ```

* 示例

  ```c
  #include <stdio.h>
  #include "prompt.h"
  
  exec sql include sqlca;
  exec sql begin declare section;
  	char cust_id[5], sqltext[256], user_name[20], user_pwd[10];
  exec sql end declare section;
  
  char cid_prompt[ ] = "Name customer cid to be deleted: ";
  
  int main(){
  	strcpy(sqltext, "delete from customers where cid = :dcid");
  	… …
  	while (prompt(cid_prompt, 1, cust_id, 4) >= 0) {
          exec sql whenever not found goto no_such_cid;
          exec sql prepare delcust from :sqltext; /* prepare statement */
          exec sql execute delcust using :cust_id; /* using clause ... replaces ":n" above */
          exec sql commit work; continue;
          no_such_cid: printf("No cust %s in table\n",cust_id);
          continue;
  	}
  	…
  }
  ```






#### 数据字典

数据字典通常存储的是数据库和表的元数据，让高级语言或者客户端可以获取，即模式本身的信息：

* 与关系相关的信息

  * 关系名字
  * 每一个关系的属性名及其类型
  * 视图的名字及其定义
  * 完整性约束

* 用户与账户信息，包括密码

* 统计与描述性数据：如每个关系中元组的数目

* 物理文件组织信息：

  * 关系是如何存储的(顺序/无序/散列等)
  * 关系的物理位置

* 索引相关的信息

  

  

数据字典的结构

* 也是存储在磁盘上的关系
* 专为内存高效访问设计的特定的数据结构

