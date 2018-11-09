[理解Android进程创建流程](http://gityuan.com/2016/03/26/app-process-create/)

[一个APP从启动到主页面显示经历了哪些过程?](https://www.jianshu.com/p/a72c5ccbd150)

```sequence
participant Launch进程
participant System_Service(AMS)
participant Zygote进程
participant APP进程

Launch进程->System_Service(AMS):1. 通过Binder机制,调用AMS的startActivity()
Note over System_Service(AMS):2.如果目标Process不存在
System_Service(AMS)->Zygote进程:3.通过Socket通知Zygote进程fork子进程
Zygote进程->APP进程:4.fork出子进程,调用ActivityThread.main() 
```

**Zygote进程:**该进程是所有Java进程的父进程

**System_Service进程:**也是由Zygote进程fork出来的,该进程负责启动各种系统服务,AMS,PMS等系统级别的服务

```java 
 public static void main(String[] args) {
		...
        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

最后在fork出的进程中,会通过反射调用ActivityThread.main()方法,接着 APP进程会通过Biner机制调用AMS`attachApplicaion()`方法,之后AMS会通过Binder机制调用scheduleLaunchActivity()