**概念：**

Java一直是面向对象的语言，一切皆对象，如果想要调用一个函数，函数必须属于一个类或对象，然后在使用类或对象进行调用。但是在其它的编程语言中，如js，c++，我们可以直接写一个函数，然后在需要的时候进行调用，即可以说是面向对象编程，也可以说是函数式编程。

从功能上来看，面向对象编程没什么不好的地方，但是从开发的角度来看，面向对象编程会多写很多可能是重复的代码行。比如创建一个Runnable的匿名类的时候：

```
Runnable runnable = new Runnable() {
    @Override
    public void run() {
        System.out.println("do something...");
    }
};
```

这一段代码中真正有用的只有run方法中的内容，剩余的部分都是属于Java编程语言的结构部分，没什么用，但是要写。

Java8开始，引入了函数式编程接口与Lambda表达式，帮助我们`写更少更优雅的代码`

```
// 一行即可
Runnable runnable = () -> System.out.println("do something...");
```



**函数式接口**

概念：

- 一个接口类`只有一个`抽象的方法叫做函数式接口(Functional Interface)
- 在JDK中大部分函数式接口都会标记上`@FunctionalInterface`注解，并不是所有的函数式接口都要写`@FunctionalInterface`注解，只是用来方便我们区分哪些是函数式接口的，当然如果标记了这个注解，内部有多个抽象方法的时候，会报编译错误。



内置函数式接口：

JDK中已经内置了一些标准的函数式接口，位于`java.util.function`包下

- 最常见的四种，Function，Consumer，Supplier，Predicate。标准输入输出，优雅代码所推荐的写法 
  - Function：即一个入参一个出参的场景。
  - Consumer：一个入参，但是没有出参
  - Supplier：无入参，一个出参
  - Predicate：可以看做是特殊的Function，一个入参，出参为bool类型。
- 两个入参的函数式接口
  - BiFunction<T, U, R>
  - BinaryOperator\<T\> extends BiFunction<T,T,T>
  - BiConsumer<T, U>
  - BiPredicate<T, U>
- 一元函数式接口
  - UnaryOperator\<T\> extends Function<T, T>
- 原始类型Function，因为Java的原始类型或者叫做主类型，int，short，double之类的，不能作为泛型参数。
  - 以int类型：IntFunction, IntComsumer,IntSupplieer, IntPredicate, IntToDoubleFcuntion, IntToLongFunction, IntUnaryOperator, ToIntFunction



方法引用：

函数式接口可以直接传递方法引用

- 对象名 ::引用成员方法
- 类名 ::引用静态方法
- 类名 ::引用实例方法
- 类名 ::new引用构造器




##### Lamda

作用：

- 使用Lambda表达式简化匿名内部类的书写，但Lambda表达式并不能取代所有的匿名内部类，只能用来取代函数接口（Functional Interface）的一种简写。



原理：

- Lambda表达式通过invokedynamic指令实现，执行的方法不再和类型绑定，因此不知道方法实现，所以把执行何种 invoke 的权利交给了用户定义的引导方法。（详解可以参考的我的 JVM 系列文章）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201026142335684.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70#pic_center)


  

- 强调一下，上面说了 lamda 是在当前 class 中生成一个静态方法，所以如果调用外部方法，那么也都是当前 class 的，并且 this 也不是说自己有一个 this 引用，而是会把这个 this 的对象，作为编译后静态方法的参数传进来，就和匿名内部类调用外部参数是一个道理。

  ```java
  public class Hello {
  	Runnable r1 = () -> { System.out.println(this); }; // 输出 Hello，因为把 this 的对象作为静态方法的参数
  	Runnable r2 = () -> { System.out.println(toString()); }; // 输出 Hello，因为就是调用当前类的方法
  	public static void main(String[] args) {
  		new Hello().r1.run();
  		new Hello().r2.run();
  	}
  	public String toString() { return "Hello"; }
  }
  ```



