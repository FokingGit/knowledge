## okhttp是如何进行异步请求的?

通过Okhttp中的Dispatch

- 维护了请求队列和线程池,实现高并发

```java
//okhttp的线程池,核心线程为0,最大线程数为Integer的最大值,当非核心线程线程数量大于0时,当线程执行完毕,空闲大于60秒的之后,回收该空闲线程
//对于同步队列SynchronousQueue,是没有缓存的,当有新添加任务的时候,就直接新开启一个线程
//注:1. 当任务<=核心线程 使用核心线程执行任务
//	2. 当任务>核心线程,队列未满,将任务存放至队列中
//	3. 当任务>核心线程,队列已满,开启新线程执行任务
//	4. 当线程数量>最大线程数量 根据Handler的不同,处理结果不同,默认情况下是
public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```



1. 创建OkHttpClient、Request、Call

2. 通过Call的enqueue方法,发起异步请求

   ```java
   @Override public void enqueue(Callback responseCallback) {
       synchronized (this) {
         if (executed) throw new IllegalStateException("Already Executed");
         executed = true;
       }
       captureCallStackTrace();
       eventListener.callStart(this);
       client.dispatcher().enqueue(new AsyncCall(responseCallback));
     }
   ```

   最终调用了Dispatch的enqueue(),在这个过程,将一步请求封装在AsyncCall,对于同步请求来说,是封装一个RealCall,AsyncCall是RealCall的静态内部类.

3. Dispatcher.enqueue()

   ```java
   synchronized void enqueue(AsyncCall call) {
       if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
         runningAsyncCalls.add(call);
         executorService().execute(call);
       } else {
         readyAsyncCalls.add(call);
       }
     }
   ```

   Dispatch中的enqueue是一个线程安全的方法.当异步请求数量小于最大请求数量并且同一主机的请求数量小于最大请求数量时,将该请求加入立即执行的请求的队列,同时在线程池中完成请求,反之加入等待队列中,待满足条件之后,重新发起请求.

4. 执行过程

   ```java
   public abstract class NamedRunnable implements Runnable {
     protected final String name;
   	//给每一个Runnablet添加一个名字,目前还不知道为什么?
     public NamedRunnable(String format, Object... args) {
       this.name = Util.format(format, args);
     }
   
     @Override public final void run() {
       String oldName = Thread.currentThread().getName();
       Thread.currentThread().setName(name);
       try {
         execute();
       } finally {
         Thread.currentThread().setName(oldName);
       }
     }
   
     protected abstract void execute();
   }
   
   final class AsyncCall extends NamedRunnable {
       private final Callback responseCallback;
   
       AsyncCall(Callback responseCallback) {
         super("OkHttp %s", redactedUrl());
         this.responseCallback = responseCallback;
       }
   
       String host() {
         return originalRequest.url().host();
       }
   
       Request request() {
         return originalRequest;
       }
   
       RealCall get() {
         return RealCall.this;
       }
   
       @Override protected void execute() {
         boolean signalledCallback = false;
         try {
           //通过各级拦截器获取响应
           Response response = getResponseWithInterceptorChain();
           if (retryAndFollowUpInterceptor.isCanceled()) {
             signalledCallback = true;
             responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
           } else {
             signalledCallback = true;
             //回掉返回
             responseCallback.onResponse(RealCall.this, response);
           }
         } catch (IOException e) {
           if (signalledCallback) {
             // Do not signal the callback twice!
             Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
           } else {
             eventListener.callFailed(RealCall.this, e);
             responseCallback.onFailure(RealCall.this, e);
           }
         } finally {
             //请求结束,结束这个请求
           client.dispatcher().finished(this);
         }
       }
     }
   ```

5. 结束一个请求

   ```java
   private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
       int runningCallsCount;
       Runnable idleCallback;
       synchronized (this) {
           //从队列中移除这个请求
         if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
   		//执行下一个异步请求
         if (promoteCalls) promoteCalls();
         runningCallsCount = runningCallsCount();
           //所以请求结束之后,执行idleCallBack
         idleCallback = this.idleCallback;
       }
   
       if (runningCallsCount == 0 && idleCallback != null) {
         idleCallback.run();
       }
     }
   ```

6. 执行下一个异步请求

   ```java
   private void promoteCalls() {
       if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
       //如果缓存队列里没有待执行的请求,则跳出
       if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.
   	//遍历等待队列
       for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
         AsyncCall call = i.next();
   
         if (runningCallsForHost(call) < maxRequestsPerHost) {
             //执行请求,将该请求从等待队列中删除,添加到运行队列
           i.remove();
           runningAsyncCalls.add(call);
           executorService().execute(call);
         }
   
         if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
       }
     }
   ```
