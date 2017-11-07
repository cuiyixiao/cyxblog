---
layout: post
title: Notes on Muduo Timer
categories: c++ net
---
Recently I am studying [Muduo](https://github.com/chenshuo/muduo), a net library written by ChenShuo.
And I think the timer system of it is worth learning, so I did some notes on it in this article.

## What is a timer?
*A timer allows us to do something at a specific time later.*  
This is not quoted from someone's words. This is just my own understanding.
Say, now we want to run a function, instead of running it immediately, we want to run it after 2 seconds. And a timer enables us to do something like that.
 
## How do we use timer in Muduo?
At the very beginning, we have to know a very important idea about Muduo: **One loop per thread**, which is also emphasized by the author.
The core class of Muduo is `EventLoop`.
Each `EventLoop` is bind to a thread and will loop forever to run the thread until we call the `quit()` method of it.
Every thing runs in one of the EventLoops in Muduo.
So, if we want to run a function in 2 seconds, we run the function in one of the EventLoops. Apparently, we run our functions in some thread.
Muduo provides us with three methods to add a timer and one to cancel a timer:   
{% highlight c++ %}
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
{% endhighlight %}
It would be unnessary to explain what does every above method do. However, I should still explain something else:
*   A `Timer` is actually an expiration time with a callback function. The callback function would be called when it expired. 
*   A `TimerId` is an identification of a timer so that we can cancel a timer.
*   `Timestamp` is actually micro seconds since epoch.
*   `TimerCallback` is a function object, which we want to run in the future.
I will explain them in detail later.  

## Overview - How does these timer function work?  
Now we come to the critical and interesting part, let's try to figure out how our functions are called when their time expired.
In this section, we will take a look at the whole process, from the moment a timer was added to the moment its callback was called.
And in next section we will dig some details of the implementation.  
Firstly, I need to introduce the core class of the timer system: `TimerQueue`.
It is invisible to the clients.
The private members of it is as follow:  
{% highlight c++ %}
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

{% endhighlight %}
I am not going to elaborate every member here.
I will mention these members later.
Now we see the whole process. 
Actually, the problem is all about how to make our thread waked up when time is up.
#### How to do that?
The way Muduo use to do such thing is quite different from what I once did.
I once used two stupid methods, `sleep` and `busy loop and judge`.
They are bad as far as accuracy is concerned.
Besides, they are quite 'low' and 'unprofessional', right?
Students may do that.
But we need to be  professional.  
Muduo uses `timerfd` with `poll`.
When I refer to `poll`, I mean `select`, `poll`, `epoll`.
They are provided by linux system.
If you are not clear about them, `man` them on a linux system or search the internet.
I assumed you know the basic things about `poll` here.  
However, `timerfd` is something new to me.
A `timerfd` is a special file descriptor which will become readable when the time we set expires.
It is done by the linux system.
Wow, wonderful! This enables us to take an expiration as the same thing with an I/O event.
And we can detect these events with `poll`.
#### How to add a Timer?
1.   At the very beginning, we create a timerfd by calling `timerfd_create` of linux.
     This is done only once.
     But we may set the timerfd many times later.
     We may have a lot of timer, and we want to be notified when the earliest one of them expired.
2.   Insert the timer into the data structure we use to store them, which is `timers_` here.
     As you can see, there is another member `activeTimers_`.
     Unfortunately I don't understand what is it for.
     But it doesn't matter.
     Notice the definition of `TimerList` and `Entry`.
     `TimerList` is a set of `Entry`.
     An `std::set` is an ordered container implemented by RBTree.
     We store entries in it so that they can be sorted by their expiration time, which is represented by a `Timestamp`.
     Considering different timers may have the same timestamp, we make pair a timestamp with a pointer of the timer.
     This ensures us no collisions between timers.
3.   Reset the `timerfd` if necessary.
     This should be done when the latest inserted timer would expire earliest.
     We do this by calling linux function `timerfd_settime`.  
 
#### How to call the callbacks when they expired?
As I mentioned above, a timerfd is treated as a normal fd.
And we use `poll` function to detect that.
By the way, let's take a look at Muduo's thread model.
Muduo is of the `Reactor` Pattern.
Every single loop of `EventLoop` looks like this:  
![img](https://github.com/Irving-cl/Irving-cl.github.io/_assets/notes_on_muduo_timer_system/muduo_reactor.png)  
