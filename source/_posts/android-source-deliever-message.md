---
title: Android消息机制源码解析
date: 2016-03-10 13:25:43
categories: [android, java]
tags: [android, java]
---
## 前言
相信做过Android开发的同学都知道，主线程中不能做长时间的操作，否则会出现ANR。耗时的操作应该放在其他线程中，操作结束后通过Handler发送一个消息，主线程接收到消息以后进行UI操作。这个过程中牵涉到Android的消息机制，为了更好的学习和理解Android消息机制，这篇文章将从源码的角度进行介绍和分析。

## 源码分析
Android消息机制主要牵涉以下几个类：**Looper**、**Handler**、**MessageQueue**、**Message**。在这几个类的协同配合下，构建出了android的消息系统。在介绍这几个类之前，还有一个不得不提到的类就是**ThreadLocal**。关于ThreadLocal的详细介绍请参考[**彻底理解ThreadLocal**](http://blog.csdn.net/lufeng20/article/details/24314381)。

### ThreadLocal
ThreadLocal类适用于在线程生命周期内数据的长期存储。

![android_source_deliever_message_threadlocal](/images/android_source_deliever_message_threadlocal.jpg)

上面的图简单介绍了ThreadLocal的工作原理。每个Thread都有一个Values叫做localValues，localValues是一个存放object的数组，存放规则是threadlocal和具体存储的object成对存放。在ThreadLocal的set和get方法调用时，首先会根据当前所在的线程（如Thread2），找到对应的localValues数组（localValues2）。
* threadlocal.get example1：Thread2对应localValues2，数组中找到调用的threadlocal引用（即threadlocal22），返回object22。
* threadlocal.get example2：Thread2对应localValues2，数组中找不到调用的threadlocal引用（即threadlocal11），返回null。
* threadlocal.set example1：Thread2对应localValues2，数组中找到调用的threadlocal引用（即threadlocal22），更新object22为newObject。
* threadlocal.set example2：Thread2对应localValues2，数组中找不到调用的threadlocal引用（即threadlocal11），在数组末尾新增两个对象，分别是threadlocal11和newObject。

### MessageQueue
MessageQueue是管理消息的队列，从数据结构上将它是一个单链表。它有两个最重要的方法：**enqueueMessage**和**next**。

**enqueueMessage**的作用是按照消息处理的时间顺序，将当前需要处理的message插入到链表中的合适位置。enqueueMessage方法的源码如下：
``` java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        ...
    }
    return true;
}
```
* Line2：msg.target是一个handler对象，会在后面的文章中详细介绍，这里需要知道加入链表的message的handler对象不能为null。
* Line5：msg.isInUse()返回该message是否正在使用中。我们在使用message的时候，尽量采用handler的obtainMessage方法，这样获得的Message会清除掉使用状态，在后面的介绍中会提到。
* Line10：mQuitting是Looper退出的标志位，当looper执行quit方法之后，发送的message对象都不会再加到消息队列当中了。Looper会在后面详细介绍。
* Line17-44：实现了按照处理时间顺序，将message插入适当的位置。

**next**的作用是根据当前系统时间，在单链表所有满足“消息处理时间早于当前系统时间”的消息中，找到处理时间最早的一条消息，返回该消息，并将该消息从链表中移除。next方法的源码如下：
``` java
Message next() {
    ...
    for (;;) {
    	...
        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }
            ...
        }
        ...
    }
}
```
* Line10-37：实现了查找消息的功能。
* Line40: mQuitting是Looper退出的标志位，当looper执行quit方法之后，该方法直接返回null。

需要注意的是：在enqueueMessage和next方法中都加入了synchronized(this)，从而保证了在一个消息队列中，插入和获取消息的线程安全，也就保证了消息队列的准确性。

### Looper
Looper从字面上理解，是一个循环器，它的作用是不断的查询并分派对应线程中所有需要处理的消息。
Looper的构造方法为：
``` java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```
在构建looper对象时，就会生成成员变量mQueue和mThread。

Looper有两个重要的成员变量，**sThreadLocal**和**mQueue**，类型分别是**ThreadLocal**和**MessageQueue**。

**sThreadLocal(ThreadLocal类)**，是用于存储不同进程中对应的Looper对象。根据前面对于ThreadLocal工作原理的介绍，sThreadLocal.get方法返回执行线程存储的object对象，这里就是looper对象，而sThreadLocal.set方法则将一个Looper对象存放到执行线程对应的localValues数组当中。

**mQueue(MessageQueue类)**，是一个消息队列。

Looper还有三个重要的方法：**prepare**、**loop**和**quit**。

**prepare**方法的作用是生成一个新的Looper对象，并执行threadLocal.set方法保存到线程的localValues数组当中。prepare方法的源码如下：
``` java
private static void prepare(boolean quitAllowed) {
    if (sThreadLocal.get() != null) {
        throw new RuntimeException("Only one Looper may be created per thread");
    }
    sThreadLocal.set(new Looper(quitAllowed));
}
```
* 静态方法，会根据当前执行的线程进行处理。
* 在某个线程中第一次调用Looper.prepare方法，此时在执行线程的localValues数组当中找不到sThreadLocal对象，因此sThreadLocal.get方法会返回null。再调用sThreadLocal.set方法，将sThreadLocal和Looper对象成对保存到线程的localValues数组中。
* 在相一线程中第二次调用Looper.prepare方法，此时在执行线程的localValues数组当中可以找到sThreadLocal对象，因此sThreadLocal.get方法会返回之前保存的looper对象，会抛出异常。
* 结论：
  1、一个线程中只能调用一次Looper.prepare方法。
  2、由于threadLocal的原理，looper对象保存在执行线程的localValues数组当中，可以认为looper对象是与线程一一对应的。

**loop**方法的作用是启动循环，查询并分派需要处理的消息。loop方法的源码如下：
``` java
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;
    ...
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }
        ...
        msg.target.dispatchMessage(msg);
        ...
    }
}
```
* 静态方法，会根据当前执行的线程进行处理。
* Line2：调用myLooper方法（即sThreadLocal.get方法），获得当前线程的localValues数组中，sTheadLocal对应的looper对象。这个looper对象与当前线程所对应。
* Line3-5：looper对象为null，抛出异常。异常说明中也详细说明了原因，在调用loop方法前需要先调用prepare方法。
  分析：在prepare方法中，会生成一个新的looper对象，并调用sThreadlocal.set方法保存在线程的localValues数组中。只有执行了这个过程之后，sThreadLocal.get方法才能获得到保存的looper对象。
* Line6：获得looper对象的消息队列。由于looper对象本身是与线程一一对应的，因此这个消息队列也是与线程一一对应的。
* Line8-17: 这是一个死循环，不断的查询需要处理的消息。其中：
  Line9：调用MessageQueue的next方法，获得需要处理的消息。
  Line10-13: 返回的消息为null，从MessageQueue的next方法中我们知道，当调用了Looper的quit方法以后，next才会返回null，表明查询循环已结束。
  Line15：msg.target是一个Handler对象，调用其dispatchMessage方法分派处理消息。具体的实现后面会详细分析。

**quit**方法的作用是结束消息查询循环，不再处理后续的消息。quit方法的源码如下：
``` java
public void quit() {
    mQueue.quit(false);
}
```
* 非静态方法，核心作用是将指定Looper对象下MessageQueue中的mQuitting赋值成true，表示结束循环。之后MessageQueue的next方法均为返回null。

### Handler
Handler的构造方法如下：
``` java
public Handler() {
    this(null, false);
}
public Handler(boolean async) {
    this(null, async);
}
public Handler(Callback callback) {
    this(callback, false);
}
public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }

    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
public Handler(Looper looper) {
    this(looper, null, false);
}
public Handler(Looper looper, Callback callback) {
    this(looper, callback, false);
}
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```
* 前三种构造方法都会进入到第四种构造方法当中(Line11-27)：
  Line20：调用myLooper方法（即sThreadLocal.get方法），获得当前线程的localValues数组中，sTheadLocal对应的looper对象。这个looper对象与当前线程所对应。
  Line21-24：获取的looper对象为null，这里和前面分析的原因一样，必须先执行Looper的prepare方法。
  Line25-26：handler对象的MessageQueue即Looper对象的MessageQueue。初始化CallBack，CallBack后面会进行介绍。
* 第五和第六种构造方法都会进入到第七种构造方法当中(Line36-39)：
  根据指定的Looper和CallBack初始化Handler对象，MessageQueue为指定Looper对象的MessageQueue。

Handler有三个重要的成员变量，**mLooper**、**mQueue**和**mCallback**。

**mLooper**是构造handler必须的一个成员，我们可以指定传入需要的looper对象，若不指定的话，就根据执行Handler构造方法时所处的线程，来构建线程对应的Looper对象(Looper中的sThreadLocal.get方法）。
* 不管是指不指定looper对象，handler都将与某一个looper对象所关联。由于looper对象本身是与线程绑定的，因此handler对象也具有与线程绑定的特性。

**mQueue**是handler对象中mLooper所拥有的MessageQueue。

**mCallback**是handler处理消息的回调接口，后面在handler处理消息方法中(dispatchMessage)，会详细介绍。

Handler有三个重要的方法：**obtainMessage**、**sendMessage**和**dispatchMessage**。

**obtainMessage**方法有很多的重载，差别在于Message对象携带的参数不同（这个参数用于Message的标识）。我们只需知道这个方法会返回一个Message对象msg，并且msg.target等于调用该方法的handler对象（在Looper的loop方法中有提到)。

**sendMessage**的作用是发送消息，并告知MessageQueue插入该消息。方法的源码如下：
``` java
public final boolean sendMessage(Message msg)
{
    return sendMessageDelayed(msg, 0);
}
public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
* 最终方法都会进入到enqueueMessage方法。该方法有三个参数，第一个参数是需要插入的MessageQueue消息队列，第二个参数是需要插入的Message消息，第三个参数是消息处理的系统时间。
* 正常调用sendMessage方法时：
  Line13会将handler对象的mQueue作为需要插入的MessageQueue
  Line23会将msg.target赋值成当前的handler对象。

**dispatchMessage**方法的作用是分派和处理消息。这一步是整个Android消息机制的最后一步。dispatchMessage方法的源码如下：
``` java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        handleMessage(msg);
    }
}
private static void handleCallback(Message message) {
    message.callback.run();
}
public void handleMessage(Message msg) {
}
```
* Line2：msg.callback是一个Runnable对象，如果不为空，优先执行msg.callback.run。如new Handler().post一个Runnable对象r，会构建一个Message对象，r作为Message对象的callback。
* Line5-9：mCallback是Handler的成员变量。
* Line10：调用Handler的handlerMessage方法(参见Line16-17)。默认的方法是空方法，可以继承或者派生Handler的子类，覆写该方法来处理消息。

## 主线程的消息机制
我们知道Java程序的入口是main方法，Android的启动入口为ActivityThread类的main方法。源码如下：
``` java
public static void main(String[] args) {
    ...
    Looper.prepareMainLooper();
    ...
    Looper.loop();
    ...
}
```
* 调用了Looper的prepareMainLooper方法，这个方法其实就是在主线程中调用了Looper.prepare方法，为后续主线程中处理消息做了准备。
* 调用了Looper.loop方法，我们就可以在主线程中发送和处理消息了。

## 结论
通过前面的源码分析，我们可以得到以下结论：

![android.source_deliever_message_summury](/images/android.source_deliever_message_summury.jpg)

* Looper对象是与线程一一对应的，且一个线程只能有一个Looper对象。
* Looper对象拥有一个MessageQueue。
* 可以认为Thread、Looper和MessageQueue是捆绑构建Handler对象的。
  对于某线程threadA，有且只有一个looperA，同时对应一个messageQueueA，这些对象都是一一对应的。需要注意的是，looper可以构建出很多的handler对象，如handlerB、handlerC，但这些构建出来的handler对象均是与looper所在线程所对应的，handler的dispatchMessage方法（处理消息的最终方法）也是运行在looper所在线程当中的。
* 传递和处理消息都需要构建handler对象，构建handler对象前需要先调用Looper.prepare方法。注意，一个线程中，Looper.prepare方法只能调用一次。
* 处理消息需要启动Looper循环，即只有在调用了Looper.loop方法后，才启动消息处理机制。

## Demo

新建一个Android的程序（这里就不赘述了），在Launcher Activity代码如下：
``` java
public class MainActivity extends Activity {

    private Handler handler1 = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            System.out.println("handler1 is in Thread " + Thread.currentThread().getName());
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Thread threadA = new Thread() {
            private Handler handler2 = new Handler() {
                @Override
                public void handleMessage(Message msg) {
                    System.out.println("handler2 is in Thread " + Thread.currentThread().getName());
                }
            };
            @Override
            public void run() {
                Looper.prepare();
                final Handler handler3 = new Handler() {
                    @Override
                    public void handleMessage(Message msg) {
                        System.out.println("handler3 is in Thread " + Thread.currentThread().getName());
                    }
                };
                Thread threadB = new Thread() {
                    @Override
                    public void run() {
                        Message m3 = handler3.obtainMessage();
                        handler3.sendMessage(m3);
                        Message m2 = handler2.obtainMessage();
                        handler2.sendMessage(m2);
                        Message m1 = handler1.obtainMessage();
                        handler1.sendMessage(m1);
                    }
                };
                threadB.start();
                Looper.loop();
                System.out.println("in Thread after loop");
            }
        };
        threadA.start();
    }
}
```
### 声明Handler对象
声明Handler对象需要Handler对应线程先调用Looper.prepare方法。
handler1显然是在主线程中，主线程在Android启动时就已经调用了prepare方法了。
handler2是threadA的成员变量，因此所在线程为调用new Thread方法时所在的线程，此例中为主线程。已经调用了prepare方法了。
handler3是在threadA的run方法中定义的，因此handler3所在线程应为threadA。因此在handler3生成之前，我们需要在threadA中先调用Looper.prepare方法（Line22）。
### 发送消息
我们在threadB中同时向handler1、handler2、handler3中发送消息。主线程在Android启动时就已经调用了Looper.loop方法，我们需要处理的是threadA中的Looper.loop方法(Line41)。在threadA中执行了Looper.loop方法以后，就可以给threadA中对应的handler（即handler3）发送消息了。不过执行了loop方法后，threadA就阻塞在loop方法了，即Line42的打印语句不会被执行到。只有在threadA中执行了Looper的quit方法后，Line42的打印语句才会执行。我们可以在Line26之后插入一行代码`Looper.myLooper().quit();`即可。这行代码本质是调用了sThreadLocal.get方法，获取当前线程对应的Looper对象，并且执行quit结束查询。Looper.loop方法中获得的message返回null，结束循环。因此可以执行后面的语句了。
### 处理消息
handler1、handler2、handler3都覆写了handleMessage方法，这些方法分别是在主线程、主线程和threadA中执行的。相应的输出语句如下：
``` bash
03-11 05:13:48.806  18345-18358/com.example.xiajw.mystudy I/System.out﹕ handler3 is in Thread Thread-153
03-11 05:13:48.914  18345-18345/com.example.xiajw.mystudy I/System.out﹕ handler2 is in Thread main
03-11 05:13:48.914  18345-18345/com.example.xiajw.mystudy I/System.out﹕ handler1 is in Thread main
```
