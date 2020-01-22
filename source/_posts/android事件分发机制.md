---
title: android事件分发机制
date: 2020-01-10 14:36:31
tags:
---

# View的事件分发机制

## 点击事件的传递规则

点击事件的分发，其实就是对MotionEvent事件的分发过程，当一个MotionEvent点击产生以后，系统需要把这个事件传递给一个具体的view，这个传递的过程就是分发过程。点击事件的分发过程由三个方法来共同完成：dispatchTouchEvent、onInterceptTouchEvent，onTouchEvent。

~~~
public boolean dispatchTouchEvent(MotionEvent ev)

用来进行事件的分发。如果事件能够传递给当前View，那么此方法一定会被调用，返回结果受当前View的onTouchEvent和下级View的dispatchTouchEvent方法的影响，表示是否分发当前事件。
~~~

~~~
public boolean onInterceptTouchEvent(MotionEvent event)

在dispatchTouchEvent方法内部调用，如果当前View拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，返回结果表示是否拦截当前事件。
~~~

~~~
public boolean onTouchEvent（MotionEvent event）

在dispatchTouchEvent方法中调用，用来处理点击事件，返回结果表示是否消耗当前事件，如果不消耗，则在同一个事件序列当中，当前View无法再接收到事件。
~~~

上述三个方法的关系可以用如下伪代码表示：

~~~
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean consume = false;
    if (onInterceptTouchEvent(ev)) {
        consume = onTouchEvent(ev);
    } else {
        consume = child.dispatchTouchEvent(ev);
    }
    return consume;
}
~~~

通过上述伪代码，我们可以大致了解点击事件的传递规则：对于一个根ViewGroup来说，点击事件产生后，首先会传递给它，这时它的dispatchTouchEvent会被调用，如果这个ViewGroup的onInterceptTouchEvent方法返回true就表示它要拦截当前事件，接着事件就会交给这个ViewGroup处理，即它的onTouchEvent方法就会被调用；如果这个ViewGroup的onInterceptTouchEvent方法返回false就表示它不拦截当前事件，这时当前事件就会继续传递给它的子元素，接着子元素的dispatchTouchEvent方法就会被调用，如此反复直到事件被处理。

当一个View需要处理事件时，如果它设置了OnTouchListener，那么OnTouchListener中的onTouch方法会被回调。这时事件处理要看onTouch的返回值，如果返回false，则当前View的onTouchEvent方法会被调用，如果返回true，那么onTouchEvent方法不会被调用。由此就可见，给View设置的OnTouchListener优先级比onTouchEvent高，在onTouchEvent中，如果有设置OnClickListener，其优先级最低，处于事件传递的尾端。

当一个点击事件产生后，它的传递过程遵循如下顺序：Activity->Window->View,即事件先传递给Activity，Activity再传递给Window，最后Window再传递给顶级View。顶级View接收到事件后，就会按照事件分发机制去分发事件。如果一个View的onTouchEvent返回false，那么它父容器的onTouchEvent会被调用，以此类推。如果所有的元素都不处理这个事件，那么这个事件将会最终传递给Activity处理，即Activity的onTouchEvent方法会被调用。

View没有onInterceptTouchEvent方法，一旦有点击事件传递给它，它的onTouchEvent方法就会被调用。



