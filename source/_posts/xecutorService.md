title: ExecutorService
author: gslg
date: 2019-03-15 17:43:09
tags:
---
## java并发ExecutorService
#### 源码
```java
public interface ExecutorService extends Executor {

    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();

    boolean isTerminated();

    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

   
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```
`ExecutorService`用于管理中止任务或者跟踪异步任务处理。  
要使用`ExecutorService`,首先需要定义一个任务:  
```java
public class Task implements Runnable {
    @Override
    public void run() {
        System.out.println("task begin........");
    }
}
```
,接着我们可以使用工厂类`Executors`实例化一个`ExecutorService`:
```java
 ExecutorService executorService = Executors.newFixedThreadPool(4);
 executorService.submit(new Task());
```
，`Executors`提供了返回各种`ExecutorService`的工厂方法，例如如果我们需要单线程执行我们的任务，可以这样获取:
```java
ExecutorService executorService = Executors.newSingleThreadExecutor();
```
,最后我们关闭`ExecutorService`：
```java
executorService.shutdown();
```
正如上面代码所示，`ExecutorService`可以被关闭，一旦关闭后他就拒绝接收新任务了。它提供了两种关闭的方法:  

- `void shutdown()`允许已经提交的任务执行完成后再终结
- `List<Runnable> shutdownNow()`会立即中止等待开始的任务和正在执行中的任务

一旦终止后，Executor就不会有正在执行的任务，也不会有等待执行的任务，同时也不能提交新任务。不再使用的`ExecutorService`应当被关闭，以便释放它的资源。  
在上面的代码中，我们通过`executorService.submit(new Task())`来提交任务，方法`submit`扩展了基本方法`Executor.execute(Runnable)`,它创建并返回一个可以用来取消或者等待完成的`Future`接口.

在java doc文档中提供了一个简单的网络服务例子，它会等待每个请求的到来并从线程池中获取一个线程来服务请求:
```java
class NetworkService implements Runnable {
   private final ServerSocket serverSocket;
   private final ExecutorService pool;

   public NetworkService(int port, int poolSize)
       throws IOException {
     serverSocket = new ServerSocket(port); 
     pool = Executors.newFixedThreadPool(poolSize);
   }

   public void run() { // run the service
     try {
       for (;;) {
         pool.execute(new Handler(serverSocket.accept())); //这里会阻塞直到获取一个socket请求
       }
     } catch (IOException ex) {
       pool.shutdown(); //关闭
     }
   }
 }

 class Handler implements Runnable {
   private final Socket socket;
   Handler(Socket socket) { this.socket = socket; }
   public void run() {
     // read and service request on socket  //处理请求
   }
 }
```
下面是分两阶段关闭一个`ExecutorService`,首先调用`shutdown`来拒绝接收到来的新任务，接着在必要时刻调用`shutdownNow`来取消任何遗留的任务:
```java
 void shutdownAndAwaitTermination(ExecutorService pool) {
   pool.shutdown(); // 拒绝接收新提交的任务
   try {
     // 等待直到所有正在执行的任务中止
     if (!pool.awaitTermination(60, TimeUnit.SECONDS)) {
       pool.shutdownNow(); // 取消当前正在执行的任务
       // 等待直到任务响应告知已经被取消
       if (!pool.awaitTermination(60, TimeUnit.SECONDS))
           System.err.println("Pool did not terminate");
     }
   } catch (InterruptedException ie) {
     // 如果当前线程本身被中断
     pool.shutdownNow();
     // 保持当前线程的中断状态
     Thread.currentThread().interrupt();
   }
 }
```

方法`boolean awaitTermination(long timeout, TimeUnit unit)`会阻塞等待直到所有任务完成或者超过预设的等待时间。  
`invokeAny`和 `invokeAll`是最常用的执行一组批量任务的形式，并且它会等待至少一个或者所有任务完成:

- `invokeAny`执行给定的任务集，只要有一个成功完成，就返回。一旦一个正常返回或者抛出异常，所有未完成的任务都会被取消。
- `invokeAll`会等待所有任务执行完成后返回.