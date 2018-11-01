### 阐述之前先需要明白几个知识点：
1.Android系统在启动的时候，SystemService进程会启动手机手机相关的服务，例如：ActivityMangerService，PackageManager，WindowManager。
2.应用程序在完成各种操作时，也是一种C/S架构，请求响应的模式。对ActivityManagerService来说负责管理Activity的生命周期。
3.ActivityManagerService和应用程序时两个进程，这就涉及到进程间的通信，它们之间是通过Binder机制来完成通信的。

### 主要过程：
1.Launcher应用程序通过Binder机制通知AMS(ActivityManagerService),它要启动一个Activity。

2.AMS通过Binder机制通知Launcher进入Paused状态。

3.Launcher通过Binder机制通知AMS他已经进入Paused的状态，这个时候AMS就创建了一个新的进程，用来启动一个ActivityThread实例，即将要启动的这个实例就在这个ActivityThread中进行。这个时候ActivityThread就开始走main方法，进行一系列初始化过程。
```
public final class ActivityThread {
    public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");
        SamplingProfilerIntegration.start();
        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);
        Environment.initForCurrentUser();
        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());
        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
}
```
4.ActivityThread通过BInder机制将一个ApplicationThread的Binder对象传入AMS以便之后与之进行交互。即上述代码中函数attach最终调用了ActivityManagerService的远程接口ActivityManagerProxy的attachApplication函数，传入的参数是mAppThread，这是一个ApplicationThread类型的Binder对象，它的作用是用来进行进程间通信的。

5.AMS收到Binder对象，部署完毕之后，再通过BInder机制调用ApplicationThread的scheduleLaucherActivity(),这个方法内部又会调用ActivityThread的queueOrSendMessage()方法，借助Handler，通知可以进行一系列的操作了。
```
private final class ApplicationThread extends ApplicationThreadNative {  
  
        ......  
  
        // we use token to identify this activity without having to send the  
        // activity itself back to the activity manager. (matters more with ipc)  
        public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,  
                ActivityInfo info, Bundle state, List<ResultInfo> pendingResults,  
                List<Intent> pendingNewIntents, boolean notResumed, boolean isForward) {  
            ActivityClientRecord r = new ActivityClientRecord();  
  
            r.token = token;  
            r.ident = ident;  
            r.intent = intent;  
            r.activityInfo = info;  
            r.state = state;  
  
            r.pendingResults = pendingResults;  
            r.pendingIntents = pendingNewIntents;  
  
            r.startsNotResumed = notResumed;  
            r.isForward = isForward;  
  
            queueOrSendMessage(H.LAUNCH_ACTIVITY, r);  
        }  
  
        ......  
  
    }
```
### BInder机制

![Binder机制图解](http://upload-images.jianshu.io/upload_images/3747483-6581b1044006352f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
Binder机制中，由一系列系统组件组成，Client、Service、ServiceManager、BInder驱动程序

    1. Client、Server和Service Manager实现在用户空间中，Binder驱动程序实现在内核空间中。
    2. Binder驱动程序和Service Manager在Android平台中已经实现，开发者只需要在用户空间实现自己的Client和Server。
    3. Binder驱动程序提供设备文件/dev/binder与用户空间交互，Client、Server和Service Manager通过open和ioctl文件操作函数与Binder驱动程序进行通信。
    4. Client和Server之间的进程间通信通过Binder驱动程序间接实现。
    5. Service Manager是一个守护进程，用来管理Server，并向Client提供查询Server接口的能力。

本文仅是对大致流程的一个综述，如果想深入研究，可参考下列文章
1.[Android深入浅出之Binder机制](http://www.cnblogs.com/innost/archive/2011/01/09/1931456.html)
2.[Android从启动到程序运行整个过程的整理](http://www.cnblogs.com/zyanrong/p/5661114.html)