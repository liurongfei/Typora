# 一、源码

> ArrayList底层是维护一个Object数组，默认为空数组，每次add时，会先检查数组长度，如果当前数组为默认数组，即空数组，则调用Arrays.copyOf方法，扩容到默认容量10，否则扩容到响应的长度

## 成员变量

```java
Object[] elementData;//object 数组
int size;//arraylist长度
private static final int DEFAULT_CAPACITY = 10;//默认容量10
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};//默认数组，初始化为空数组
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;//arraylist的最大容量
```

## add流程

1.add(E e)

```java
public boolean add(E e){
        //该方法是检查数组长度并扩容
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        //插入元素
        elementData[size++] = e;
        return true;
}
```

2.ensureCapacityInternal(int minCapacity)

```java
private void ensureCapacityInternal(int minCapacity) {
    //计算容量并传入
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
```

3.calculateCapacity(Object[] elementData, int minCapacity)

```java
//计算数组容量
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    //如果数组未初始化，则返回10，否则返回提供的容量，即size+1
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}
```

4.ensureExplicitCapacity(int minCapacity)

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```



5.

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

 

```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```