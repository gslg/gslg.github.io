---
title: java.util.concurrent.Executor
author: gslg
tags:
  - java
  - concurrency
categories:
  - java并发
date: 2019-03-15 10:53:00
---
## java并发包Executor学习


#### 源码
```java
/* .....
 * @since 1.5
 * @author Doug Lea
 */
public interface Executor {

    /**
     * Executes the given command at some time in the future.  The command
     * may execute in a new thread, in a pooled thread, or in the calling
     * thread, at the discretion of the {@code Executor} implementation.
     *
     * @param command the runnable task
     * @throws RejectedExecutionException if this task cannot be
     * accepted for execution
     * @throws NullPointerException if command is null
     */
    void execute(Runnable command);
}
```
`Executor`是一个接口，表示一系列可以用来执行已提交任务(`Runnable Tasks`)的对象.  
它的源码很简单，就只有一个`void execute(Runnable command);`接口方法.  
`Executor`接口提供了一种将任务提交(`submit`)与任务如何运行的机制解耦的方式，包括线程使用，调度等细节。  
`Executor`通常用来代替传统的显示的创建线程，例如，之前我们需要在一个新线程中执行任务,我们一般这样写:
```java
        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("hello");
            }
        }).start();
```
<!--more-->
简单来说就是`new Thread(new RunnableTask()).start()`这种模板方式。  
使用`Executor`来创建任务，我们可以用下面的方式:
```java
Executor executor = anExecutor //一个Executor的具体实现
executor.execute(new RunnableTask1());
executor.execute(new RunnableTask2());
```
可以看到，使用`Executor`屏蔽掉了显示的创建线程`new Thread(...)`  
但是，`Executor`接口并没有严格要求执行必须是异步的，也就是说任务的执行可以由当前调用线程执行或者新创建线程执行。  
例如，我们如果要在当前线程执行任务，我们可以定义以下`Executor`实现:
```java
class DirectExecutor implements Executor{
  public void execute(Runnable r){
     r.run();
  }
}
```
这样，我们就可以直接在当前线程执行任务:
```java
Executor executor = new DirectExecutor();
executor.execute(()->System.out.println("当前线程执行"));
```
但是，更一般的情况是我们会在一个新线程中去执行任务，例如:
```java
class ThreadPerTaskExecutor implements Executor{
   public void execute(Runnable r){
     new Thread(r).start();
   }
}
```
这样，我们就为每一个任务都新建线程去执行。  
许多`Executor`实现对任务如何以及何时调度施加了一些限制。例如在下面的`SerialExecutor`中，我们把队列中的任务按序丢给另外一个executor去执行，相当于是一个组合的执行器.
```java
public class SerialExecutor implements Executor {

    private final Queue<Runnable> tasks = new ArrayDeque<>();

    private final Executor executor;

    Runnable active;

    public SerialExecutor(Executor executor) {
        this.executor = executor;
    }

    @Override
    public synchronized void execute(final Runnable r) {
        //向任务队列中添加任务
        tasks.offer(new Runnable() {
            @Override
            public void run() {
                try {
                    r.run();
                } finally {
                    scheduleNext();
                }
            }
        });
        //如果当前没有正在执行的任务，那么调度执行下一个任务
        if (active == null) {
            scheduleNext();
        }
    }

    /**
     * 调度执行下一个任务
     */
    protected synchronized void scheduleNext() {
        if ((active = tasks.poll()) != null) {
            executor.execute(active);
        }
    }
}
```

我们可以简单测试一下:
```java
public class SerialExecutorTest {
    public static void main(String[] args) {

        Executor serialExecutor = new SerialExecutor(new DirectExecutor());
        //我们添加10个任务来执行
        for (int i = 0; i < 10; i++) {
            serialExecutor.execute(new TaskRunnable(i, i + 1));
        }

    }

    public static class TaskRunnable implements Runnable {

        //任务ID
        private int taskId;
        //模拟的任务执行时间秒数
        private int seconds;

        public TaskRunnable(int taskId, int seconds) {
            this.taskId = taskId;
            this.seconds = seconds;
        }

        @Override
        public void run() {
            try {
                //模拟任务执行时间
                TimeUnit.SECONDS.sleep(seconds);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("任务[" + taskId + "]执行完毕,耗时:" + seconds + "秒");
        }
    }
}
```
可以在控制台看到，任务按预期的调度执行了:
>任务[0]执行完毕,耗时:1秒  
>任务[1]执行完毕,耗时:2秒  
>任务[2]执行完毕,耗时:3秒  
>任务[3]执行完毕,耗时:4秒  
>任务[4]执行完毕,耗时:5秒  
任务[5]执行完毕,耗时:6秒  
任务[6]执行完毕,耗时:7秒  
任务[7]执行完毕,耗时:8秒  
任务[8]执行完毕,耗时:9秒  
任务[9]执行完毕,耗时:10秒  

#### 内存一致性影响
在javadoc中关于`Executor`的内存一致性说明:
{% blockquote https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/Executor.html %}
Memory consistency effects: Actions in a thread prior to submitting a Runnable object to an Executor happen-before its execution begins, perhaps in another thread.
{% endblockquote %}

理解这句话有点难，举个例子来说:
```java
public class MemoryTest {

    static class Person {

        private int age;

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }
    }

    public static void main(String[] args) {
        final Person p = new Person();
        Executor executor = Executors.newFixedThreadPool(10);

        Runnable task2 = new Runnable() {
            @Override
            public void run() {
                System.out.println("任务2执行结果:" + p.getAge());
            }
        };

        Runnable task1 = new Runnable() {
            @Override
            public void run() {
                p.setAge(20);
                //任务1中提交任务2
                System.out.println("任务1执行完成..开始执行任务2....");
                executor.execute(task2);
            }
        };

        executor.execute(task1);
    }
}
```
在上面这个例子中，task1提交了另一个任务task2，在task1提交task2之前执行的任何动作对task2的执行都满足于happen-before关系，因此在task2执行时person的age值是20.  
稍微改造下:  
```java
public class MemoryTest2 {

    static class Person {

        private int age;

        public int getAge() {
            return age;
        }

        public void setAge(int age) {
            this.age = age;
        }
    }

    public static void main(String[] args) {
        final Person p = new Person();
        Executor executor = Executors.newFixedThreadPool(10);

        Runnable task2 = new Runnable() {
            @Override
            public void run() {
                System.out.println("任务2执行结果:" + p.getAge());
            }
        };

        Runnable task1 = new Runnable() {
            @Override
            public void run() {
                //任务1中提交任务2
                try {
                //模拟执行时间，方便观察
                    TimeUnit.SECONDS.sleep(3);
                    p.setAge(20);
                    System.out.println("任务1执行完成.....");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };

        executor.execute(task1); //①
        executor.execute(task2); //②
    }
}
```
在这个例子中，task1和task2是独立提交执行的(①-②)，因此不存在happen-before关系，因此在task2中获取age可能是0或者20，这取决于task1中setAge和task2中getAge谁先执行.

#### 总结
在`java.util.concurrent`包中，提供了`Executor`的实现`ExecutorService`,这是一个更广泛的接口。`ThreadPoolExecutor`提供了一个可扩展的线程池实现。`Executors`为这些类提供了方便的工厂方法。





