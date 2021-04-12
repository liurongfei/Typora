## ConcurrentHashMap



## Fork/Join 框架

Fork/Join框架是Java 7提供的一个用于并发执行任务的框架。

Fork/Joing框架可以把一个大任务分割成若干个小任务，最终汇总每一个小人物结果得到大人物结果



### 1.工作窃取算法

工作窃取算法是指某个线程从其他队列里窃取任务来进行。

将任务分割成若干个子任务，将这些子任务分别放到不同的队列里，并为每个队列创建一个单独的线程来执行队列里的任务。

这个线程每次都从队首获取任务执行。

如果某个子任务队列，其线程执行完了这个队列里的所有任务，它就会去其他线程的队列里窃取一个任务来执行，从这个队列的尾部获取任务执行。

**优点**：充分利用线程进行并行计算，减少了线程间的计算。

**缺点**：在某些情况下还是存在竞争，比如双端队列种只有一个任务时，该算法会消耗更多的系统资源。



### 2.Fork/Join框架的设计

1. 分割任务。
2. 执行任务并合并结果

Fork/Join使用两个类来完成以上两件事。

1. ForkJoinTask:Fork/Join任务提供了在任务种执行fork()和join()操作的机制，通常情况下只需要继承它的子类来创建响应的Fork/Join任务。
   * RecursiveAction:用于没有返回结果的任务
   * RecursiveTask:用于有返回结果的任务
2. ForkJoinPool：ForkJoinTask需要通过ForkJoinPool来执行。

任务分割出的子任务会添加到当前工作线程锁维护的双端队列中，进入队列的头部，当一个线程的队列里没有任务时，将会从其他线程的队列的尾部获取一个任务



### 3.使用Fork/Join框架

**案例：使用Fork/Join框架计算1+2+3+4**

```java
public class Demo {
    public static void main(String[] args) {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        AddCountTask task = new AddCountTask(1,4);
        ForkJoinTask<Integer> submit = forkJoinPool.submit(task);
        try {
            System.out.println(submit.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
class AddCountTask extends RecursiveTask<Integer>{
    private static final int THRESHOLD = 2;//最小任务数
    private int start;//任务开始
    private int end;//任务结束

    public AddCountTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum = 0;
        //判断当前任务是否小于最小阈值
        boolean canCompute = (end-start) <= THRESHOLD;
        //如果任务已经最小了，开始计算任务
        if(canCompute){
            for (int i = start; i <=end ; i++) {
                sum += i ;
            }
        }else {//如果任务大于阈值，分裂成两个任务计算
            int middle = (start+end)/2;
            //分成两个子任务
            AddCountTask left = new AddCountTask(start,middle);
            AddCountTask right = new AddCountTask(middle + 1, end);
            //执行任务
            left.fork();
            right.fork();
            //合并结果
            sum = left.join()+right.join();
        }
        return sum;
    }
}
```

### Fork/Join框架的异常处理



### 5.Fork/Join框架的实现原理

ForkJoinPool由ForkJoinTask数组和ForkJoinWorkerThread数组组成。

ForkJoinTask数组负责将存放程序提交给ForkJoinPool任务，而ForkJoinWorkerThread负责执行这些任务

