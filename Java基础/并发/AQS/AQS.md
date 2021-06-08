# AbstractQueuedSynchronizer

AbstractQueuedSynchronizer：抽象同步队列

JUC很多类的底层都是基于AQS实现的

* `ReentrantLock`的内部类Sync就是继承了AQS
* `ReentrantReadWriteLock`的内部类Sync继承了AQS
* `CountDownLatch`的内部类Sync继承了AQS
* `Semaphore`的内部类Sync继承了AQS
* `ThreadPoolExecutor`的内部类Worker继承了AQS

> `CyclicBarrier`底层是ReentrantLock和Condition实现的



## state

AQS底层维护了一个volatile int变量state，用于表示共享资源

```java
private volatile int state;
```



1. 在`ReentrantLock`中

state表示锁的状态

* **加锁时使state加一**
* **释放锁时使state减一**，

如果是同一个锁多次加锁，那么state的值会继续增加，

相应地就需要释放多次锁才能使state变成0(即没有锁没有被线程占有)

2. 在`ReentrantReadWriteLock`中

与ReenrantLock中的state类似,区别是读写锁中的state分为两部分

* **高16位**表示读锁的状态,计算时state无符号右移16位获取
* **低16位**表示写锁的状态,计算使将state与0x0000FFFF相与获取
* 当读状态增加时,state+(1<<16)
* 当写状态增加时,state+1

3. 在`CountDownLatch`中

state表示计数,由构造函数传入,相当于初始设置占有锁的数量

* 每次调用countDown()方法时计数一次,实质是调用释放锁的方法tryRelease(),使state减一
* 调用await()方法会调用tryAcquired()方法获取锁,实质是判断state是否为0

4. 在`Semaphore`中

state表示`当前可用信号量许可证数量`,由构造器传入初始值

* 当调用acquire()时从信号量获取一个许可证,使**state减一** 
* 当state为0时表示信号量可用许可证为0,不能够获取了,调用acquire()将会阻塞
* 调用release()将释放一个许可证,使**state加一**

5. 在`ThreadPoolExecutor`中

state表示,构造器初始固定设置为-1

* 当lock()时,将state从0置为1,表示locked状态
* 当释放锁时将state置为0,表示unlocked状态

## Node

AQS有一个内部类Node，用于维护一个先进先出的线程等待队列

AQS包含两个私有变量head和tail

```java
private transient volatile Node head;//等待队列的头部
private transient volatile Node tail;//等待队列的尾部
```

除了初始化，只能通过setHead(Node node)方法来设置头部

当新的线程需要等待时，通过AQS的方法enq()来添加到队列尾部



### 构造函数

Node内部类有两个构造方法，一个用于addWaiter()添加同步队列，一个用于Condition添加条件队列

```java
Node(Thread thread, Node mode) {     // Used by addWaiter
    this.nextWaiter = mode;
    this.thread = thread;
}
Node(Thread thread, int waitStatus) { // Used by Condition
    this.waitStatus = waitStatus;
    this.thread = thread;
}
```

### 成员变量

1. volatile Node next

用于同步队列,表示当前节点的下一节点

2. volatile Node prev

用于同步队列,表示当前节点的上一节点

3. Node nextWaiter

用于条件队列，表示下一 节点,

4. volatile int waitStatus

等待状态，有4个值

```java
static final int CANCELLED =  1;//线程已取消
static final int SIGNAL    = -1;//后继线程需要释放
static final int CONDITION = -2;//表示当前线程在条件队列中
static final int PROPAGATE = -3;//表示下一个acquireShared应该无条件传播
```

## CondtionObject

AQS的内部类,继承了Condition接口,用于条件队列

在ReentrantLock中,通过newCondition()方法获取一个Condition



### 方法

1. doSignalAll(Node first)

唤醒条件队列中所有等待的线程

2. await()

使当前线程进入等待队列,由其他线程调用signal()唤醒