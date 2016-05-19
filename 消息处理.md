#通过源码分析Android 的消息处理机制

我们知道，Android应用是通过消息来驱动的，每一个进程被fork之后，都会在该进程的UI线程（主线程）中启动一个消息队列，主线程会开启一个死循环来轮训这个队列，处理里面的消息。

通过Android进程的入口 ActivityThread#main 方法可以看到这个逻辑：

```java

    public static void main(String[] args) {

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        Looper.loop();
	
	//只要运行正常，都不会执行到这段代码
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

```

另外，我们在平时开发中，如果想在子线程中更新UI，必须手动创建一个Looper对象，并且调用它的loop方法：

```java
class TestThread extends Thread {
       public Handler mHandler;
	
        public void run() {

	    // 为当前线程创建Looper对象，同时消息队列MessageQueue也准备好了
            Looper.prepare();
  
            mHandler = new Handler() {
                public void handleMessage(Message msg) {
                    
		    //这里可以更新UI了

                }
            }; 
  
	    // 开始处理消息队列中的消息
            Looper.loop();
        }
}
```

在这两个过程中，都看到了Looper的身影，因为Looper就是消息队列的管理者，我们调用他的prepareMainLooper和prepare方法，就会为当前线程创建好消息队列，调用looper方法就会开始消息队列的轮询处理过程。

要理解消息处理机制，除了Looper，我们还要理解其他类：

> * MessageQueue：消息队列
> * Message：消息
> * Handler：消息的处理者

## UML类图
![此处输入图片的描述][1]

## 逐个类分析

### Looper

先从Looper开始，上面提到的两个静态方法 prepareMainLooper()和prepare()的作用就是为当前线程创建消息队列，loop()调度这个消息队列。

来看Looper#prepareMainLooper：

```java
 public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
	    
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
```
prepareMainLooper是通过调用prepare()方法实现，并且通过myLooper()方法得到了当前线程的Looper，主线程的Looper，就是sMainLooper。

接着看prepare方法：

```java
/**
*  @param uitAllowed 是否允许消息循环退出，在ActivityThread#main 中，我们创建的是主线程的消息循环，肯定不允许退出，其他地方，则可以退出
*/
private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
这个方法中为每一个线程new了一个Looper对象，并且设置到sThreadLocal中。sThreadLocal是ThreadLocal类型的变量（线程局部变量），它保证了每一个线程有一个自己独有的Looper对象，

myLooper()：
```java
/**
* 从sThreadLocal中获取当前线程的Looper对象
*/
public static Looper myLooper() {
        return sThreadLocal.get();
}
```
方法很简单，就是返回了当前线程所独有的那个Looper对象。

Looer对象创建好了，同时消息队列也创建好了，接下来就是让整个消息机制跑起来，这就需要通过Looper#loop实现：
```java
    public static void loop() {

	//得到当前线程的Looper
        final Looper me = myLooper();

	//没有Looper对象，直接抛异常
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }

	//得到当前Looper对应的消息队列
        final MessageQueue queue = me.mQueue;
	
	//一个死循环，不停的处理消息队列中的消息，消息的获取是通过MessageQueue的next()方法实现
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
		
	    //调用Message的target变量（也就是Handler了）的dispatchMessage方法来处理消息
            msg.target.dispatchMessage(msg);

            msg.recycleUnchecked();
        }
    }
```
首先，得到当前线程的Looper，再通过Looper得到当前线程的消息循环，然后轮询这个消息队列，处理每一个消息，下面我就来看消息队列MessageQueue以及消息Message。

### MessageQueue
MessageQueue就是Message队列，消息队列这一数据结构是通过一个Message链实现的，Message对象有一个next字段指向它的下一结点。

```java
public final class MessageQueue {
	Message mMessages;

	//消息队列的初始化，销毁，轮询过程，阻塞，唤醒，都是通过本地方法实现的
	private native static long nativeInit();
	private native static void nativeDestroy(long ptr);
	private native static void nativePollOnce(long ptr, int timeoutMillis);
	private native static void nativeWake(long ptr);
	private native static boolean nativeIsIdling(long ptr);


	/**
	*  把消息加入到消息循环中
        *  @param msg 
	*  @param when 是么时候被执行，在消息队列中会按照时间排序
 	*/
	boolean enqueueMessage(Message msg, long when) {}

	/**
	*  把消息加入到消息循环中
        *  @param msg 
	*  @param when 什么时候被执行
 	*/
	boolean enqueueMessage(Message msg, long when) {}
	
        /**移除消息*/
	void removeMessages(Handler h, int what, Object object) {
}
```

Message对象放入队列通过enqueueMessage()方法实现，消息的依次获取是通过next()方法实现，其中当获取不到可处理Message对象时，该方法会进入等待状态。

### Message

Message就是消息的封装对象，它实现了Parcelable接口，因此可以跨进程传输。

```java
public final class Message implements Parcelable {
	
	//消息的标识符
	public int what;

	//两个int形的扩展参数
	public int arg1; 
	public int arg2;

	//一个Object拓展参数
	public Object obj;

	//消息发送者的UID
	public int sendingUid = -1;
	
	//消息带的data
	Bundle data;

	//target对象，也就是处理当前消息的Handler
	Handler target;

	//指向消息队列中下一个消息
	Message next;
	
	//消息同步锁
	private static final Object sPoolSync = new Object();

	//消息池
	private static Message sPool;

	//消息池大小
	private static int sPoolSize = 0;

	//消息池最大消息数量
	private static final int MAX_POOL_SIZE = 50;
}

```
对消息的管理是通过一个消息池来实现的，因为消息池的存在，我们在需要创建一个Message对象时，最好使用可复用消息对象的方法来创建它。

正确的使用方法是：

```java
Message msg = mHandler.obtainMessage(WHAT);
msg.sendToTarget();
```
obtainMessage方法中实际上会调用Message#obtain方法来从消息池中取一个对象。

```java
public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

错误的使用方法：

```java
Message msg = new Message();
msg.what = WHAT;
mHandler.sendMessage(msg);
```
### Handler

Handler 就是消息循环（MessageQueue）中消息（Message）的真正处理者，通过Looper#loop可以看到：

```java
public static void loop() {

        for (;;) {
            Message msg = queue.next(); // might block

            msg.target.dispatchMessage(msg);
        }
    }
```
Message的target属性就是一个Handler对象，对消息的处理实际上是通过Handler#dispatchMessage方法来处理的，而dispatchMessag方法会调用Handler#handleMessage方法，我们可以重写handleMessage方法
来真正的处理消息的回调。

  [1]: http://7xr1sh.dl1.z0.glb.clouddn.com/304B.tmp.png