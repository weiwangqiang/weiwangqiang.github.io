---
layout:     post
title:      "Android跨进程之ADIL原理"
subtitle:   " \"带你剖析AIDL实现原理\""
date:       2021-05-16 17:30:00
author:     "Weiwq"
header-img: "img/background/post-sample-image.jpg"
catalog:  true
tags:
    - Android

---

> “在之前有写过一篇《大话android 进程通信之AIDL》，但是一直没有补充对应的实现原理“

## 引言

说实在的，AIDL跨进程方式用得比较少，也一直比较神秘，这篇文章将剖析AIDL的通信过程，开车！

## 一、AIDL的用法

列一下本文用到的AIDL通信方式，代码详情见  [大话android 进程通信之AIDL](https://blog.csdn.net/to_perfect/article/details/77871537) 

首先定义Server端

```java
public class BookManagerService extends Service {
    private List<Book> bookList = new ArrayList<>();
    private Binder binder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() {
            return bookList;
        }

        @Override
        public void addBook(Book book) {
            if (!bookList.contains(book)) {
                bookList.add(book);
            }
        }
    };
    
    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }
}
```

Client端的调用如下：

```java
private IBookManager iBookManager = null;
private ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.i(TAG, "onServiceConnected: ");
            iBookManager = IBookManager.Stub.asInterface(service);
            try {
                Book book = new Book(1, "web App");
                iBookManager.addBook(book);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            Log.i(TAG, "onServiceDisconnected: ");
        }
    };
```

在绑定service后，可以拿到server端提供的IBinder，通过asInterface方法转为对应的AIDL接口，即可实现与Server端的通信。

那asInterface 方法做了啥？我们知道，在定义AIDL接口后，build整个项目，就会生成对应的java接口。我们来看看对应的实现

```java
// AIDL接口对应的Java接口
public interface IBookManager extends android.os.IInterface {
  
    // 提供给client 继承的抽象类，client端可以实现 AIDL中定义的方法。
    public static abstract class Stub extends android.os.Binder implements com.example.aidl.IBookManager {
        ...
        public static com.example.aidl.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.aidl.IBookManager))) {
                // 同一个进程，不需要binder通信，直接获取对接的接口
                return ((com.example.aidl.IBookManager) iin);
            }
            // 涉及到跨进程，需要创建一个代理对象
            return new com.example.aidl.IBookManager.Stub.Proxy(obj);
        }

    @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(descriptor);
                    java.util.List<com.example.aidl.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(descriptor);
                    com.example.aidl.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.example.aidl.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }
        
        // 代理对象，用于处理binder通信相关的操作
        private static class Proxy implements com.example.aidl.IBookManager {
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
            
            @Override
            public java.util.List<com.example.aidl.Book> getBookList() throws android.os.RemoteException {
              android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.example.aidl.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.example.aidl.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
           
            @Override
            public void addBook(com.example.aidl.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }
        // 定义方法的调用序号
        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }
    // 对应的Java接口方法
    public java.util.List<com.example.aidl.Book> getBookList() throws android.os.RemoteException;
    public void addBook(com.example.aidl.Book book, com.example.aidl.Book book1) throws android.os.RemoteException;
}

```

可以看到，内部的抽象类Stub是实现了IBookManager接口，并且内部有一个私有的proxy类，用于处理binder通信。client端只需要基础Stub，并实现对应的自定的java接口即可。

所以 `IBookManager.Stub.asInterface(service)` 这段代码实际是返回了Stub的代理类——proxy，并传入obj作为mRemote。

## 二、Stub.Proxy类

这个proxy类主要的工作是 处理传参、返回值、资源回收的一些工作。拿 `addBook` 这个方法来说，具体如下：

```java
           @Override
            public void addBook(com.example.aidl.Book book) throws android.os.RemoteException {
                // 获取入参的 Parcel
                android.os.Parcel _data = android.os.Parcel.obtain();
                // 获取返回值的 Parcel
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    // 标注远程服务名称
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    // 调用远端的binder，即Stub类实现
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    // 读取异常
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
```

该方法通过Parcel 这个对象实现入参和返回值的传递（Parcel 主要作用是将我们的数据进行打包，用于在binder中传输）。

我们知道如果需要通过binder传递对象，那么这个对象必须要实现Parcelable接口，那是因为需要把对象打包到Parcel对象中。

```java
if ((book != null)) {
    // Parcel 内部采用了mNativePtr指针来记录写入的值。这里写入1，表示入参是一个有效值
     _data.writeInt(1);
    // 把 book的一些属性值写入_data，方便Server端反序列化
     book.writeToParcel(_data, 0);
 } else {
    // 入参为无效值，写入0
    _data.writeInt(0);
 }
```

在封装好Parcel后，proxy就直接调用了remote的transact方法，之后就是异常的校验动作。

proxy是怎么获取到返回值的呢？我们来看getBookList方法实现

```java
            @Override
            public java.util.List<com.example.aidl.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.example.aidl.Book> _result;
                try {
                    ....
                    _result = _reply.createTypedArrayList(com.example.aidl.Book.CREATOR);
                } finally {
                    ....
                }
                return _result;
            }
```

这里是拿到_reply对象，然后做了一次反序列化的动作。createTypedArrayList实现如下：

```java
public final class Parcel {
    public final <T> ArrayList<T> createTypedArrayList(Parcelable.Creator<T> c) {
            int N = readInt();
            if (N < 0) {	
                return null;
            }
            ArrayList<T> l = new ArrayList<T>(N);
            while (N > 0) {
                if (readInt() != 0) {
                    l.add(c.createFromParcel(this));
                } else {
                    l.add(null);
                }
                N--;
            }
            return l;
    }
}
```

Book 的实现如下：

```java
public class Book implements Parcelable {
    public int bookId;
    public String bookName;
    private Book(Parcel parcel) {
        bookId = parcel.readInt();
        bookName = parcel.readString();
    }
    
    public static final Creator<Book> CREATOR = new Creator<Book>() {

        @Override
        public Book createFromParcel(Parcel parcel) {
            return new Book(parcel);
        }

        @Override
        public Book[] newArray(int i) {
            return new Book[i];
        }
    };
}
```

## 三、Stub类

刚才有提到，Proxy addBook方法中会调用mRemote的transact方法，实现传参数和获取返回值。我们来看看mRemote是怎么来的。

回到 `IBookManager.Stub.asInterface` 方法，上面有说到，在构建Proxy对象的时候，需要传入IBinder对象，这个IBinder对象是onServiceConnected 传入的service。而service正是Server端的onBinder返回的binder对象。由于client端与Server端的AIDL接口是一样的，并且Server端的binder实现也是继承于Stub。所以Client端的proxy实际调用的是Server端的Stub对象。

我们先看看Stub的onTransact实现，这个方法比较长，传入的参数有4个

- code：用于标识是调用哪个方法，即AIDL是通过序号判断client端调用的是哪个方法
- data：存放方法调用传入的参数
- reply：存放方法调用返回的参数
- flags：用于判断Client端transact()方法是否立即返回

```java

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                   ....
               case TRANSACTION_getBookList: {
                    data.enforceInterface(descriptor);
                    // 调用服务端的方法，获取返回值
                    java.util.List<com.example.aidl.Book> _result = this.getBookList();
                    // 在reply头部写入无异常的标识
                    reply.writeNoException();
                    // 将数据写入reply中
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    // 执行对应的接口
                    data.enforceInterface(descriptor);
                    com.example.aidl.Book _arg0;
                    // 判断参数是否有效，这里与client端写参数前，也会写一个int值是对应的。
                    if ((0 != data.readInt())) {
                        // 反序列化，从data中获取参数对象
                        _arg0 = com.example.aidl.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    // 调用Server端的addBook方法，实现方法的逻辑
                    this.addBook(_arg0);
                   // 在reply头部写入无异常的标识
                    reply.writeNoException();
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }
```

我们可以看到，Stub类其实比较简单，主要处理client端的方法调用请求，获取传入的参数，调用Server端的接口，封装返回值。

问题来了Stub的onTransact方法是怎么调用的？上面不是说了，client端拿到了Server端的binder，然后直接调用的嘛~~

是这样吗？

接着看，Stub的父类是一个Binder，我们直接在Server端的addBook方法打个断点看看，调用栈如下，即是被Binder的execTransact方法回调的。

<img src="/img/blog_aidl_analyze/1.png" width="100%" height="40%"> 

Binder的execTransact方法如下，该方法是私有的，并且内部没有其他地方调用。我们注意到该方法的备注，说明是在C++层调用的该方法

```java
    // Entry point from android_util_Binder.cpp's onTransact
    private boolean execTransact(int code, long dataObj, long replyObj,
            int flags) {
        BinderCallsStats binderCallsStats = BinderCallsStats.getInstance();
        BinderCallsStats.CallSession callSession = binderCallsStats.callStarted(this, code);
        Parcel data = Parcel.obtain(dataObj);
        Parcel reply = Parcel.obtain(replyObj);
        boolean res;
        final boolean tracingEnabled = Binder.isTracingEnabled();
        try {
            if (tracingEnabled) {
                Trace.traceBegin(Trace.TRACE_TAG_ALWAYS, getClass().getName() + ":" + code);
            }
            res = onTransact(code, data, reply, flags);
        } catch (RemoteException|RuntimeException e) {
            ....
        }
    }
```

找到android_util_Binder.cpp的实现，里面有这么一个方法，

```java
// 传入code、data、reply flags参数
status_t onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0) override
    {
        JNIEnv* env = javavm_to_jnienv(mVM);

        ALOGV("onTransact() on %p calling object %p in env %p vm %p\n", this, mObject, env, mVM);

        IPCThreadState* thread_state = IPCThreadState::self();
        const int32_t strict_policy_before = thread_state->getStrictModePolicy();

        // 调用java层的方法，并传入参数
        jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
            code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
        ....
    }
```

那gBinderOffsets.mExecTransact 值是多少？

```java
// 注册一个binder
static int int_register_android_os_Binder(JNIEnv* env)
{
    jclass clazz = FindClassOrDie(env, kBinderPathName);

    gBinderOffsets.mClass = MakeGlobalRefOrDie(env, clazz);
    // 映射的即为java层binder的execTransact方法
    gBinderOffsets.mExecTransact = GetMethodIDOrDie(env, clazz, "execTransact", "(IJJI)Z");
    gBinderOffsets.mGetInterfaceDescriptor = GetMethodIDOrDie(env, clazz, "getInterfaceDescriptor",
        "()Ljava/lang/String;");
    gBinderOffsets.mObject = GetFieldIDOrDie(env, clazz, "mObject", "J");

    return RegisterMethodsOrDie(
        env, kBinderPathName,
        gBinderMethods, NELEM(gBinderMethods));
}
```

至此，server端的onTransact 调用链路就明了了，client端并不是直接调用Stub的onTransact方法，而是通过了Binder驱动，间接调用。

## 四、client端捕获异常

在处理参数的时候，读者有发现没有，无论是client端还是Server端，都会异常信息，这个是做什么用的？

在讲作用之前，先提一个问题：如何将Server端的异常抛给client端？

我们知道，client端在调用Server端的方法时候，必须要catch住RemoteException这个异常。如果Server端抛了一个RuntimeException，client端能捕获吗？

我们来看一个实例，修改Server端的getBookList 方法，并直接抛一个RuntimeException的异常。

```java
    @Override
        public void addBook(Book book) {
            if (!bookList.contains(book)) {
                bookList.add(book);
            }
            throw new RuntimeException("服务端挂了");
        }
```

然后在client端做一下打印，

```java
   @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Log.i(TAG, "onServiceConnected: -  ");
            iBookManager = IBookManager.Stub.asInterface(service);
            try {
                Book book = new Book(1, "web App");
                iBookManager.addBook(book);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
```

发现client端并没有收到Server端的异常信息

<img src="/img/blog_aidl_analyze/3.png" width="100%" height="40%">

反而是Server端抛出了异常

<img src="/img/blog_aidl_analyze/2.png" width="100%" height="40%">

难道Server端的异常无法被client端捕获？我们回到Stub实现类，发现在onTransact中，调用完方法后，首先会调用 `reply.writeNoException();` 

```java
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws   android.os.RemoteException {
           switch (code) {
                case TRANSACTION_getBookList: {
                    data.enforceInterface(descriptor);
                    java.util.List<com.example.aidl.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                   }
                   ....
            }
            .....
         }
```

从注解可以知道，writeNoException 的作用是标识该包裹没有任何异常，那有异常怎么办？

```java
public final class Parcel {
      /**
         * Special function for writing information at the front of the Parcel
         * indicating that no exception occurred.
         *
         * @see #writeException
         * @see #readException
         */
        public final void writeNoException() {
           .....
        }
}
```

在Parcel 类里面还找到一个类似方法 `writeException`，看起来是写入异常的接口，并且支持SecurityException，BadParcelableException，IllegalArgumentException等异常。

```java
   /** 用于将异常结果写入包裹的标头，以便在从事务返回异常时使用  
     * Special function for writing an exception result at the header of
     * a parcel, to be used when returning an exception from a transaction.
     * Note that this currently only supports a few exception types; any other
     * exception will be re-thrown by this function as a RuntimeException
     * (to be caught by the system's last-resort exception handling when
     * dispatching a transaction).
     * 
     */
public final void writeException(Exception e) {
        int code = 0;
        if (e instanceof Parcelable
                && (e.getClass().getClassLoader() == Parcelable.class.getClassLoader())) {
            // We only send Parcelable exceptions that are in the
            // BootClassLoader to ensure that the receiver can unpack them
            code = EX_PARCELABLE;
        } else if (e instanceof SecurityException) {
            code = EX_SECURITY;
        } else if (e instanceof BadParcelableException) {
            code = EX_BAD_PARCELABLE;
        } else if (e instanceof IllegalArgumentException) {
            code = EX_ILLEGAL_ARGUMENT;
        } else if (e instanceof NullPointerException) {
            code = EX_NULL_POINTER;
        } else if (e instanceof IllegalStateException) {
            code = EX_ILLEGAL_STATE;
        } else if (e instanceof NetworkOnMainThreadException) {
            code = EX_NETWORK_MAIN_THREAD;
        } else if (e instanceof UnsupportedOperationException) {
            code = EX_UNSUPPORTED_OPERATION;
        } else if (e instanceof ServiceSpecificException) {
            code = EX_SERVICE_SPECIFIC;
        }
        writeInt(code);
        StrictMode.clearGatheredViolations();
        if (code == 0) {
            // 当前异常不在支持的范围内，就直接抛出RuntimeException
            if (e instanceof RuntimeException) {
                throw (RuntimeException) e;
            }
            throw new RuntimeException(e);
        }
        writeString(e.getMessage());
        ...
    }
```

恩，问题是，reply固定调用的是writeNoException，怎么才能调用到writeException呢？

我们从之前分析的调用栈，再往上看看。

上面有提到Stub类重写了Binder的onTransact方法。并且Binder在调用onTransact方法的时候，会catch住异常，分析下面代码，就会发现，如果 onTransact方法发生异常，就会被catch主，并且当flags & FLAG_ONEWAY) != 0的时候，就会调用reply的writeException方法！！至此，我们知道Server端要怎么写入异常了。

```java
 private boolean execTransact(int code, long dataObj, long replyObj,
            int flags) {
        Parcel data = Parcel.obtain(dataObj);
        Parcel reply = Parcel.obtain(replyObj);
        boolean res;
        ...
        try {
            ...
            res = onTransact(code, data, reply, flags);
        } catch (RemoteException|RuntimeException e) {
            if ((flags & FLAG_ONEWAY) != 0) {
                if (e instanceof RemoteException) {
                    Log.w(TAG, "Binder call failed.", e);
                } else {
                    Log.w(TAG, "Caught a RuntimeException from the binder stub implementation.", e);
                }
            } else {
                // 写入异常
                reply.setDataPosition(0);
                reply.writeException(e);
            }
            res = true;
        } 
 }
```

那client端如何读取异常呢？之前一直提到，client端在读取server端返回值的时候，先会调用`readException`方法检查异常，我们看看这个方法做了什么

```java
public final class Parcel {
       /** 特殊功能，用于从包裹的标头读取异常结果，在接收到事务处理结果后使用。如果已将异常写入到包裹中，则会为您抛出异常，否则返回该异常，并让您从包裹中读取正常结果数据。
         * Special function for reading an exception result from the header of
         * a parcel, to be used after receiving the result of a transaction.  This
         * will throw the exception for you if it had been written to the Parcel,
         * otherwise return and let you read the normal result data from the Parcel.
         */
        public final void readException() {
            int code = readExceptionCode();
            if (code != 0) {
                String msg = readString();
                readException(code, msg);
            }
        }

        public final void readException(int code, String msg) {
            switch (code) {
                case EX_PARCELABLE:
                    if (readInt() > 0) {
                        SneakyThrow.sneakyThrow(
                                (Exception) readParcelable(Parcelable.class.getClassLoader()));
                    } else {
                        throw new RuntimeException(msg + " [missing Parcelable]");
                    }
                case EX_SECURITY:
                    throw new SecurityException(msg);
                case EX_BAD_PARCELABLE:
                    throw new BadParcelableException(msg);
                case EX_ILLEGAL_ARGUMENT:
                    throw new IllegalArgumentException(msg);
                case EX_NULL_POINTER:
                    throw new NullPointerException(msg);
                case EX_ILLEGAL_STATE:
                    throw new IllegalStateException(msg);
                case EX_NETWORK_MAIN_THREAD:
                    throw new NetworkOnMainThreadException();
                case EX_UNSUPPORTED_OPERATION:
                    throw new UnsupportedOperationException(msg);
                case EX_SERVICE_SPECIFIC:
                    throw new ServiceSpecificException(readInt(), msg);
            }
            throw new RuntimeException("Unknown exception code: " + code
                    + " msg " + msg);
        }
}
```

从备注上可以看出，主要是从reply中检查是否有异常信息，如果有就抛出来。

分析到这里，想必读者知道Server端如何将异常信息抛给client端了吧？Binder已经为我们封装好了实现方式。

还是上面的案例，我们把RuntimeException修改为Binder支持的异常 NullPointerException，再来试试

这回client端能捕获异常信息了，并且Server 端无任何异常打印

<img src="/img/blog_aidl_analyze/4.png" width="100%" height="40%">

## 后记

由于时间仓促，如有错误，还请多多指教。

——Weiwq  于 2021.05 广州