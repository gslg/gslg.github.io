---
title: Java类和接口的初始化
author: gslg
tags:
  - jls
categories: []
date: 2019-01-23 12:45:00
---
类的初始化由执行类中静态代码块的初始化和类中声明的静态域的初始化两部分组成  
接口的初始化由接口中声明的域（常量）的初始化组成

#### 初始化的时机
类或接口T在以下任意情况发生之前会立即进行初始化:
 - T是一个类并且创建了一个T的实列
 - T中声明的一个静态方法被调用
 - T中声明的一个静态域被分配(assigned)
 - T中的一个静态域被使用并且该域不是一个常量变量(constant variable)
 
 当一个类被初始化时，它的超类也会被初始化(如果之前未被初始化的话)，以及声明任何默认方法的父接口也会被初始化(如果之前未被初始化)；`接口自身的初始化并不会导致其父接口的初始化`  
 
 引用一个静态域(`static field`)只会导致实际声明该域的类进行初始化，尽管它可能通过子类名、子接口、或者实现接口的某个类名进行引用。
 
 调用`Class`类或包`java.lang.reflect`中的某些方法也会导致类或接口进行初始化`(Class.forName)`.
 <!--more-->
 
 对于上面4种情况,我们可以举例来验证一下.下面Hello这个类声明了一个常量A，常量变量B(具体什么是常量变量可以参考《java language specification》第4.12.4节)，一个静态变量c,以及两个静态代码块(23-30行)
 
 
 ```java 
public class Hello {
    /**常量*/
    public static final int A = initA();
    /**常量变量*/
    public static final int B = 10;
    /**静态域*/
    public static int c;
    /**静态代码块1*/
    static {
        System.out.println("Hello静态代码块加载了1.........");
    }
    /**静态代码块2*/
    static {
        System.out.println("Hello静态代码块加载了2.........");
    }
    public Hello() {
        System.out.println("Hello构造函数初始化了........");
    }
    public static void hello() {
        System.out.println("Hello.hello方法被调用了........");
    }
    public static int initA() {
        System.out.println("Hello.initA方法被调用了........");
        return 10;
    }
}
```
接着，我们写测试类来依次验证上述4种情况.  

###### 1.验证实例化
```java 
    /**测试Hello实例化*/
    @Test
    public void testInstance(){
        new Hello();
    }

```
可以在控制台看到如下输出: 
> Hello.initA方法被调用了........  
> Hello静态代码块加载了1.........  
> Hello静态代码块加载了2.........  
> Hello构造函数初始化了........  

从输出中我们可以看到Hello类确实在`实例化之前`被初始化了,同时因为Hello是一个类，它的初始化包括静态代码块的执行和静态域的初始化两部分，这点从输出中也可以得到印证。首先第一行输出证明初始化了静态域A(`接着是B,c这里没法从输出中直观的验证`)；接着第2,3行输出证明初始化执行了静态代码块1和2(`静态域或静态代码块的执行顺序和声明顺序有关`)；最后一行表明Hello的构造函数被调用。

###### 2.测试静态方法的调用

```java
    /** 测试调用Hello的静态方法*/
    @Test
    public void testStaticMethodInvoke(){
        Hello.hello();
    }
```
控制台输出:
> Hello.initA方法被调用了........  
> Hello静态代码块加载了1.........  
> Hello静态代码块加载了2.........  
> Hello.hello方法被调用了........    

这里我们调用了Hello类中的静态方法getA,同理可以看到类A首先进行了初始化。

###### 3.测试静态域被分配(即赋值)
```java
   /**测试静态域被赋值*/
    @Test
    public void testStaticFieldAssigned(){
        Hello.c = 20;
    }
```
控制台输出:
> Hello.initA方法被调用了........  
> Hello静态代码块加载了1.........  
> Hello静态代码块加载了2.........  

同理可以得出静态域被分配(赋值)也会触发初始化操作

###### 4.测试使用类的常量  
我们首先测试使用常量A
```java
    /**测试调用Hello的常量A*/
    @Test
    public void testConstantA(){
       int a =  Hello.A;
       System.out.println("a=" + a);
    }
```
控制台输出:
> Hello.initA方法被调用了........  
Hello静态代码块加载了1.........  
Hello静态代码块加载了2.........  
a=10  

可以看到调用常量A触发了类Hello的初始化动作.  
再来测试使用常量B:
```java
    /**测试调用Hello的常量B*/
    @Test
    public void testConstantVariableB(){
       int b =  Hello.B;
        System.out.println("b=" + b);
    }
```
控制台输出:
> b=10  

会奇怪的发现控制台只输出了b=10,而没有其他信息。`也就是说使用Hello.B并没有触发Hello类的初始化动作;`  
这是为什么呢?  
我们再次看第4种情况`T中的一个静态域被使用并且该域不是一个常量变量(constant variable)`,T中一个静态域被使用也会触发类的初始化操作，但是有一个前提是该静态域必须不是一个常量变量.对于常量变量的定义可以详见java语言规范中第4.12.4节的定义，这里先简单说明一下，对于类中的用`final static`声明的常量字段，如果直接使用基本类型或string类型的字面量表达式直接赋了值，那么就可以认为该字段是一个常量变量.因此Hello类中的静态字段B是一个常量变量，使用它不会触发Hello类的初始化操作.

###### 5.最后测试一下如果几种情况同时出现
```java
    /** 测试所有情况*/
    @Test
    public void testAll(){
        new Hello();
        Hello.hello();
        System.out.println("Hello.A=" + Hello.A);
        System.out.println("Hello.B=" + Hello.B);
    }
```
控制台输出:
> Hello.initA方法被调用了........  
Hello静态代码块加载了1.........  
Hello静态代码块加载了2.........  
Hello构造函数初始化了........  
Hello.hello方法被调用了........  
Hello.A=10  
Hello.B=10  

可以看到,`类Hello只会被初始化一次`。
















