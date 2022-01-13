# 无处不在的小土-timerfd与定时任务
定时任务，是网络编程当中经常需要处理的一类任务。比如说，基于长连接的服务，为了确认连接没有中断，通常都会要求客户端每隔一段时间发送一个心跳包给服务器，服务器也会返回一个响应到客户端， 告知彼此都还活着。

为了实现定时任务，我一开始的想法是再创建一个线程，在这个线程里面通过调用 sleep 阻塞一段时间，当 sleep 返回之后， 就通过[上一节](https://gaoyichao.com/Xiaotu/?book=Linux%E4%B8%8B%E7%9A%84%E4%BA%8B%E4%BB%B6%E4%B8%8E%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B&title=eventfd_%E5%94%A4%E9%86%92PollLoop)中介绍的 eventfd 来唤醒 PollLoop 来执行定时任务。 按照这个逻辑应该可以写出满足需求的代码，但它引入了多线程，而我们目前还有考虑并行运算所带来的任何竞争冒险的问题。 那么有没有可能，像处理多个连接那样，通过 PollLoop 的循环框架在单个线程中完成这件事情呢？

本文将要介绍的 timerfd 以及相关的系统调用就可以担此大任。我们将对它们进行封装提供一个计时器，并编写一个简单的 demo。 在[下一篇文章](https://gaoyichao.com/Xiaotu/?book=Linux%E4%B8%8B%E7%9A%84%E4%BA%8B%E4%BB%B6%E4%B8%8E%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B&title=%E6%97%B6%E9%97%B4%E8%BD%AE%E7%9B%98%E4%B8%8E%E8%B6%85%E6%97%B6%E8%BF%9E%E6%8E%A5)我们会将之应用到 echo 服务器上，让它具有超时断开连接的功能。

## 1. timerfd 简介

![](https://gaoyichao.com/Xiaotu/Linux%E4%B8%8B%E7%9A%84%E4%BA%8B%E4%BB%B6%E4%B8%8E%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B/img/man_timerfd_create.png)

参考我们以前写单片机程序时的[计时器](https://gaoyichao.com/Xiaotu/?book=stm32&title=%E8%AE%A1%E6%97%B6%E5%99%A8%E4%B9%8B%E6%97%B6%E5%9F%BA%E5%8D%95%E5%85%83)，要让服务器判断超时主动断开连接的一个比较直接的想法就是，构建一个计时器， 当计时溢出的时候产生一个事件，驱使我们调用超时的回调函数，并在这个函数中通知 PollLoop 循环关闭连接。我们知道，PollLoop 的内核 poll 调用监听的是文件描述符的读写错事件。 既然有人说在 linux 系统中，万物皆可以是文件，那么如果我们能用文件描述符来表示计时器，不就可以接到 PollLoop 的循环框架下了吗。

在 Linux 系统中，有一组以 timerfd 为前缀的调用。它们分别是 timerfd_create、timerfd_settime、timerfd_gettime。 我们可以通过指令`$ man timerfd_create`来查看相关文档，如右图所示。

其中，timerfd_create 用于创建一个计时器对象，并为之提供一个文件描述符，用于通知进程计时事件。它有两个参数，clockid 说明了计时器的类型。它有以下几种选择：

-   **CLOCK_REALTIME**: 这是一种可以设置的系统级实时时钟。
-   **CLOCK_MONOTONIC**: 这是一种不可修改的，单调递增的计时器。
-   **CLOCK_BOOTTIME**: 与 CLOCK_MONOTONIC 类似的，这也是一个不可修改的单调递增的计时器。只是当系统休眠的时候，CLOCK_MONOTONIC 是不会计时的。 而它在这段时间中也会计时。
-   **CLOCK_REALTIME_ALARM**: 功能上与 CLOCK_REALTIME 没有本质区别，只是当系统休眠的时候会唤醒系统。但是这要求调用者具有 CAP_WAKE_ALARM 的能力。
-   **CLOCK_BOOTTIME_ALARM**: 功能上与 CLOCK_BOOTTIME 没有本质区别，只是当系统休眠的时候会唤醒系统。但是这要求调用者具有 CAP_WAKE_ALARM 的能力。

在构建 timerfd 的时候，我们还可以通过参数 flags，来设定计时器的一些特性。目前主要是 TFD_NONBLOCK 用于设定文件描述符工作在非阻塞的状态下。 TFD_CLOEXEC 则用于通过 fork-exec 创建新的进程并运行其它程序时自动关闭子进程中的文件描述符。它们可以通过位或运算进行组合，目前我们不考虑这些特性。

创建了计时器之后，我们需要通过调用 timerfd_settime 来启动或者停止计时。该调用有 4 个参数，如上边右侧的截图所示。其中，fd 是将要操作的计时器的文件描述符； flags 描述了计时特性，这里我们只关注 TFD_TIMER_ABSTIME，表示绝对计时。如果设置了该特性，只有当计时器达到了第三个参数 new_value 中的 it_value 字段所描述的时刻，才认为计时到期。 默认情况下，采用的都是相对计时，即相对于调用 timerfd_settimer 的时刻，经过 new_value.it_value 字段描述的时间后，认为计时到期。

```
        `struct timespec {
           time_t tv_sec;
           long   tv_nsec;
       };

       struct itimerspec {
           struct timespec it_interval;    
           struct timespec it_value;
       };`

```

我们对计时器有两种需求。其一，我们希望有一个闹钟在经过了一段时间，或者到达了某个特殊时刻之后，通知我们去处理一些事务。其二，我们需要周期性的工作，即每过一段时间就去完成一个特定任务。 这两个需求都体现在 timerfd_settime 的第三个参数 new_value 上了。

该参数的数据类型是`struct itimerspec`，其定义如右侧代码所示。 它有两个字段，其中 it_value 表示计时器第一次到期的时刻，如果是相对计时器则经过该字段描述的时间之后，计时器第一次计时到期。若是绝对计时器，则要求计时器到达该时刻。 这就满足了我们的第一个需求。 字段 it_interval 描述的是计时周期，计时器在第一次到期之后，每个该字段描述的一段时间之后，都会产生依次计时到期时间。这满足了我们的第二个需求。

此外，timerfd_settime 还有第四个参数 old_value，用于返回当前计时器的计时设置。如果我们不关心它，可以传递一个 NULL。当然，我们也可以通过调用 timerfd_gettime 来获取计时设置。 通过这三个系统调用，我们就可以创建一个使用文件描述符来表示的计时器，设置和获取计时到期条件。下面，我们对它们进行封装，以融合到我们的 PollLoop 框架下。

```
        `class Timer {
        private:
            PollEventHandlerPtr mEventHandler;
            struct timespec mOriTime;
            int mFd;
        };`

```

## 2. Timer——计时器的封装

为了方便的使用 timerfd，这里我们将它们封装到类 Timer 中。如右侧的代码片段所示，我们为之定义了三个私有的成员。

-   **mEventHandler**是一个[事件分发器](https://gaoyichao.com/Xiaotu/?book=Linux%E4%B8%8B%E7%9A%84%E4%BA%8B%E4%BB%B6%E4%B8%8E%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B&title=reactor%E6%A8%A1%E5%BC%8F%E7%9A%84echo%E6%9C%8D%E5%8A%A1%E5%99%A8#PollEventHandler)， 用于通过 PollLoop 循环来监听文件描述符 mFd 所对应的计时器的到期事件，并调用相应的回调函数 OnReadEvent。 我们将在 Timer 的构造函数中完成该对象的实例化工作，用户还需要通过 ApplyHandlerOnLoop 接口将它注册到一个 PollLoop 循环上。
-   **mFd**是一个通过调用 timerfd_create 获得的文件描述符，它对应一个计时器。当计时到期事件发生的时候，我们都可以通过调用 read(2) 来获取计时溢出的次数。 所以我们所关心的计时到期事件，实质上是文件描述符 mFd 的可读事件。
-   **mOriTime**是一个用于绝对计时的参考时间点。

下面左侧是 Timer 的构造函数，我们首先通过调用 timerfd_create 构建计时器，并用成员 mFd 记录下它的文件名描述符。在断言描述符一定大于零之后，实例化了事件分发器， 并打开对可读事件的监听功能。最后，在注册读事件的回调函数 OnReadEvent。该回调函数的实现如下面右边的代码所示，我们通过调用 read 读取计时溢出次数，将之记录在局部变量 exp 中。 然后调用计时溢出的回调函数。后面我们会看到这个回调函数将由 TcpServer 提供，并在该回调中通过时间轮盘的形式判定网络连接是否超时。

| \`\``
`Timer::Timer() {mFd = timerfd_create(CLOCK_REALTIME, 0);
    assert(mFd > 0);
    mEventHandler = PollEventHandlerPtr(new PollEventHandler(mFd));
    mEventHandler->EnableRead(true);
    mEventHandler->EnableWrite(false);
    mEventHandler->SetReadCallBk(std::bind(&Timer::OnReadEvent, this));
}\`

````

 | ```
`void Timer::OnReadEvent() {
    uint64_t exp;
    ssize_t s = read(mFd, &exp, sizeof(exp));
    if (mTimeOutCb)
        mTimeOutCb();
}`

````

 \|

针对我们刚刚提到的计时器的两种需求，我们在 Timer 中定义了两个接口 RunAfter 和 RunEvery，来分别用于设置单次定时任务和周期定时任务。下面是单次定时任务 RunAfter 的实现片段， 它有两个输入参数。time 表示从调用该函数开始，经历一段时间之后，执行回调函数 cb 中定义的任务。

因为我们采用的是绝对计时器，所以在调用 timerfd_settime 之前，需要先获取当前的时间。在 linux 系统中有调用 clock_gettime 来完成这一任务，我们将当前时间记录在成员变量 mOriTime 中。 作为计时的参考点。

```
        `void Timer::RunAfter(const timespec & time, EventCallBk cb) {
            if (clock_gettime(CLOCK_REALTIME, &mOriTime) == -1) {
                perror("clock_gettime failed!");
                exit(1);
            }`

```

接下来，根据 mOriTime 和输入的计时配置构建 new_value 对象。RunAfter 只执行一次定时任务，所以这里将字段 it_interval 中的秒和纳秒字段都置为 0。

```
        `    struct itimerspec new_value;
            new_value.it_value.tv_sec = mOriTime.tv_sec + time.tv_sec;
            new_value.it_value.tv_nsec = mOriTime.tv_nsec + time.tv_nsec;
            new_value.it_interval.tv_sec = 0;
            new_value.it_interval.tv_nsec = 0;`

```

最后，我们调用 timerfd_settime 设置定时。如果成功完成定时设置，就把输入的回调任务 cb 赋值给 mTimeOutCb。

```
        `    if (timerfd_settime(mFd, TFD_TIMER_ABSTIME, &new_value, NULL) == -1) {
                perror("timerfd_settime");
                exit(1);
            }
            mTimeOutCb = std::move(cb);
        }`

```

周期定时任务 RunEvery 的大体与 RunAfter 都是一致的，只在构建计时配置对象 new_value 的时候，略有不同。如下面的代码片段所示，RunEvery 给字段 it_interval 赋值了。

```
        `struct itimerspec new_value;
        new_value.it_value.tv_sec = mOriTime.tv_sec;
        new_value.it_value.tv_nsec = mOriTime.tv_nsec;
        new_value.it_interval.tv_sec = time.tv_sec;
        new_value.it_interval.tv_nsec = time.tv_nsec;`

```

我们提供了一个周期输出日志的[demo](https://github.com/gaoyichao/XiaoTuNetBox/blob/v0.0.5/test/t_Timer.cpp)，如下面的代码片段所示。我们在 main 函数中， 先创建了 PollLoop 和 Timer 对象。然后通过 RunEvery 接口，设定计时器每隔一秒调用一次回调函数 OnTimeOut。如右侧所示，在 OnTimeOut 中，我们直接输出函数名称。 最后在 main 函数中注册 timer 的事件分发器，并开启 Loop 循环。如果编译运行一切顺利，我们是可以看到程序在终端里每隔一秒输出一个 "OnTimeOut" 的。

| \`\``
        `int main(int argc, char \*argv\[]) {
            PollLoopPtr loop = CreatePollLoop();
            TimerPtr timer = TimerPtr(new Timer());

            struct timespec t = { 1, 0 };
            timer->RunEvery(t, std::bind(OnTimeOut, timer));
        
            ApplyOnLoop(timer, loop);
            loop->Loop(10000);
            return 0;
        }`

````

 | ```
        `void OnTimeOut(TimerPtr const & timer) {
            std::cout << __FUNCTION__ << std::endl;
        }`

````

 \|

## 3. 完

我们可以通过 timerfd_create 构建一个计时器并获取它的文件描述符，有了文件描述符我们就可以将计时过程融合到 PollLoop 的框架下。所以为之创建了一个数据类型 Timer 来对其进行封装。 通过 timerfd_settime 可以添加定时方案，我们为 Timer 提供了 RunAfter 和 RunEvery 两个接口，分别用于在指定时间之后执行任务，或者周期性的运行。 
 [https://gaoyichao.com/Xiaotu/?book=Linux%E4%B8%8B%E7%9A%84%E4%BA%8B%E4%BB%B6%E4%B8%8E%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B&title=timerfd%E4%B8%8E%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1](https://gaoyichao.com/Xiaotu/?book=Linux%E4%B8%8B%E7%9A%84%E4%BA%8B%E4%BB%B6%E4%B8%8E%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B&title=timerfd%E4%B8%8E%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1)
