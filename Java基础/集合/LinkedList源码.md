# 一、源码

> LinkedList实现了List接口和Deque接口，底层是维护双向链表

## 成员变量

```java
transient int size = 0;//大小
transient Node<E> first;//头节点
transient Node<E> last;//尾节点
```



## 添加元素

```java
//调用linkLast()方法，将元素插入到链表末尾
public boolean add(E e) {
    linkLast(e);
    return true;
}
```



```java
//采用尾插法插入
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

