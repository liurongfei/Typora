# 一、源码

> ArrayDeque是实现了Deque接口的队列，底层维护一个Object数组，通过head和tail实现队列



## 成员变量

```java
transient Object[] elements;
transient int head;//队首下标
transient int tail;//队尾下标
private static final int MIN_INITIAL_CAPACITY = 8;//初始容量8
```

## 构造函数

```java
//初始容量16
public ArrayDeque() {
elements = new Object[16];
}
```

