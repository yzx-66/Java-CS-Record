### String

#### 底层

**内部结构**

```
public final class String implements Serializable,Comparable<String>,CharSequence {
       // 用于存储字符串的值
       private final char value[]
       // 缓字符串的hash code
       private int hash;
}
```

- 值类使用final修适的原因
  - 安全，不会再有自类进行修改，从而导致出现安全问题
  - 高效，线程安全（因为值不变所以，所以多线程访问时不用加锁）、可以缓存（各种包装类型缓存池）

**构造方法**

public String ( String original )

public String ( char value[] )

public String ( StringBuilder builder )

public String ( StringBuffer buffer)  // 深拷贝时要加synchronized锁



**开辟位置**
String s = "abc"  ---> 常量池（字节码指令 ldc）

String s = new String("abc")  ---> 堆

String s = string.internal() ---> 常量池（入池，如果常量池已经存在则直接返回常量池的字符串）

String.valueOf() ---> obj != null ? obj.toString : "null"

#### 字符串拼接

**StringBuffer**

```
public final class StringBuffer extends AbstractStringBuilder implements Serializable, CharSequence{   
	
	// 字符串缓冲，不可序列化
	// 对外提供了改变toStringCache的方法
	transient char[] toStringCache;
	
	// 加锁，并发安全，性能低
	public synchronized StringBuffer append(String str){
         toStringCache = null;
         super.append(String.valueOf(str));
         return this;
    }

}
```

**StringBuilder**

```
// 可序列化
char[] value;

// 不加锁，就只是相当与一个可变string
public StringBuilder append(String str) {
        super.append(str);
        return this;
 }
```



**String + String 原理**

```
String a = "12345";
a=a+1;
a=a+1;
```

```
0: ldc           #2                  // String 12345
2: astore_1
3: new           #3                  // class java/lang/StringBuilder
6: dup
7: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
10: aload_1
11: invokevirtual #5                  // Method java/lang/StringBuilder.append:													(Ljava/lang/String;)Ljava/lang/StringBuilder;
14: iconst_1
15: invokevirtual #6                  // Method java/lang/StringBuilder.append:													(I)Ljava/lang/StringBuilder;
18: invokevirtual #7                  // Method java/lang/StringBuilder.toString:												()Ljava/lang/String;
21: astore_1
22: new           #3                  // class java/lang/StringBuilder
25: dup
26: invokespecial #4                  // Method java/lang/StringBuilder."<init>":()V
29: aload_1
30: invokevirtual #5                  // Method java/lang/StringBuilder.append:													(Ljava/lang/String;)Ljava/lang/StringBuilder;
33: iconst_1
34: invokevirtual #6                  // Method java/lang/StringBuilder.append:													(I)Ljava/lang/StringBuilder;
37: invokevirtual #7                  // Method java/lang/StringBuilder.toString:												()Ljava/lang/String;
40: astore_1
41: return
```

即：

- 每一个拼接语句都会通过StringBuilder的空构造方法new一个实例，然后调用其append方法，最后再调用toString
- 两次拼接都是：new StringBuilder().append(a).append(1).toString();



**常量拼接时编译优化**

```
String s1 = "ja" + "va";
String s2 = "java";
System.out.println(s1==s2); //true
```



**变量拼接时final修适的String视为常量**

```
String s = "java";
final String src1 = "ja"; // 相当于常量"ja"
String dist1 = src1 + "va"; // 相当于常量拼接
String src2 = "ja";
String dist2 = src2 + "va"; // 变量拼接需入池

System.out.println((s == dist1)); // true
System.out.println((s == dist2)); // false
System.out.println(s == dist2.intern()); // true
```

#### 补充
**String进行大规模数据操作的时候，会有什么问题？**

字符串直接拼接：会产生大量StringBuilder对象

spilt、substring等：因为String不可变，每次都会调用new String，所以可能会产生过多String，可以自己构建缓存（用hashmap），不建议直接每次都用.internal() 去查看常量池有没有，因为方法区是用来存放装载类和创建类实例时用到的元数据，可能会发生溢出。


**toString和String.valueOf的区别，toString当对象为null时会怎么样**

str.toString() 必须保证String不为null，不然会NPE

String.valueOf(str) str可以为null，返回值为"null"

**String a = new String("a")几个对象**

两个，一个在常量池、一个在堆上

 **string为何会产生很多对象**

每一条拼接语句都会产生一个StringBuilder对象

每一个字符串的真实值都会在常量池存在一份

**stringbuilder初始化数组大小**

默认16
