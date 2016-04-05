# Android AIDL 原理解析

如果去阅读Android的源代码，就会发现里面大量用到了Binder、AIDL相关知识，比如当我们去使用```AMS```、```PMS```、```WMS```这些核心服务，因为他们都运行在 ```system_server``` 进程，普通应用想调用他们提供的服务（例如：```startActivity()```，就需要```AMS```来实现），就必须要跨进程调用，因此，我们在阅读代码之前，必须先去尝试理解Binder、AIDL相关知识。

为什么要使用AIDL呢？

通过AIDL，可以让本地调用远程服务器的接口就像调用本地接口那么简单，让用户无需关注内部细节，只需要实现自己的业务逻辑接口，内部复杂的参数序列化发送、接收、客户端调用服务端的逻辑，你都不需要去关心了。

## 一 从一个例子开始

我们通过一个简单的跨进程调用的例子来理解AIDL。

设计一个简单的场景：

> 我们有一个```CoreService```运行在```"com.cundong.practice:core"```进程中，提供文件下载服务，我们会在```"com.cundong.practice"```进程中去bind这个Service，调用它为我们提供的服务。

实现起来很简单，只需要以下几个步骤：

 - 定义一个AIDL文件
```java
 interface IDownloadService {
 
    /**
     * 下载
     * @param url
     */
     void download(String url);

     /**
     * 删除下载任务
     * @param url
     */
     void delete(String url);

     /**
     * 停止下载任务
     * @param url
     */
     void stop(String url);

     /**
     * 获取下载队列大小
     * @return
     */
     int getQueueSize();
}
```

- 通过AIDL的代码生成器，生成实现Proxy、Stub代码逻辑
以Android Studio为例，自动生成的IDownloadService.java位于\app\build\generated\source\aidl\debug目录下，核心方法我已添加注释。

```java
public interface IDownloadService extends android.os.IInterface {
    /**
     * 下载
     *
     * @param url
     */
    public void download(java.lang.String url) throws android.os.RemoteException;

    /**
     * 删除下载任务
     *
     * @param url
     */
    public void delete(java.lang.String url) throws android.os.RemoteException;

    /**
     * 停止下载任务
     *
     * @param url
     */
    public void stop(java.lang.String url) throws android.os.RemoteException;

    /**
     * 获取下载队列大小
     *
     * @return
     */
    public int getQueueSize() throws android.os.RemoteException;

    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.cundong.touch.IDownloadService {
    
        // 注意，通过以下代码我们发现，我们在.aidl中定义的方法名字其实没什么意义，服务器端根本没有使         // 用这些名字，而是自动为这些方法生成了递增的ID，根据方法的顺序而不是名字来一个一个处理
        static final int TRANSACTION_download = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_delete = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        static final int TRANSACTION_stop = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
        static final int TRANSACTION_getQueueSize = (android.os.IBinder.FIRST_CALL_TRANSACTION + 3);
        private static final java.lang.String DESCRIPTOR = "com.cundong.touch.IDownloadService";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.cundong.touch.IDownloadService interface,
         * generating a proxy if needed.
         * 
         * 注：
         * 这个方法里面有个处理，通过queryLocalInterface查询，如果服务端和客户端都是在同一个进程，那          * 么就不需要跨进程了，直接将IDownloadService当做普通的对象来使用即可，否则返回远程对象
         * 的代理
         */
        public static com.cundong.touch.IDownloadService asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.cundong.touch.IDownloadService))) {
                return ((com.cundong.touch.IDownloadService) iin);
            }
            return new com.cundong.touch.IDownloadService.Stub.Proxy(obj);
        }
        
        /**
        * 返回Binder实例
        */
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        /**
        * 根据code参数来处理，这里面会调用真正的业务实现类
        */
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_download: {
                    data.enforceInterface(DESCRIPTOR);
                    java.lang.String _arg0;
                    _arg0 = data.readString();
                    this.download(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_delete: {
                    data.enforceInterface(DESCRIPTOR);
                    java.lang.String _arg0;
                    _arg0 = data.readString();
                    this.delete(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_stop: {
                    data.enforceInterface(DESCRIPTOR);
                    java.lang.String _arg0;
                    _arg0 = data.readString();
                    this.stop(_arg0);
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_getQueueSize: {
                    data.enforceInterface(DESCRIPTOR);
                    int _result = this.getQueueSize();
                    reply.writeNoException();
                    reply.writeInt(_result);
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.cundong.touch.IDownloadService {
            private android.os.IBinder mRemote;
            
            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }
            
            //Proxy的asBinder()返回位于本地接口的远程代理
            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }
            
            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /**
             * 下载
             *
             * @param url
             */
            @Override
            public void download(java.lang.String url) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeString(url);
                    
                    //Proxy中， 业务接口中其实没有做任何业务逻辑处理，仅仅是收集本地参数，序列化
                    //后通过IBinder的transact方法，发给服务器端，并且通过_reply来接收服务器端
                    //的处理结果
                    mRemote.transact(Stub.TRANSACTION_download, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            /**
             * 删除下载任务
             *
             * @param url
             */
            @Override
            public void delete(java.lang.String url) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeString(url);
                    mRemote.transact(Stub.TRANSACTION_delete, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            /**
             * 停止下载任务
             *
             * @param url
             */
            @Override
            public void stop(java.lang.String url) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeString(url);
                    mRemote.transact(Stub.TRANSACTION_stop, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            /**
             * 获取下载队列大小
             *
             * @return
             */
            @Override
            public int getQueueSize() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                int _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getQueueSize, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readInt();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }
    }
}

```
- 继承```IDownloadService.Stub```，实现真正的服务器端接口

```java
public class IDowanloadServiceImpl extends IDownloadService.Stub {
    private final Random mGenerator = new Random();

    @Override
    public void download(String url) throws RemoteException {
        //TODO 真正的业务逻辑
    }

    @Override
    public void delete(String url) throws RemoteException {
        //TODO 真正的业务逻辑
    }

    @Override
    public void stop(String url) throws RemoteException {
        //TODO 真正的业务逻辑
    }

    @Override
    public int getQueueSize() throws RemoteException {
        //TODO 真正的业务逻辑
        return mGenerator.nextInt(100);
    }
}
```
- 远程Service的onBind()接口返回Stub的IBinder
```java
public class CoreService extends Service {

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return new IDowanloadServiceImpl();
    }
}
```

- 客户端发起远程调用
```java
private ServiceConnection mConnection = new ServiceConnection() {

        @Override
        public void onServiceConnected(ComponentName className, IBinder service) {

            sIDownloadService = IDownloadService.Stub.asInterface(service);
            mBound = true;
        }

        @Override
        public void onServiceDisconnected(ComponentName arg0) {
            mBound = false;
        }
    };
 ```
 
 ```java
 //调用
 if (mBound) {
         int size = 0;
         try {
            size = sIDownloadService.getQueueSize();
         } catch (RemoteException e) {
             e.printStackTrace();
         }
  }
 ```
 
## 二 原理分析

通过阅读自动生成的IDownloadService.java，可以看出，这是一个经典的代理模式架构。AIDL的代码生成器，已经根据.aidl文件自动帮我们生成Proxy、Stub（抽象类）两个类，并且把客户端代理mRemote的transact()过程以及
服务器端的onTtransact()过程默认实现好了，我们只需要在服务器端继承Stub，实现自己的业务类（在onTtransact()中会调用）。

### 1. UML图
![IDownloadService.java UML图][4]

### 2. 代码分析

代码主要分为Proxy、Stub两部分。

#### 1) Proxy

Proxy运行在客户端，它实现了IDownloadService接口，并且持有一个远程代理IBinder mRemote，mRemote不做任何业务逻辑处理，仅仅通过IBinder接口的transact()方法，把客户端的调用参数序列化后transact到远程服务器。

示例：
```java
@Override
public void download(java.lang.String url) throws android.os.RemoteException {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
            try {
                _data.writeInterfaceToken(DESCRIPTOR);
                _data.writeString(url);
                mRemote.transact(Stub.TRANSACTION_download, _data, _reply, 0);
                _reply.readException();
            } finally {
                _reply.recycle();
                _data.recycle();
            }
}         
```
_data即调用接口传入的参数，_reply为调用方法得到的返回值，```mRemote.transact(Stub.TRANSACTION_download, _data, _reply, 0);```为调用过程。

#### 2) Stub

Stub运行在服务器端，继承自Binder，同样也实现了IDownloadService接口，它的核心逻辑在onTransact方法中：

```java
@Override
public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
    switch (code) {
        case TRANSACTION_download: {
                data.enforceInterface(DESCRIPTOR);
                java.lang.String _arg0;
                _arg0 = data.readString();
                this.download(_arg0);
                reply.writeNoException();
                return true;
        }
        case TRANSACTION_delete: {
            ...
        }
        case TRANSACTION_stop: {
            ...
        }
        case TRANSACTION_getQueueSize: {
            ...
        }
    }
    return super.onTransact(code, data, reply, flags);
}
```
远程服务器端通过IBinder接口的onTtransact()方法来接收数据、处理数据，并且调用真实的业务逻辑代码（如上面的download接口），通过reply像客户端传递返回值。

另外，Stub中另外一个比较重要的接口就是asInterface()接口，我们在客户端真正使用的时候通常会这样使用它：

```java
IDownloadService sIDownloadService = IDownloadService.Stub.asInterface(IBinder service);
```
通过方法名字，我们大致可以猜出，它大概实现的功能，就是将一个IBinder对象转化为接口对象IDownloadService，通过代码可以看到具体逻辑：

```java
public static com.cundong.touch.IDownloadService asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.cundong.touch.IDownloadService))) {
                return ((com.cundong.touch.IDownloadService) iin);
            }
            return new com.cundong.touch.IDownloadService.Stub.Proxy(obj);
        }
```
 先通过queryLocalInterface查询，如果服务端和客户端都是在同一个进程，那么就不需要跨进程了，直接将IDownloadService当做普通的对象来使用，否则会返回远程对象的代理对象。


至此，我们就通过源码层面了解了AIDL是如何工作了。

参考：

[1][http://www.jianshu.com/p/cfb1d2a109a2]

[2][http://blog.csdn.net/xude1985/article/details/9232049]

[3][http://developer.android.com/intl/zh-cn/guide/components/aidl.html]


  [1]: http://www.jianshu.com/p/cfb1d2a109a2
  [2]: http://blog.csdn.net/xude1985/article/details/9232049
  [3]: http://developer.android.com/intl/zh-cn/guide/components/aidl.html
  [4]: http://7xr1sh.dl1.z0.glb.clouddn.com/469751777359982261.jpg