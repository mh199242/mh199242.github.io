---
title: handler
date: 2020-01-02 14:18:07
tags:
---

# Android的消息机制

## Android的消息机制概述

Handler是Android消息机制的上层接口，开发过程中只需要和Handler交互即可。Handler的使用过程很简单，通过它可以轻松地将一个任务切换到Handler所在的线程中去执行。使用场景简介：有时候需要在子线程中进行耗时的I/O操作，可能是读取文件或者访问网络等，当耗时操作完成以后可能需要在UI上做一些改变，由于Android开发规范的限制，我们并不能在子线程中更新UI，这个时候可以通过Handler就可以将更新UI的操作切换到主线程中执行。

Android的消息机制主要是指Handler的运行机制，Handler的运行需要底层的MessageQueue和Looper的支撑。MessageQueue用来存储消息，以队列的形式对外提供插入和删除的操作。虽然叫消息队列，但它的内部存储结构不是真正的队列，而是采用单链表的数据结构来存储消息列表。Looper会以无限循环的形式去查询MessageQueue是否有新消息，如果有的话就处理消息。Looper中还有一个特殊的概念ThreadLocal，它的作用是可以在每个线程中存储数据。Handler创建的时候会采用当前线程的Looper来构造消息循环系统，ThreadLocal可以在不同的线程中互不干扰地存储并提供数据，通过ThreadLocal可以轻松获取每个线程的Looper。除了UI线程，其他线程默认没有Looper，如果需要使用Handler就必须为线程创建Looper。UI线程ActivityThread被创建时就会初始化Looper，这就是在UI线程中默认可以使用Handler的原因。


### 系统为什么不允许在子线程中访问UI？

Android的UI控件不是线程安全的，如果在多线程中并发访问，会导致UI控件处于不可预期的状态。如果对UI控件的访问加上锁机制，首先会让UI访问的逻辑变得复杂，其次锁机制会降低UI访问的效率，因为锁机制会阻塞某些线程的执行。因此，最简单高效的方法就是采用单线程模型，也就是只能主线程才处理UI操作。


## Handler的工作过程

Handler创建时会采用当前线程的Looper来构建内部的消息循环系统，如果当前线程没有Looper，需要创建Looper。Handler创建完毕后，线程内部的Looper和MessageQueue就可以和Handler一起协同工作。可以通过Handler的post方法将一个Runnable投递到Looper中去处理，也可以通过Handler的send方法发送一个消息到Looper去处理，post方法最终也是通过send方法来完成的。

### send方法的工作过程

当Handler的send方法被调用时，它会调用MessageQueue的enqueueMessage方法将消息插入到消息队列，然后Looper发现有新消息，就会处理这个消息，最终消息中的Runnable或者Handler的handleMessage方法会被调用。Looper是运行在创建Handler的线程中的，这样Handler中的业务逻辑就可以被切换到创建Handler的线程中执行。


## MessageQueue的工作原理

MessageQueue主要包含两个操作：插入和读取，读取操作本身会伴随着删除操作，插入和读取分别对应的方法是enqueueMessage和next。MessageQueue内部通过单链表的数据结构来维护消息列表，单链表在插入和删除上比较有优势。


### 插入队列的过程是：

~~~
 boolean enqueueMessage(Message msg, long when) {
   ...

   synchronized (this) {
       ...

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

       // We can assume mPtr != 0 because mQuitting is false.
       if (needWake) {
           nativeWake(mPtr);
       }
   }
   return true;
}
~~~

（1）如果队列为空，或者当前处理的时间点为0（when的数值，when表示Message将要执行的时间点），或者当前Message需要处理的时间点先于队列中的首节点，那么就将Message放入队列首部，否则进行第2步。

（2）遍历队列中Message，找到when比当前Message的when大的Message，将Message插入到该Message之前，如果没找到则将Message插入到队列最后。

（3）判断是否需要唤醒，一般是当前队列为空的情况下，next那边会进入睡眠，需要enqueue这边唤醒next函数。


### 读取消息的过程：

~~~
Message next() {
   ...
   int pendingIdleHandlerCount = -1; // -1 only during first iteration
   int nextPollTimeoutMillis = 0;
   for (;;) {
       if (nextPollTimeoutMillis != 0) {
           Binder.flushPendingCommands();
       }

       nativePollOnce(ptr, nextPollTimeoutMillis); // jni函数

       synchronized (this) {
           // Try to retrieve the next message.  Return if found.
           final long now = SystemClock.uptimeMillis();
           Message prevMsg = null;
           Message msg = mMessages;
           if (msg != null && msg.target == null) { //target 正常情况下都不会为null，在postBarrier会出现target为null的Message
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
               nextPollTimeoutMillis = -1; // 等待时间无限长
           }
       }
   }
}
~~~

判断msg的执行时间(when)是否比当前时间(now)的大，如果小，则将msg从队列中移除，并且返回msg，结束。如果大则设置等待时间nextPollTimeoutMillis为(int)

next方法是一个无限循环的方法，如果消息队列中没有消息，那么next方法会一直阻塞在这里。

## Looper的工作原理
Looper在Android的消息机制中扮演消息循环的角色，它会不停地从MessageQueue中查看是否有新消息，如果有新消息就会立刻处理，否则就一直阻塞。我们需要通过 Looper.prepare() 方法才能给当前线程创建 Looper 对象，并且需要调用 Looper.loop() 方法才能让当前线程的 MessageQueue 中的消息循环被处理。

~~~
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.";
    }

    //获取looper实例中的mQueue（消息队列）
    final MessageQueue queue = me.mQueue;

    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();
    //进入消息循环
    for (;;) {
        //next()方法用于取出消息队列里的消息
        //如果取出的消息为空，则线程阻塞
        Message msg = queue.next();
        if (msg == null) {
            return;
        }

        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        final long traceTag = me.mTraceTag;
        if (traceTag != 0) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        try {
            //消息派发：把消息派发给msg的target属性，然后用dispatchMessage方法去处理
            //Msg的target其实就是handler对象，下面会继续分析
            msg.target.dispatchMessage(msg);
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }

        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }

        final long newIdent = Binder.clearCallingIdentity();
        if (ident != newIdent) {
            Log.wtf(TAG, "Thread identity changed from 0x"
                    + Long.toHexString(ident) + " to 0x"
                    + Long.toHexString(newIdent) + " while dispatching to "
                    + msg.target.getClass().getName() + " "
                    + msg.callback + " what=" + msg.what);
        }
        //释放消息占据的资源
        msg.recycleUnchecked();
    }
}
~~~

（1）loop方法是一个死循环，唯一跳出循环的方式是MessageQueue的next方法返回了null。当Looper的quit方法被调用时，Looper就会调用MessageQueue的quit或者quitSafely方法通知消息队列退出，当消息队列被标记为退出状态时，它的next方法就会返回null，loop方法就可以停止无限循环。

（2）loop方法会调用MessageQueue的next方法获取新消息，next方法是一个阻塞操作，没有消息时，next方法会阻塞，导致loop方法也一直阻塞。

（3）如果next方法返回了新消息，Looper会通过msg.target.dispatchMessage(msg)处理这条消息，这里的msg.target就是发送这条消息的Handler对象。

## Handler的工作原理

Handler的工作过程主要包含消息的发送和接收过程。消息的发送可以通过post和send方法来实现，post方法最终也是通过send方法来实现的。

### Handler发送消息的过程

~~~
public final boolean sendMessage(Message msg){
   return sendMessageDelayed(msg, 0);
}

public final boolean sendMessageDelayed(Message msg, long delayMillis){
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
~~~

Handler发送消息的过程是向消息队列中插入了一条消息，MessageQueue的next()方法就会返回这条消息给Looper，Looper收到消息后就开始处理了，最终消息由Looper交由Handler来处理，Handler的dispatchMessage方法会被调用，Handler进入处理消息的阶段。dispatchMessage的实现如下：

~~~
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
~~~

### Handler处理消息的过程：

首先检查Message的callback是否为null，不为null就通过handleCallback来处理消息。Message的callback是一个Runnable对象，实际上就是Handler的post方法传递的Runnable参数，handleCallback的逻辑如下：

~~~
private static void handleCallback(Message message) {
    message.callback.run();
}
~~~

如果Handler的callback为空，检查mCallback是否为空，不为空就调用mCallback的handleMessage方法来处理消息。

最后调用Handler的handleMessage方法来处理消息。

## ThreadLocal的工作原理

 ThreadLocal是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储后，只有在指定线程中可以获取到存储的数据，其他线程无法获取到。当某些数据是以线程为作用域并且不同的线程有不同的数据副本时，就可以考虑采用ThreadLocal。比如对于Handler来说，它需要获取当前线程的Looper，Looper的作用域就是线程并且不同线程有不同的Looper，这个时候通过ThreadLocal就可以轻松实现Looper在线程中的存取数据。