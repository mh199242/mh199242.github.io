---
title: Android开发艺术探索第二章
date: 2020-06-02 14:32:42
tags:
---

## IPC机制

### Android IPC简介

   IPC是Inter-Process Communication的缩写，含义是进程间通信或者跨进程通信，是指两个进程间进行数据交互的一个过程。线程是CPU调度的最小单元，同时线程是一种有限的系统资源。而进程一般指一个执行单元，在PC和移动设备上指一个程序或者一个应用。一个进程可以包含多个线程，因此进程和线程是包含与被包含的关系。最简单的情况下，一个进程中可以只有一个线程，即主线程，在Android里面主线程也叫UI线程，在UI线程里才能操作界面元素。很多时候，一个进程中需要执行大量耗时的任务，如果这些任务放在主线程中去执行就会造成界面无法响应，严重影响用户体验，这种情况在PC系统和移动系统中都存在，在Android中有一个特殊的名字叫做ANR(Application Not Responding)，即应用无响应。解决这个问题就需要用到线程，把一些耗时的任务放在线程中即可。

   IPC不是Andrord中所独有的，任何一个操作系统都需要有相应的IPC机制，比如Windows上可以通过剪贴板、管道和邮槽等来进行进程间通信，Linux上可以通过命名管道、共享内容、信号量等来进行进程间通信。在Android中最有特色的进程间通信方式就是Binder，通过Binder可以轻松地实现进程间通信。除了Binder，Android还支持Socket，通过Socket也可以实现任意两个终端之间的通信，当然同一个设备上的两个进程通过Socket通信自然也是可以的。

   说到IPC的使用场景就必须提到多进程，只有面对多进程这种场景下，才需要考虑进程间通信。多进程的情况分为两种，第一种情况是一个应用因为某些原因自身需要采用多进程模式来实现，至于原因，可能有很多，比如有些模块由于特殊原因需要运行在单独的进程中，又或者为了加大一个应用可使用的内存所以需要通过多进程来获取多份内存空间。Android对单个应用所使用的最大内存做了限制，早期的一些版本可能是16MB，不同设备有不同的大小。另一种情况是当前应用需要向其他应用获取数据，由于是两个应用，所以必须采用跨讲程的方式来获取所需的数据，甚至我们通过系统提供的ContentProvider去查询数据的时候，其实也是一种进程间通信，只不过通信细节被系统内部屏蔽了，我们无法感知而已。


### Android中的多进程模式

#### 开启多进程模式

在Android中使用多进程只有一种方法，那就是给四大组件指定android:process

```java
    <activity android:name=".MainActivity">
        <intent-filter>
            <action android:name="android.intent.action.MAIN" />

            <category android:name="android.intent.category.LAUNCHER" />
        </intent-filter>
    </activity>

    <activity android:name=".BookManagerActivity" />

    <service
        android:name=".BookManagerService"
        android:process=":remoteBook" />
```

#### 多进程模式的运行机制

  使用多进程会造成如下几方面的问题：

   （1）静态成员和单例模式完全失效：因为Android为每个进程都分配一个独立的虚拟机，不同的虚拟机在内存上有不同的地址空间，这就导致在不同的虚拟机中访问同一个类的对象会产生多份副本，在一个进程中操作对象只会影响当前进程，对其他进程不会造成任何影响。

   （2）线程同步机制完全失效：因为操作的不是同一块内存，不管是锁对象还是锁全局类都无法保证线程同步，因为不同进程锁的不是同一个对象。

   （3）SharedPreferences可靠性下降：因为SharedPreferences不支持两个进程同时去执行写操作，否则会导致一定几率的数据丢失，这是因为SharedPreferences底层是通过读/写XML文件来实现的，并发写显然是可能出问题的，甚至并发读/写都有可能出问题。

   （4）Application会多次创建：因为运行在同一个进程中的组件是属于同一个虚拟机和同一个Application的，同理，运行在不同进程中的组件是属于两个不同的虚拟机和Application的。


### Binder

   直观来说，Binder是Android中的一个类，它实现了IBinder接口。从IPC角度来说，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，该通信方式在Linux中没有；从Android Framework角度来说，Binder是ServiceManager连接各种Manager(ActivityManager、WindowManager等等）和相应Managerservice的桥梁；从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。

```java
public interface IBookManager extends android.os.IInterface {
    /**
     * Default implementation for IBookManager.
     */
    public static class Default implements com.example.aidltest.IBookManager {
        @Override
        public java.util.List<com.example.aidltest.Book> getBookList() throws android.os.RemoteException {
            return null;
        }

        @Override
        public void addBook(com.example.aidltest.Book book) throws android.os.RemoteException {
        }

        @Override
        public android.os.IBinder asBinder() {
            return null;
        }
    }

    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.example.aidltest.IBookManager {
        private static final java.lang.String DESCRIPTOR = "com.example.aidltest.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.example.aidltest.IBookManager interface,
         * generating a proxy if needed.
         */
        public static com.example.aidltest.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.example.aidltest.IBookManager))) {
                return ((com.example.aidltest.IBookManager) iin);
            }
            return new com.example.aidltest.IBookManager.Stub.Proxy(obj);
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
                case TRANSACTION_getBookList: {
                    data.enforceInterface(descriptor);
                    java.util.List<com.example.aidltest.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(descriptor);
                    com.example.aidltest.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = com.example.aidltest.Book.CREATOR.createFromParcel(data);
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

        private static class Proxy implements com.example.aidltest.IBookManager {
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
            public java.util.List<com.example.aidltest.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<com.example.aidltest.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    boolean _status = mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    if (!_status && getDefaultImpl() != null) {
                        return getDefaultImpl().getBookList();
                    }
                    _reply.readException();
                    _result = _reply.createTypedArrayList(com.example.aidltest.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public void addBook(com.example.aidltest.Book book) throws android.os.RemoteException {
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
                    boolean _status = mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    if (!_status && getDefaultImpl() != null) {
                        getDefaultImpl().addBook(book);
                        return;
                    }
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }

            public static com.example.aidltest.IBookManager sDefaultImpl;
        }

        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);

        public static boolean setDefaultImpl(com.example.aidltest.IBookManager impl) {
            if (Stub.Proxy.sDefaultImpl == null && impl != null) {
                Stub.Proxy.sDefaultImpl = impl;
                return true;
            }
            return false;
        }

        public static com.example.aidltest.IBookManager getDefaultImpl() {
            return Stub.Proxy.sDefaultImpl;
        }
    }

    public java.util.List<com.example.aidltest.Book> getBookList() throws android.os.RemoteException;

    public void addBook(com.example.aidltest.Book book) throws android.os.RemoteException;
}
```

   这个类的结构其实很简单，首先，它声明了两个方法getBookList和addBook，显然这就是我们在IBookManager.aidl中所声明的方法，同时它还声明了两个整型的id分别用于标识这两个方法，这两个id用于标识在transact过程中客户端所请求的到底是哪个方法。接着，它声明了一个内部类Stub，这个Stub就是一个Binder类，当客户端和服务端都位于同一个进程时，方法调用不会走跨进程的transact过程，而当两者位于不同进程时，方法调用需要走transact过程，这个逻辑由Stub的内部代理类Proxy来完成。

* **DESCRIPTOR**

  Binder的唯一标识，一般用当前Binder的类名表示

* **asInterface(android.os.IBinder obj)**

  用于将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象，这种转换过程是区分进程的，如果客户端和服务端位于同一进程，那么此方法返回的就是服务端的Stub对象本身，否则返回的是系统封装后的Stub.proxy对象  

* **asBinder**

  此方法用于返回当前Binder对象

* **onTransact**

  这个方法运行在服务端的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。该方法的原型为public Boolean onTransact (int code,android.os.Pareel data,android.os.Pareel reply,int flags)，服务端通过code可以确定客户端所请求的目标方法是什么，接着从data中取出目标方法所需要的参数（如果目标方法中有参数的话），然后执行目标方法，当目标方法执行完毕后，就向reply中写入返回值（如果目标方法有返回值的话），onTransact方法的执行过程就是这样的。需要注意的是，如果此方法返回false,那么客户端的请求会失败，因此我们可以利用这个特性来做权限验证，毕竟我们也不希望随便一个进程都能远程调用我们的服务。


### Android中的IPC方式

#### 使用Bundle

   Activity、Service、Broadcast Receiver三大组件都支持在Intent中传递Bundle数据，由于Bundle实现了Parcelable接口，所以它可以在不同的进程间传输。基于这一点，当我们在一个进程中使用另外一个进程的Activity、Service、Broadcast Receiver，我们就可以在Bundle中附加我们需要传输给远程进程的信息，然后用Intent发送出去。当然，我们传输的数据必须能够序列化，比如基本数据类型，实现了Parcelable接口的对象，实现了Serializable接口的对象以及一些Android支持的特殊对象。

#### 使用文件共享

   文件共享是一种不错的进程间通信的方式，两个进程通过读/写同一个文件来交换数据，比如A进程把数据写入文件，B进程通过读取这个文件来获取数据。由于Android系统基于Linux，使得其并发读写文件可以没有限制地进行，甚至两个线程同时对同一个文件进行写操作都是允许的。通过文件交换数据很好使用，除了可以交换一些文本信息外，我们还可以序列化一个对象到文件系统，在另一个进程中恢复这个对象。文件共享方式适合在对数据同步要求不高的进程之间进行通信，并且要妥善处理并发读/写的问题。

#### 使用ContentProvider

   ContentProvider是Android中提供的专门用来在不同应用之间数据共享的方式，ContentProvider的底层实现同样也是Binder。系统预置了许多ContentProvider，比如通讯录信息、日程表信息等，要跨进程访问这些信息，只需要通过ContentResolver的query、update、insert和delete方法即可。

#### 使用Socket

   Socket也叫做套接字，是网络通信中的概念，它分为流式套接字和用户数据报套接字两种，分别是应用于网络的传输控制层中的TCP和UDP协议，TCP是面向连接的协议，提供稳定的双向通讯功能，TCP连接的建立需要经过“三次握手”才能完成，为了提供稳定的数据传输功能，其本身提供了超时重传机制，因此具有很高的稳定性；而UDP是无连接的，提供不稳定的单向通信功能，当然UDP也可以实现双向通信功能。在性能上，UDP具有更好的效率，其缺点是不保证数据一定能够正确传输，尤其是在网络拥塞的情况下。

#### 使用Messenger

   Messenger可以在不同进程中传递Message对象，在Message中放入我们需要传递的数据，就可以轻松地实现数据的进程间传递了。Messenger是一种轻量级的IPC方案，它的底层实现是AIDL。
 
* **服务端进程**
  
   首先，我们需要在服务端创建一个Service来处理客户端的连接请求，同时创建一个Handler并通过它来创建一个Messenger对象，然后在Service的onBind中返回这个Messenger对象底层的Binder即可。

```java
public class MessengerService extends Service {
    private static final String TAG = "MessengerService";

    private Messenger messenger = new Messenger(new MessengerHandler());

    private static class MessengerHandler extends Handler {
        @Override
        public void handleMessage(@NonNull Message msg) {
            switch (msg.what) {
                case 0:
                    Log.d(TAG, "服务端收到客户端的消息：" + msg.getData().getString("sendMsg"));

                    Messenger replyMessenger = msg.replyTo;
                    Message replyMessage = Message.obtain(null, 1);
                    Bundle bundle = new Bundle();
                    bundle.putString("reply", "服务端收到你的消息了，回复给你");
                    replyMessage.setData(bundle);

                    try {
                        replyMessenger.send(replyMessage);
                    } catch (RemoteException e) {
                        e.printStackTrace();
                    }
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return messenger.getBinder();
    }
}
```

* **客户端进程**

   首先要绑定服务端的Service，绑定成功后用服务端返回的IBinder对象创建一个Messenger，通过这个Messenger就可以向服务端发送消息了，发消息类型为Message对象。如果需要服务端能够回应客户端，就和服务端一样，我们还需要创建一个Handler并创建一个新的Messenger，并把这个Messenger对象通过Message的replyTo参数传递给服务端，服务端通过这个replyTo参数就可以回应客户端。

```java
public class MessengerActivity extends AppCompatActivity {
    private static final String TAG = "MessengerActivity";

    private Messenger replyMessenger = new Messenger(new ReplyMessengerHandler());

    private static class ReplyMessengerHandler extends Handler {
        @Override
        public void handleMessage(@NonNull Message msg) {
            switch (msg.what) {
                case 1:
                    Log.d(TAG, "客户端收到的回复：" + msg.getData().getString("reply"));
                    break;
                default:
                    super.handleMessage(msg);
            }
        }
    }

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            Messenger messenger = new Messenger(service);
            Message sendMessage = Message.obtain(null, 0);
            Bundle bundle = new Bundle();
            bundle.putString("sendMsg", "我给你发消息了");
            sendMessage.setData(bundle);
            sendMessage.replyTo = replyMessenger;
            try {
                messenger.send(sendMessage);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger);

        Intent intent = new Intent(this, MessengerService.class);
        bindService(intent, mConnection, BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(mConnection);
    }
}
```

#### 使用AIDL 
AIDL文件支持的数据类型：

（1）基本数据类型（int、long、char、boolean、double等）；

（2）Sting和CharSequence；

（3）List：只支持ArrayList，里面每个元素都必须能够被AIDL支持；

（4）Map：只支持HashMap，里面的每个元素都必须被AIDL支持，包括key和value；

（5）Parcelable：所有实现了Parcelable接口的对象；

（6）AIDL：所有的AIDL接口本身也可以在AIDL文件中使用

* **创建AIDL接口
  
```java
// IBookManager.aidl
package com.example.aidltest;
import com.example.aidltest.Book;

interface IBookManager {
     List<Book> getBookList();
     void addBook(in Book book);
}
```

```java
// Book.aidl
package com.example.aidltest;

parcelable Book;
```

```java
public class Book implements Parcelable {
    public int bookId;
    public String bookName;

    public Book(int bookId, String bookName) {
        this.bookId = bookId;
        this.bookName = bookName;
    }

    protected Book(Parcel in) {
        bookId = in.readInt();
        bookName = in.readString();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel dest, int flags) {
        dest.writeInt(bookId);
        dest.writeString(bookName);
    }
}
```



* **服务端**
  
   服务端首先要创建一个Service用来监听客户端的连接请求，然后创建一个AIDL文件，将暴露给客户端的接口在这个AIDL文件中声明，最后在Service中实现这个AIDL接口即可。

```java
public class BookManagerService extends Service {
    private static final String TAG = "BookManagerService";

    private List<Book> mBookList = new ArrayList<>();

    private Binder mBinder = new IBookManager.Stub() {
        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            Log.d(TAG, "客户端添加了新书:" + book.bookName);
            mBookList.add(book);
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
        mBookList.add(new Book(1, "Android"));
        mBookList.add(new Book(2, "IOS"));
    }

    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }
}
```

* **客户端**

   客户端所要做事情就稍微简单一些，首先需要绑定服务端的Service，绑定成功后，将服务端返回的Binder对象转成AIDL接口所属的类型，接着就可以调用AIDL中的方法了。
 
```java
public class BookManagerActivity extends AppCompatActivity {
    private static final String TAG = "BookManagerActivity";

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            IBookManager bookManager = IBookManager.Stub.asInterface(service);
            try {
                List<Book> list = bookManager.getBookList();
                Log.d(TAG, "旧书列表: ");
                for (int i = 0; i < list.size(); i++) {
                    Log.d(TAG, list.get(i).bookName);
                }

                Book newBook = new Book(3, "开发艺术探索");
                bookManager.addBook(newBook);

                List<Book> newList = bookManager.getBookList();
                Log.d(TAG, "新书列表 : ");
                for (int i = 0; i < newList.size(); i++) {
                    Log.d(TAG, newList.get(i).bookName);
                }
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_aidl);

        Intent intent = new Intent(this, BookManagerService.class);
        bindService(intent, mConnection, BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(mConnection);
    }
}
```
### 选用合适的IPC方式

| 名称 | 优点 | 缺点 | 适用场景 |
| :----: | :----: | :----: | :----: |
| Bundle | 简单易用 | 只能传输Bundle支持的数据类型 | 四大组件间的进程间通信 |
| 文件共享 | 简单易用 | 不适合高并发场景，并且无法做到进程间的即时通信 | 无并发访问情形，交换简单的数据实时性不高的场景 |
| ContentProvider | 在数据源访问方面功能强大，支持一对多并发数据共享 | 可理解为受约束的AIDL | 一对多的进程间数据共享 |
| Socket | 功能强大，可以通过网络传输字节流，支持一对多并发实时通信 | 实现细节稍微有点繁琐 | 网络数据交换 |
| Messenger | 功能一般，支持一对多串行通信和实时通信 | 不能很好地处理高并发，数据通过Message传输，因此只能传输Bundle支持的数据类型 | 低并发的一对多即时通信 |
| AIDL | 功能强大，支持一对多并发通信和实时通信 | 使用稍复杂，需要处理好线程同步 | 一对多实时通信 |