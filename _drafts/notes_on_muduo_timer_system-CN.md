---
layout: post
title: Muduo定时器的实现
categories: c++ net
---
我在开发的过程中不时会需要用到定时器这样的功能。
比如之前在做网络库的测试时，我编写客户端模拟大量并发连接，希望每个Session每隔0.5s发送一条消息。
而我没有什么合适的方法来做这件事。  
我首先想到了`sleep`。
但这种方法造成定时器延时太大，而且显然不可能为每个session都调用sleep，这样就失去了大量并发模拟的意义。
另一种方法是采用`busy loop`这种忙等待的方式，然后不断判断是否超时。
这种方法又太浪费CPU，对CPU消耗太大。
直到我看到了**Muduo**网络库中定时器的实现，我学会了一种更好的实现定时器的方法。  
[Muduo](https://github.com/chenshuo/muduo)是由陈硕编写的一个专门用于Linux系统的网络库。
这里说明一下，本篇说的定时器也是基于Linux系统实现的，其他平台的这里不讨论。
你或许会奇怪为什么做个定时器扯到网络库去了？
我见过许多网络库都会提供定时器的功能，比如**asio**也有这样的功能。
而且我之所以想好好整理一下这个定时器的实现也是因为它将网络库的功能和定时器巧妙地整合到了一起。

## 什么是定时器？
说到这里读者应该也明白我说的定时器是个什么玩意儿了。
这个东西当然也没有什么明确的定义，只能用通俗的话来讲。
定时器就是个可以让我们在未来某个时间做某件事情的东西。
我们用定时器来做到诸如“在2s之后执行一个函数”或者“每隔2s执行一个函数”之类的事情。
嚯，这正是我之前测网络库时模拟并发连接需要的东西。

## 如何使用Muduo的定时器？
首先我们需要了解关于**Muduo**网络库的一个很重要的概念——**One loop per thread**，作者陈硕也在他的著作中不断提到这一点。
在**Muduo**中每一个线程都和一个事件循环绑定，不断地去跑这个事件循环。
当然了，我们要定时去做的事情肯定也是在某一个线程中完成的。
**Muduo**的一个核心类是`EventLoop`，就是上面提到的事件循环。
当调用了`EventLoop`的`loop`方法后它就会开始不断循环处理事件，直到我们调用`quit`退出它。
而我们要用的定时器接口也是由`EventLoop`提供的，它为我们提供了3个添加`Timer`和一个取消`Timer`的方法：
``` c++
    ///
    /// Runs callback at 'time'.
    /// Safe to call from other threads.
    ///
    TimerId runAt(const Timestamp& time, const TimerCallback& cb);
    ///
    /// Runs callback after @c delay seconds.
    /// Safe to call from other threads.
    ///
    TimerId runAfter(double delay, const TimerCallback& cb);
    ///
    /// Runs callback every @c interval seconds.
    /// Safe to call from other threads.
    ///
    TimerId runEvery(double interval, const TimerCallback& cb);
    ///
    /// Cancels the timer.
    /// Safe to call from other threads.
    ///
    void cancel(TimerId timerId);
```
似乎没有必要一一解释每个方法的用途了，名字和注释已经很清楚了。
不过有些东西还是要解释一下：
*   `Timer`： 这其实是一个结构体，包含了一个*超时时间*和一个*回调函数对象*。
    当超时了之后，就该去调用那个回调函数。
*   `TimerId`： 是一个`Timer`的标识，用于取消某个`Timer`。
*   `Timestamp`： 时间戳，本质是从Epoch时间到现在的微秒数。
    是的，这里可以看出这个定时器最多精确到微秒。
*   `TimerCallback`： 即我们要回调的回调函数。

## 定时器实现的原理
现在我们来看最关键最有趣的部分，我们的回调函数是如何在我们设定的时间到了之后被回调的呢？
接下来我们就来看从一个`Timer`被添加，一直到它的回调函数被调用的过程中发生了一些什么。
首先我需要先介绍一下**Muduo**定时器中的核心类：`TimerQueue`。
当然它对用户是不可见的。
它的一些成员变量如下所示：
``` c++
  // FIXME: use unique_ptr<Timer> instead of raw pointers.
  // This requires heterogeneous comparison lookup (N3465) from C++14
  // so that we can find an T* in a set<unique_ptr<T>>.
  typedef std::pair<Timestamp, Timer*> Entry;
  typedef std::set<Entry> TimerList;
  typedef std::pair<Timer*, int64_t> ActiveTimer;
  typedef std::set<ActiveTimer> ActiveTimerSet;

  void addTimerInLoop(Timer* timer);
  void cancelInLoop(TimerId timerId);
  // called when timerfd alarms
  void handleRead();
  // move out all expired timers
  std::vector<Entry> getExpired(Timestamp now);
  void reset(const std::vector<Entry>& expired, Timestamp now);

  bool insert(Timer* timer);

  EventLoop* loop_;
  const int timerfd_;
  Channel timerfdChannel_;
  // Timer list sorted by expiration
  TimerList timers_;

  // for cancel()
  ActiveTimerSet activeTimers_;
  bool callingExpiredTimers_; /* atomic */
  ActiveTimerSet cancelingTimers_;

```
这里我不一一详细解释每个成员的作用了。
将代码放在这里是因为之后可能会提到里面的一些成员变量。
接下来我们来看整个过程。
其实我们要解决的最核心的问题无非就是如何在超时的时候收到通知。
### 如何在超时的时候收到通知？
**Muduo**采用的方法和我在开头提及的两种方法大不相同。
我以前的做法显然很low很不专业，只能说是学生的做法。
现在我们应该更加专业了。
**Muduo**使用的是`timerfd`配合`poll`来获取侦测超时这一事件。
这里的`poll`是个统称，包括`select`，`poll`，`epoll`。
这些都是有Linux系统提供的。
如果对`poll`不太了解的话，需要先去了解一下这些和I/O多路复用。
然而`timerfd`对我而言也是一个新的事物，
`timerfd`是一个特殊的文件描述符，它会在我们设定的时间到了之后变为**readable**的状态。
这是由系统完成的。
哇哦，这样的话我们不就能把超时这件事当成一个普通网络事件了吗。
然后复用网络处理的模型来处理超时事件。
啥意思呢？就是说本来我们可能会用`poll`这样的方式来处理大量网络I/O事件。
现在有了`timerfd`，我们可以把我们设定的超时也看成是一个普通的网络事件。
然后套用原来I/O多路复用的模式去处理。
因此我觉得这种方式很好，或许也正是基于这个原因，Linux系统才会提供`timerfd`这样的东西吧。
### 如何添加一个Timer？
现在我们来看看添加Timer是如何具体实现的。
1.   初始化的时候通过调用系统函数`timerfd_create`来创建一个`timerfd`。
对于每一个线程只需要一个`timerfd`就够了，create也只需要做一次。
但是我们会在之后多次设置这个fd。
2.   将这个新的timer插入一个数据结构中，在这里的话就是`timers_`这个成员。
注意到`TimerList`和`Entry`的定义。
`TimerList`是一个`Entry`的set。
`Entry`是一个`Timestamp`和一个`Timer`的指针。
我们知道`std::set`是一个用红黑树实现的有序容器。
这样做是因为需要按照超时的时间费降序，然后可以每次根据当前的时间做一次二分搜索，取出所有超时的`Timer`。
是的，这不是最好的实现方式，每次取出一个`Timer`的开销都是**O(logN)**，作者自己也这样说。
最好的实现方法是使用堆，libevent中使用的是四叉堆。
但是标准库中提供的堆并提供删除的操作。
当然，使用平衡树的性能也已经足够了。
最后一点要注意，两个timer的timestamp相同也是有可能的，因此我们不能单独只用一个timestamp来进行增删操作。
因此将`Entry`这样定义，这保证了我们不会产生timer之间的冲突。
3.   可能需要重新设置timer_fd。
为了使超时尽可能精准，我们当然希望在最早的timer超时的时候收到反应。
这也就是说，如果新插入的timer时间最早的情况下，我们需要重新设置timer_fd。
我们通过调用`timerfd_settime`来重新设置fd的超时时间，
 
### 如何在timer超时的时候调用其回调函数？
熟悉I/O多路复用和poll的用法朋友几乎可以跳过这一段了。
之前也已经提到了，现在timerfd也就是被当作一个普通的fd。
所有的做法都和原来的用法一模一样。
我们还是来看一下，顺便了解一下**Muduo**的模型。
**Muduo**是非常典型的`Reactor`模式。
`EventLoop`的每一次循环都做了下面这些事：
![img](https://raw.githubusercontent.com/Irving-cl/Irving-cl.github.io/master/_assets/notes_on_muduo_timer_system/muduo_reactor.png)
1.   一上来poller类会轮询所有的fd(通过调用相应的`poll`或者`epoll`)，得到所有fd上的事件。
然后将每一个fd上的事件传递给相应的`Channel`。
`Channel`可以看成是底层I/O和上层回调之间的一个桥梁。
这个就是专门负责timer_fd的Channel啦。
当超时发生时，Channel就会知道这个事件了。
2.   然后`timerfdChannel_`发现它负责的fd变得可读了，就去调用它的handleRead这个函数。
3.   在这个函数中，所有当前而言超时的Timer都会被删除，当然了，它们的回调函数会被调用。
最后别忘了，如果有timer到期的话，我们需要重新设置timer_fd。

## 关于timer_fd的补充
通过以上的内容，我们已经了解了如何使用Linux系统提供的`timer_fd`来实现定时器。
如果是第一次听说`timer_fd`的小伙伴相信对这个东西印象还比较模糊。
不如趁这个机会最后再讨论一些关于它的事情，顺便也看一下**Muduo**作者的用法。
当然如果要深入了解它的话，还是去看一下它的[man page](http://www.huge-man-linux.net/man2/timerfd_create.html)。  

### timer_fd的精度
精度显然是我们非常关心的问题。
仔细的同学可能注意到上文有提到**Muduo**中的`Timerstamp`是*micro seconds since Epoch*。
但不要认为`timer_fd`的精度就只到微秒了。
我们来看一下`timerfd_settime`中代表超时时间的参数的类型`itimerspec`:  
``` c++
struct timespec {
    time_t tv_sec;                /* Seconds */
    long   tv_nsec;               /* Nanoseconds */
};

struct itimerspec {
    struct timespec it_interval;  /* Interval for periodic timer */
    struct timespec it_value;     /* Initial expiration */
};
```
一目了然，**nanoseconds**，`timer_fd`是可以精确到**纳秒**的。
但是话说回来，我们也很少需要精确到纳秒的情形。
或许也是由于这个原因，**Muduo**的作者觉得精确到纳秒没有必要吧。

### timer_fd使用示例
我们直接来看**Muduo**中是怎么使用的`timer_fd`的吧。  
创建`timer_fd`的过程：
``` c++
int createTimerfd()
{
    int timerfd = ::timerfd_create(CLOCK_MONOTONIC,
                                 TFD_NONBLOCK | TFD_CLOEXEC);
    if (timerfd < 0)
    {
        LOG_SYSFATAL << "Failed in timerfd_create";
    }
    return timerfd;
}
```
其他的不必多说了，解释一下参数里的flags吧：  
`CLOCK_MONOTONIC`代表计时不会因为系统时间改变而被影响，与之相对的是`CLOCK_REALTIME`。
`TFD_NONBLOCK`相信各位都看的明白，非阻塞，别忘了`timer_fd`也是个fd，是要被读的。
`TFD_CLOEXEC`，在调用`exec`后关闭，这也是个fd都会有的flag，一般用来在子进程中关闭这个fd。  
如何设置`timer_fd`：
``` c++
struct timespec howMuchTimeFromNow(Timestamp when)
{
  int64_t microseconds = when.microSecondsSinceEpoch()
                         - Timestamp::now().microSecondsSinceEpoch();
  if (microseconds < 100)
  {
    microseconds = 100;
  }
  struct timespec ts;
  ts.tv_sec = static_cast<time_t>(
      microseconds / Timestamp::kMicroSecondsPerSecond);
  ts.tv_nsec = static_cast<long>(
      (microseconds % Timestamp::kMicroSecondsPerSecond) * 1000);
  return ts;
}

void resetTimerfd(int timerfd, Timestamp expiration)
{
  // wake up loop by timerfd_settime()
  struct itimerspec newValue;
  struct itimerspec oldValue;
  bzero(&newValue, sizeof newValue);
  bzero(&oldValue, sizeof oldValue);
  newValue.it_value = howMuchTimeFromNow(expiration);
  int ret = ::timerfd_settime(timerfd, 0, &newValue, &oldValue);
  if (ret)
  {
    LOG_SYSERR << "timerfd_settime()";
  }
}
```
这段代码好像有点意思。
我们先来看`resetTimerfd`。
其中`newValue`是我们设置的要超时的时间。
而`oldValue`将被设置成当前的超时设置，也就是上一次的设置了。
当然`oldValue`可以被设为**NULL**，如果我们不需要它的话。
现在我想大家应该大致了解**Muduo**定时器的实现了。  
然后来看`howMuchTimeFromNow`。
这个函数其实把**Muduo**中的`Timestamp`转化成Linux需要的参数类型`struct timespec`。
这里有一点需要注意，`ts`的两个成员一个代表秒，一个代表纳秒。
实际上的超时时间应该是**两者的相加**。
这是从以上的代码推测出来的，当然按照常识也是这样了。。。
不过就这点来说，我觉得官方的文档好像说的不够清楚：

最核心的要素还是`timer_fd`和`poll`的配合使用。
有任何问题欢迎和我讨论^_^   
**caoli.irving@shandagames.com**
