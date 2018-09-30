## AIDL-Binder

最近看了些进程间通信的文章,特别是Binder机制,在Android中使用Binder最通俗的就是AIDL了,好长时间不使用AIDL了,借此文章回顾下,同时加深下Binder机制的理解,也当一个学习笔记,gif中显示的内容是两个应用程序,使用分屏模式录制的.
![](https://user-gold-cdn.xitu.io/2018/9/29/1662539a974eab56?w=388&h=598&f=gif&s=4194304)

```
本文结构:
1. AIDL中涉及相关的变量方法解析
2. 过程中遇到的问题解析
3. Binder机制的知识了解
```
## 1. AIDL中涉及相关的变量方法解析

以下是通过AndroidStudio自动通过接口生成的

```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /Users/foking/DemoContainer/AIDLDemo/James/src/main/aidl/com/example/foking/aidldemo/JamesChatInterface.aidl
 */
package com.example.foking.aidldemo;
// Declare any non-default types here with import statements

public interface JamesChatInterface extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.example.foking.aidldemo.JamesChatInterface {
        private static final java.lang.String DESCRIPTOR = "com.example.foking.aidldemo.JamesChatInterface";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.example.foking.aidldemo.JamesChatInterface interface,
         * generating a proxy if needed.
         */
        public static com.example.foking.aidldemo.JamesChatInterface asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.foking.aidldemo.JamesChatInterface))) {
                return ((com.example.foking.aidldemo.JamesChatInterface) iin);
            }
            return new com.example.foking.aidldemo.JamesChatInterface.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_receiveKobeChatContent: {
                    data.enforceInterface(descriptor);
                    java.lang.String _arg0;
                    _arg0 = data.readString();
                    boolean _result = this.receiveKobeChatContent(_arg0);
                    reply.writeNoException();
                    reply.writeInt(((_result) ? (1) : (0)));
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        private static class Proxy implements com.example.foking.aidldemo.JamesChatInterface {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /**
             * Demonstrates some basic types that you can use as parameters
             * and return values in AIDL.
             */
            @Override
            public boolean receiveKobeChatContent(java.lang.String content) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                boolean _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeString(content);
                    mRemote.transact(Stub.TRANSACTION_receiveKobeChatContent, _data, _reply, 0);
                    _reply.readException();
                    _result = (0 != _reply.readInt());
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

        static final int TRANSACTION_receiveKobeChatContent = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    }

    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    public boolean receiveKobeChatContent(java.lang.String content) throws android.os.RemoteException;
}

```

**1.1 TRANSACTION_receiveKobeChatContent**

标识接口中方法生成一个常量id,当Client端调用的时候,通过Code码来区别

**1.2 Stub**

继承自Binder是一个抽象类实现了待提供方法的接口，封装了两个关键的方法`asInterface`,`onTransact`,最后我们自己的实体Binder类的要继承子该类,并且实现接口中的方法.

**1.3 Stub#DESCRIPTOR**

Binder实体的唯一标示

**1.4 Stub#asInterface()**

 用于将服务端的Binder对象转换为客户端所需的AIDL接口类型的对象,这种转换过程是区分进程的,如果客户端和服务端在同一个进程中,那个此方法返回的就是服务端的Stub对象本身,否则返回的是系统封装后的Stub.proxy对象

**1.5 Stub#onTransact()**

这个方法运行在服务端的Binder线程池中,当客户端发起跨进程请求时,远程请求会通过系统底层封装后交由此方法来处理,服务端通过code可以确定客户所请求的方法是什么,接着从data中取出目标方法所需要的参数,然后执行目标方法,当目标方法执行完毕之后,就像reply中写入返回的值,需要注意的是,如果此方法返回的是false,那么客户端的请求会失败,我们可以利用这个特性来做权限认证,毕竟我们也不希望随便一个进程都能远程调用我们的服务

**1.6 Proxy**

客户端Binder的本地代理对象

**1.7 Proxy#receiveKobeChatContent()**

这个方法运行在客户端,当客户端远程调用此方法时,他的内部是这样实现的:首先创建该方法所需要的输入型Parcel对象_data,输出型Parcel对象_reply,和返回值result,然后把该方法的参数写入_data中,接着调用transact发起请求,同时请求线程被挂起,然后服务端的onTransact方法被调用,直到请求完成,当前线程继续执行,并从_reply中取出返回结果,最后返回_reply的数据

## 2. 过程中遇到的问题解析
**2.1 报“java.lang.IllegalArgumentException: Service Intent must be explicit”**
```java
报错代码:
Intent intent = new Intent("foking.kobe");
bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);

修改之后:
Intent intent = new Intent("foking.kobe");
intent.setPackage("com.example.kobe");
bindService(intent, mServiceConnection, Context.BIND_AUTO_CREATE);
显示指定Action和Package（被调用应用的包名）。
```
**2.2 支持的数据类型**
1. Java 编程语言中的所有原语类型（如 int、long、char、boolean 等等）
2. String、CharSequence、List、Map

```
传递对象需要实现Parcel接口,并且在使用的时候需要指示数据走向的方向标记。
可以是 in、out 或 inout,原语默认为 in，不能是其他方向。
```
AIDL中的定向 tag 表示了在跨进程通信中数据的流向，其中 in 表示数据只能由客户端流向服务端， out 表示数据只能由服务端流向客户端，而 inout 则表示数据可在服务端与客户端之间双向流通。其中，数据流向是针对在客户端中的那个传入方法的对象而言的。in 为定向 tag 的话表现为服务端将会接收到一个那个对象的完整数据，但是客户端的那个对象不会因为服务端对传参的修改而发生变动；out 的话表现为服务端将会接收到那个对象的参数为空的对象，但是在服务端对接收到的空对象有任何修改之后客户端将会同步变动；inout 为定向 tag 的情况下，服务端将会接收到客户端传来对象的完整信息，并且客户端将会同步服务端对该对象的任何变动。细节请点击
[你真的理解AIDL中的in，out，inout么？](https://blog.csdn.net/luoyanglizi/article/details/51958091)

## 3.Binder机制的知识了解

**3.1 虚拟内存**

虚拟内存是计算机系统内存管理的一种技术。它使得应用程序认为它拥有连续可用的内存（一个连续完整的地址空间），而实际上，它通常是被分隔成多个物理内存碎片，还有部分暂时存储在外部磁盘存储器上，在需要时进行数据交换。与没有使用虚拟内存技术的系统相比，使用这种技术的系统使得大型程序的编写变得更容易，对真正的物理内存（例如RAM）的使用也更有效率。这个概念很重要，对于虚拟内存更深入的理解可以参考这篇文章：Linux 虚拟内存和物理内存的理解.

**3.2 用户空间和内存空间**

我们知道现在操作系统都是采用虚拟存储器，那么对32位操作系统而言，它的寻址空间（虚拟存储空间）为4G（2的32次方）。操心系统的核心是内核，独立于普通的应用程序，可以访问受保护的内存空间，也有访问底层硬件设备的所有权限。为了保证用户进程不能直接操作内核，保证内核的安全，操心系统将虚拟空间划分为两部分，一部分为内核空间，一部分为用户空间。针对linux操作系统而言，将最高的1G字节（从虚拟地址0xC0000000到0xFFFFFFFF），供内核使用，称为内核空间，而将较低的3G字节（从虚拟地址0x00000000到0xBFFFFFFF），供各个进程使用，称为用户空间。每个进程可以通过系统调用进入内核，因此，Linux内核由系统内的所有进程共享。于是，从具体进程的角度来看，每个进程可以拥有4G字节的虚拟空间。

**3.3 内核态和用户态**

当一个任务（进程）执行系统调用而进入内核代码中执行时，称进程处于内核运行态（内核态）。此时处理器处于特权级最高的（0级）内核代码中执行。当进程处于内核态时，执行的内核代码会使用当前进程的内核栈。每个进程都有自己的内核栈。
当进程在执行用户自己的代码时，则称其处于用户运行态（用户态）。此时处理器在特权级最低的（3级）用户代码中运行。当正在执行用户程序而突然被中断程序中断时，此时用户程序也可以象征性地称为处于进程的内核态。因为中断处理程序将使用当前进程的内核栈。


![](https://user-gold-cdn.xitu.io/2018/9/30/1662973e486e43fa?w=1161&h=842&f=png&s=70006)
要想完全把Binder机制了解清楚,需要操作系统的知识,笔者在看目前网上的资料的时候,也是云里雾里,只是大致的过程了解了,这里只做一个记录;

**3.4 Android中的Binder**

什么叫Binder驱动 : 在 Android 系统中，运行在内核空间，负责各个用户进程通过 Binder 实现通信的内核模块就叫 Binder 驱动（Binder Dirver）。

此外linux本身是没有Binder驱动的,是通过Linux 的动态内核可加载模块（Loadable Kernel Module，LKM）的机制；模块是具有独立功能的程序，它可以被单独编译，但是不能独立运行。它在运行时被链接到内核作为内核的一部分运行.

一次完整的 Binder IPC 通信过程：

1. 首先 Binder 驱动在内核空间创建一个数据接收缓存区
2. 接着在内核空间开辟一块内核缓存区，建立内核缓存区和内核中数据接收缓存区之间的映射关系，以及内核中数据接收缓存区和接收进程用户空间地址的映射关系；
3. 发送方进程通过系统调用 copyfromuser() 将数据 copy 到内核中的内核缓存区，由于内核缓存区和接收进程的用户空间存在内存映射，因此也就相当于把数据发送到了接收进程的用户空间，这样便完成了一次进程间的通信。

Binder通信模型:

Client进程、Service进程、ServiceManager进程、Binder驱动

过程:

1. Service进程通过Binder驱动在ServiceManager中进行注册 Binder（Server 中的 Binder 实体）,表明可以对外提供服务
2. Client进程通过Binder驱动,在ServiceManager中获取了服务端Binder的引用
3. 通过这个应用就可以完成对Service端的访问

![](https://user-gold-cdn.xitu.io/2018/9/30/16629743abdc7fe4?w=720&h=514&f=jpeg&s=31190)