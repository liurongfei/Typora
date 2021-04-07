

## 一.运行时数据区域



 虚拟机运行时的数据区

![image-20210407115126325](C:\Environment\Github\Typora\Java虚拟机\image-20210407115126325.png)



### 1.程序计数器

多线程是通过线程轮回切换、分配cpu执行时间来实现的，为了线程切换后能够恢复到正确的执行位置，每一个线程都需要一个程序计数器，各线程之间计数器互不影响，独立存储。

### 2.Java虚拟机栈

* 每一个Java方法执行时，jvm都会同步创建一个栈帧用于存储局部变量表、操作数栈、动态连接、方法出口等信息。

* 方法调用时，栈帧从Java虚拟机栈入栈，方法执行完毕，栈帧从Java虚拟机栈出栈。

* Java虚拟机栈也是线程私有的，它的生命周期与线程相同。



局部变量表：



### 3.本地方法栈

本地方法栈的作用与Java虚拟机栈的作用相似，区别是本地方法栈是为使用的native方法服务

### 4.Java堆

Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建，存放对象实例。

### 5.方法区

> 别名，非堆（Non-Heap），目的是与Java堆区分开来

方法区与Java堆一样，是各个线程共享的内存区域。用于存储已被虚拟机加载的类型信息，如类名、访问修饰符、常量池、字段描述、方法描述等。



### 6.运行时常量池

运行时常量池是方法区的一部分，用于存放编译期生成的各种字面量与符号引用。

### 7.直接内存

直接内存不是虚拟机运行时数据区的一部分。

NIO类可以使用Native函数库直接分配堆外内存。

## 二、对象内存布局

以HotSpot虚拟机为例，对象在堆内存的布局分为三部分：对象头、实例数据和对齐填充

* 对象头：包含两部分，Mark Word和Klass Point。
  * Mark Word存储对象自身运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有锁、偏向线程ID、偏向时间戳等。
  * 类型指针指向对象的类型元数据，用于jvm确认对象是哪个类的实例
* 实例数据：对象存储的有效信息，即在程序代码中定义的各种类型的字段内容
* 对齐填充：没有特别的含义，用于将对象的大小保持为8字节的整数倍

![image-20210407163119673](C:\Environment\Github\Typora\Java虚拟机\image-20210407163119673.png)

### 对象头Mark Word



![img](C:\Environment\Github\Typora\Java虚拟机\objhead64.png)

![img](C:\Environment\Github\Typora\Java虚拟机\objhead32.png)



## 三、内存溢出

### Java堆溢出

堆是存放对象实例的，只要不断创建对象，同时避免垃圾回收机制清除这些对象，就会发生内存溢出异常。

> 使用VM参数设置堆的最小值-Xms参数和堆最大值-Xmx参数，将最大最小设置为一样即可避免堆自动扩展

```java
public class OOM {
    public static void main(String[] args) {
        List<OOM> list = new ArrayList<>();
            while(true){
                OOM oom = new OOM();
                list.add(oom);
            }
    }
}
```

### 虚拟机栈和本地方法栈溢出

>  HotSpot虚拟机不区分虚拟机栈和本地方法栈，使用-Xoss设置本地方法栈大小没有任何效果。
>
> 栈容量只能通过-Xss参数设置

栈溢出有两种异常：

* 如果线程请求的栈深度大于虚拟机运行的最大深度，将抛出StackOverflowError异常，比如递归
* 如果虚拟机的栈内存允许动态扩展，当扩展栈容量无法申请到足够的内存时，将抛出OutOfMemoryError异常

```java
public class StackOverflowDemo {
    private int length;

    public void StackOverflow(){
        length++;
        StackOverflow();
    }
    public static void main(String[] args) {
        StackOverflowDemo stackOverflowDemo = new StackOverflowDemo();
        try{
            stackOverflowDemo.StackOverflow();
        }catch(Exception e){
            e.printStackTrace();
        }finally {
            System.out.println("栈深度："+stackOverflowDemo.length);
        }
    }
}
```

### 方法区和运行时常量池溢出

方法区存放类加载信息和常量池，可以通过String.intern()方法实现常量池溢出。

>  String.intern()方法是一个本地方法，如果字符串常量池已经包含等于此String对象的字符串，则返回池中该字符串的引用，否则添加此String对象到常量池，并返回此对象的引用。