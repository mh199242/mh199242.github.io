---
title: Android开发艺术探索第一章
date: 2020-06-01 22:28:31
tags:
---

## Activity的生命周期和启动模式

### Activity的生命周期

#### 典型情况下的生命周期

   * onCreate：表示Activity正在被创建，这是生命周期的第一个方法，在这个方法中，我们可以做一些初始化的工作，比如调用setContentView去加载界面布局资源，初始化Activity所需数据等
   
   * onRestart：表示Activity正在重新启动，一般情况下，当当前Activity从不可见重新变为可见时，onRestart就会被调用，这种情况一般是用户行为所导致的，比如用户按home键切换到桌面或者用户打开了一个新的Activity，这时当前的Activity就会被暂停，也就是onPause和onStop方法被执行了，接着用户又回到了这个Activity，就会出现这种情况
   
   * onStart：表示Activity正在被启动，即将开始，这时Activity已经可见了，但是还没有出现在前台，还无法和用户交互，这个时候我们可以理解为Activity已经启动了，但是我们还看不到
   
   * onResume：表示Activity已经可见了，并且出现在前台并开始活动，要注意这个和onStart的对比，这两个都表示Activity已经可见了，但是onStart的时候Activity还处于后台，onResume的时候Activity才显示到前台
   
   * onPause：表示Activity正在停止，正常情况下，紧接着onStop就会被调用，在特殊情况下，如果这个时候再快速的回到当前Activity，那么onResume就会被调用。此时可以做一些数据存储、停止动画等工作，但是注意不要太耗时了，因为这会影响到新的Activity的显示，onPause必须先执行完，新Activity的onResume才会执行
   
   * onStop：表示Activity即将停止，可以做一些稍微重量级的回收工作，同样不能太耗时
   
   * onDestroy：表示Activity即将被销毁，这是Activity生命周期的最后一个回调，在这里我们可以做一些回收工作和最终的资源释放

#### 异常情况下的生命周期

   * **情况1：资源相关的系统配置发生改变导致Activity被杀死并重新创建**
      
     当系统配置发生改变的时候，Activity会被销毁，其onPause，onStop，onDestroy均会被调用，同时由于Activity是异常情况下终止的，系统会调用onSaveInstanceState来保存当前Activity的状态，这个方法调用的时机是在onStop之前，它和onPause没有既定的时序关系，它既可能在onPause之前调用，也有可能在之后调用，需要强调的是，这个方法只出现在Activity被异常终止的情况下，正常情况下系统不会回调这个方法。当Activity被重建后，系统会调用onRestoreInstanceState，并且把Activity销毁时onSaveInstanceState保存的Bundle对象作为参数传递给onRestoreInstanceState和onCreate方法，因此我们可以通过onRestoreInstanceState和onCreate方法来判断Activity是否被重建。如果被重建了，我们就取出之前的数据并恢复，从时序上来说，onRestoreInstanceState的调用时机在onStart之后。

     同时我们要知道，在onSaveInstanceState和onRestoreInstanceState方法中，系统自动为我们做了一些恢复工作，当Activity在异常情况下需要重新创建时，系统会默认为我们保存当前的Activity视图结构，并且在Activity重启后为我们恢复这些数据，比如文本框中用户输入的数据，ListView滚动的位置，这些View相关的状态系统都会默认恢复，具体针对某一个特定的View系统能为我们恢复哪些数据，我们可以查看View的源码，和Activity一样，每一个View都有onSaveInstanceState和onRestoreInstanceState这两个方法，看一下它们的实现，就能知道系统能够为每一个View恢复哪些数据。

     关于保存和恢复View的层次结构，系统的工作流程是这样的：首先Activity被意外终止时，Activity会调用onSaveInstanceState去保存数据，然后Activity会委托Window去保存数据，接着Window再委托DecorView保存数据，DecorView是一个ViewGroup，最后DecorView再去一一通知它的子元素来保存数据，这样整个数据保存过程就完成了。可以发现，这是一种典型的委托思想，上层委托下层、父容器委托子容器去处理一件事情。

   * **情况2：资源内存不足导致低优先级的Activity被杀死**
    （1）前台Activity：正在和用户交互的Activity，优先级最高
    （2）可见但非前台Activity：比如弹出对话框，导致Activity可见但是位于后台无法和用户直接交互
    （3）后台Activity：已经被暂停的Activity，比如执行了onStop，优先级最低

### Activity的启动模式

#### Activity的LaunchMode

   * **standard**  
      标准模式，这也是系统的默认模式，每次启动一个Activity都会重新创建一个实例

   * **singleTop**  
      栈顶复用模式，在这个模式下，如果新的Activity已经位于任务栈的栈顶，那么此Activity不会被重新创建，同时它的onNewIntent方法会被调用

   * **singleTask**  
      栈内复用模式，这是一种单实例模式，在这种模式下，只要Activity在一个栈内存在，那么多次启动此Activity都不会创建实例，和singTop一样，系统也会回调其onNewIntent方法,并且它之上的Activity会出栈

   * **singleInstance**  
      单实例模式，这是一种加强的singleTask的模式，它除了具有singleTask的所有属性之外，还加强了一点，那就是具有此模式下的Activity只能单独的处于一个任务栈中

#### Activity的Flags
  
   * **FLAG_ACTIVITY_NEW_TASK**
      这个标志位的作用是为Activity指定“singleTask”启动模式，其效果和XML中指定该模式相同

   * **FLAG_ACTIVITY_SINGLE_TOP**
      这个标志位的作用是为Activity指定“singleTop”启动模式，其效果和XML中指定该模式相同

   * **FLAG_ACTIVITY_CLEAR_TOP**
      具有此标记位的Activity,当它启动时，在同一个任务栈中所有位于它上面的Activity都要出栈，这个模式一般需要和FLAG_ACTIVITY_NEW_TASK配合使用，在这种情况下，被启动的Activity的实例如果已经存在，那么系统就会调用它的onNewIntent,如果被启动的Activity采用标准模式，那么它连同它之上的Activity都要出栈，系统会创建新的Activity实例并放入栈顶

### IntentFilter的匹配规则
  
   启动Activity分为两种，显示调用和隐式调用，显示调用需要明确的指定被启动对象的组件信息，包括包名和类名，而隐式意图则不需要明确指定调用信息，原则上一个intent不应该既是显式调用又是隐式调用，如果二者共存的话以显式调用为主，显式调用很简单，这里主要介绍隐式调用，隐式调用需要intent能够匹配目标组件的IntentFilter中所设置的过滤信息，如果不匹配将无法启动目标Activity，IntentFilter中的过滤信息有action、category、data，下面是一个过滤规则的实例： 
    
    
        <activity
            android:name=".ui.activity.SplashActivity"
            android:theme="@style/ThemeSplash">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
            <intent-filter>
                <action android:name="android.intent.action.VIEW" />

                <category android:name="android.intent.category.DEFAULT" />
                <category android:name="android.intent.category.BROWSABLE" />

                <data
                    android:host="agent"
                    android:scheme="hrtagent" />
            </intent-filter>
        </activity>
    

   为了匹配过滤列表，需要同时匹配过滤列表中的action、category、data信息，否则匹配失败，一个过滤列表中的action、category、data可以有多个，所有的action、category、data分别构成不同类别，同一类型的信息共同约束当前类别的匹配过程，只有一个intent同时匹配action类别、category类别、data类别才算完全匹配，只有完全匹配才能成功启动目标Activity，另外一点，一个Activity中可以有多个intent-filter，一个intent只要能匹配一组intent-filter即可成功启动Activity


#### action的匹配规则
   
   action是一个字符串，系统预定了一些action，同时我们也可以在应用中定义自己的action，action的匹配规则是Intent中的action必须能够和过滤规则中的action匹配，这里说的匹配是指action的字符串值完全一样，一个过滤规则中可以有多个action，只要Intent中的action能够和过滤规则中的任何一个action相同即可匹配成功。针对上面的过滤规则，需要注意的是，Intent如果没有指定action，那么匹配失败。总结一下，action的匹配要求Intent中的action存在且必须和过滤规则中的其中一个action相同，action区分大小写，大小写不同字符串相同的action会匹配失败


#### category的匹配规则
   
   category是一个字符串，系统预定义了一些category，同时我们也可以在应用中定义自己的category。category的匹配规则和action不同，它要求Intent中如果含有category，那么所有的category都必须和过滤规则中的其中一个category相同。换句话说，Intent如果出现了category，不管有几个category，对于每个category来说，它必须是过滤规则中已经定义的category。当然，Intent中可以没有category，如果没有category的话，按照上面的描述，这个Intent仍然可以匹配成功。这里要注意下它和action匹配过程的不同，action
   是要求Intent中必须有一个action且必须能够和过滤规则中的某个action相同，而category要求Intent可以没有category，但是如果你一旦有category，不管有几个，每个都要能和过滤规则中的任何一个category相同。

#### data的匹配规则
   
   data的匹配规则和action类似，它也要求Intent中必须含有data数据，并且data数据能够完全匹配过滤规则中的某一个data。这里的完全匹配是指过滤规则中出现的data部分也出现在了Intent
   中的data中。data语法如下：
    
    
    <data
       android:scheme="string"
       android:host="string"
       android:port="string"
       android:path="string"
       android:pathPattern="string"
       android:pathPrefix="string"
       android:mimeType="string" />
    
   data由两部分组成，mimeType和URI，前者是媒体类型，比如image/jpeg等，可以表示图片等，URI的结构如下：

    <scheme>://<host>:<port>/[<path>|<pathPrefix>|<pathPattern>]
    http://www.baidu.com:80/search/info

* Scheme：URI的模式，比如http、file、content等，如果URI中没有指定的scheme，那么整个URI的其他参数无效，这也意味着URI无效

* Host：URI的主机，比如www.google.com，如果host未指定，那么整个URI中的其他参数无效，这也意味着URI无效
  
* Port：URI中的端口号，比如80，不过需要指定上面两个才有意义
  
* Path、pathPattern和pathPrefix：这三个参数表述路径信息，其中path表示完整的路径信息；pathPattern也表示完整的路径信息，但是它里面可以包含通配符“ * ”，“ * ” 表示0个或多个任意字符；pathPrefix表示路径的前缀信息。
