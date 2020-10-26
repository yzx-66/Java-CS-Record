### Object类

#### 基本方法

1、clone方法

保护方法，实现对象的浅复制

- 即把每个属性的原封不动复制过来，官方文档说"字段的内容本身不会被克隆", 即如果某个属性是对象，只会复制其引用，而不会复制其值
- 还说"如果一个类仅包含基本字段或对不可变对象的引用，则通常情况是不需要`super.clone` 修改对象返回的字段"，所以要实现深拷贝就要重写clone）

只有实现了Cloneable接口才可以调用该方法，否则抛出CloneNotSupportedException异常。

2、getClass方法

final方法，获得运行时类型。

3、toString方法

该方法用得比较多，一般子类都有覆盖。

4、finalize方法

该方法用于释放资源。因为无法确定该方法什么时候被调用，很少使用。

5、equals方法

该方法用于判断两个对象是否相等。一般equals和==是不一样的，但是在Object中两者是一样的。子类一般都要重写这个方法。

6、hashCode方法

该方法用于哈希查找，**这个方法在一些要要用到哈希码的集合（hash*）中用到。**

- **重写了equals方法一般都要重写hashCode方法**。

  一般必须满足```obj1.equals(obj2) == true```。可以推出```obj1.hashCode() == obj2.hashCode()```

  但是hashCode相等不一定就满足equals。不过为了提高效率，应该尽量使上面两个条件接近等价。

7、wait方法（官方文档）

wait方法必须在synchronized(obj) 同步代码块里调用（即调用wait方法的线程必须获得到锁（obj的moniter））

wait方法会释放掉锁一直阻塞，等待obj.notify唤醒，然后再去争取锁，获取成功后再去执行wait后面的代码 

wait(long timeout)设定一个超时间隔，如果超时没有notify，也会再去获取锁，

唤醒的四种情况

- 其他线程调用了该对象的notify方法。
- 其他线程调用了该对象的notifyAll方法。
- 其他线程调用了interrupt中断该线程，在重新获取到锁后，执行的是interrupt异常捕获的代码。
- timeout到了。

注意虚假唤醒：

线程也可以唤醒，而不会被通知，中断或超时，即所谓的*虚假唤醒*。尽管在实践中这种情况很少发生，但是应用程序必须通过测试应该导致线程唤醒的条件来防范它，并在条件不满足时继续等待。换句话说，**等待应该总是在循环中发生**，就像这样：

```
synchronized(obj){
      while(<条件不成立>)
      	  obj.wait（timeout）;
      
      //执行适合条件的动作
 }
```

8、notify方法

该方法唤醒在该对象上等待的某个线程。

9、notifyAll方法

该方法唤醒在该对象上等待的所有线程。

#### 补充
**创建对象的四种方式**

1、用new语句创建对象

2、运用反射手段,调用java.lang.Class 或者 java.lang.reflect.Constructor 类的newInstance()实例方法

3、调用对象的clone()方法

4、运用序列化手段,调用java.io.ObjectInputStream 对象的 readObject()方法.



**Java对象排序有哪几种方式**

1、使用Arrays.sort(Object[])实现数组排序（对象必须实现Comparable接口并复写compareTo()方法）

2、使用Collections.sort(List, Comparator)实现List排序（比较器必须实现Comparator<>接口，并实现compare()方法）

3、使用Collections.sort(List)实现List排序（对象必须实现Comparable接口并复写compareTo()方法）



**为什么重写equals还要重写hashcode**

如果同一个对象equals相同，但是hashcode不同时，在使用hash相关集合时无法表示同一个元素。

但是允许hashcode相同，equals不同



**instance of原理**

```
// obj instanceof T
boolean result;
if (obj == null) {
  result = false;
} else {
  try {
      T temp = (T) obj; // checkcast
      result = true;
  } catch (ClassCastException e) {
      result = false;
  }
}

```

如果有表达式 obj instanceof T ，那么如果 obj 不为 null 并且 (T) obj 不抛 ClassCastException 异常则该表达式值为 true ，否则值为 false 。



**instance of和getClass**

instanceof进行类型检查规则是:你属于该类吗？或者你属于该类的派生类吗？

getClass获得类型信息采用==来进行检查是否相等的操作是严格的判断。不会存在继承方面的考虑
