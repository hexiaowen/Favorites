# Rust Runtime 设计与实现-科普篇 | 下一站 - Ihcblog!
本系列文章主要介绍如何设计和实现一个基于 io-uring 的 Thread-per-core 模型的 Runtime。

我们的 Runtime 最终产品 **Monoio** 现已开源，你可以在 [github.com/bytedance/monoio](https://github.com/bytedance/monoio) 找到它。

1.  [**Rust Runtime 设计与实现 - 科普篇**](https://www.ihcblog.com/rust-runtime-design-1/)
2.  [Rust Runtime 设计与实现 - 设计篇 - Part1](https://www.ihcblog.com/rust-runtime-design-2/)
3.  [Rust Runtime 设计与实现 - 设计篇 - Part2](https://www.ihcblog.com/rust-runtime-design-3/)
4.  [Rust Runtime 设计与实现 - 组件篇](https://www.ihcblog.com/rust-runtime-design-4/)

本文是系列的第一篇，主要做一些科普啦～以及会写一个极简的 Runtime 作为例子。

## [](#并行-amp-并发 "并行 & 并发")并行 & 并发

并行不等于并发。

举例来说，我想以最快的速度探测 10 个 URL 的可用性：

1.  最 naive 的方式是串行请求，一个循环就可以解决问题。但是显然我们的 CPU 和网卡并没有达到瓶颈，而且多个 URL 之间也没有依赖性，那理论上是可以更快的。
2.  我们可以创建一个线程池来做这件事。如线程池里刚好有足够多的可用线程，我们可以直接将这 10 个任务分发给大家并行执行。并行意味着：有多个计算资源（可能是 CPU，也可以将线程认为是一个独立的计算资源），每个任务独占一个。所以在线程这个维度上，我们可以认为这些任务是并行的。
3.  我们还可以将这个 10 个 URL 创建为 10 个 Task，并在 Runtime 中异步执行。我们的 Runtime 可以是单线程的，那为什么能够起到加速的作用呢？因为这些 Task 并不会持续地独占资源。当出现 IO 阻塞时，线程执行权会被让出，留给其他任务使用。这些任务是并发的。  
    ![](https://www.ihcblog.com/images/9ee02b9cfbb49bfe66c351b6e0d94edf884393ae.png)

可以看出，并行的重点在于同时和独占，而并发的重点在于任务级别的伪同时执行，它们可以共享计算资源。

## [](#epoll-amp-io-uring "epoll & io-uring")epoll & io-uring

为了做到前面提到的 “并发”，我们需要内核提供相关的能力。

### [](#epoll "epoll")epoll

讲 epoll 的文章多如牛毛，在此简单地提一下：epoll 是 linux 下较好的 IO 事件通知机制，可以同时监听多个 fd 的 ready 状态。

它主要包含 3 个 syscall：

1.  `epoll_create`  
    创建 epoll fd。
2.  `epoll_ctl`  
    向 epoll fd 添加、修改或删除其监听的 fd event。
3.  `epoll_wait`  
    等待监听 fd event，任何一个 event 发生时即返回；同时也支持传入一个 timeout，这样即便是没有 ready event，也可以在超时后返回。

如果你不使用 epoll 这类，而是直接做 syscall，那么你需要让 fd 处于阻塞模式（默认），这样当你想从 fd 上 read 时，read 就会阻塞住直到有数据可读。

使用 epoll 的时候，需要设置 fd 为非阻塞模式。当 read 时，在没有数据的情况下，read 也会立刻返回 `WOULD_BLOCK`。这时需要做的事情是将这个 fd 注册到 epoll fd 上，并设置 `EPOLLIN` 事件。

之后在没事做的时候（所有任务都卡在 IO 了），陷入 syscall epoll_wait。当有 event 返回时，再对对应的 fd 做 read（取决于注册时设置的触发模式，可能还要做一些其他事情，确保下次读取正常）。

总的来说，这个机制十分简单：设置 fd 为非阻塞模式，并在需要的时候注册到 epoll fd 上，然后 epoll fd 的事件触发时，再对 fd 进行操作。这样将多个 fd 的阻塞问题转变为单个 fd 的阻塞。

### [](#io-uring "io-uring")io-uring

与 epoll 不同，io-uring 不是一个事件通知机制，它是一个真正的异步 syscall 机制。你并不需要在它通知后再手动 syscall，因为它已经帮你做好了。

io-uring 主要由两个 ring 组成（SQ 和 CQ），SQ 用于提交任务，CQ 用于接收任务的完成通知。任务（Op）往往可以对应到一个 syscall（如 read 对应 ReadOp），也会指定这次 syscall 的参数和 flag 等。

在 submit 时，内核会消费掉所有 SQE，并注册 callback。之后等有数据时，如网卡中断发生，数据通过驱动读入，内核就会触发这些 callback，做 Op 想做的事情，如拷贝某个 fd 的数据到 buffer（这个 buffer 是用户指定的 buffer）。相比 epoll，io-uring 是纯同步的。

> 注：本节涉及的 io-uring 相关是对带 FAST_POLL 的描述。
>
> [https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d7718a9d25a61442da8ee8aeeff6a0097f0ccfd6](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=d7718a9d25a61442da8ee8aeeff6a0097f0ccfd6)

总结一下，io-uring 和 epoll 在使用上其实差不多，一般使用方式是：直接将想做的事情丢到 SQ 中（如果 SQ 满了就要先 submit 一下），然后在没事干（所有任务都卡在 IO 了）的时候 `submit_and_wait(1)`（`submit_and_wait` 和 `submit` 并不是 syscall，它们是 liburing 对 `enter` 的封装）；返回后消费 CQ，即可拿到 syscall 结果。如果你比较在意延迟，你可以更激进地做 `submit`，尽早将任务推入可以在数据 ready 后尽早返回，但与此同时也要付出 syscall 增多的代价。

## [](#异步任务的执行流 "异步任务的执行流")异步任务的执行流

常规的编程方式下，我们的代码控制流是对应线程的。就像你在写 C 时理解的那样，你的代码会直接编译到汇编指令，然后会由操作系统提供的 “线程” 去执行，在这其中没有多余的插入的逻辑。

以 epoll 为例，基于 epoll 的异步本质上是对线程的多路复用。那么常规方式下的类似下面的代码就无法在这种场景下使用：

```plaintext
for connection = listener.accept():
    do_something(connection)
```

因为这段代码中的 accept 是需要等待 IO 的，直接阻塞在这里会导致线程阻塞，这样就无法执行其他任务了。

### [](#面向-Callback-编程 "面向 Callback 编程")面向 Callback 编程

在 C/C++ 中常被使用的 libevent 就是这种模型。用户代码不掌握主动权（因为线程的掌控权就一个，而用户任务千千万万），而是通过将 Callback 注册在 libevent 上，关联某个事件。当事件发生时，libevent 会调用用户的回调函数，并将事件的参数传递给用户。用户在初始化好一些 callback 后，将线程的主动权交给 libevent。其内部会帮忙处理和 epoll 的交互，并在 ready 时执行 callback。

这种方式较为高效，但是写起来却令人头大。举例来说，如果你想做一次 HTTP 请求，那么你需要将这段代码拆成多个同步的函数，并通过 callback 将他们串起来：  
![](https://www.ihcblog.com/images/e9ccbfb5aa44e39d6e84429595d2c4691b83ba98.png)

本来一次可以内聚在一个一个函数里的行为，被拆成了一堆函数。相比面向过程，面向状态编程十分混乱，且容易因为编码者遗忘一些细节而出问题。

### [](#有栈协程 "有栈协程")有栈协程

如果我们能在用户代码和最终产物之间插入一些逻辑呢？像 Golang 那样，用户代码实际上只对应到可被调度的 goroutine，实际拥有线程控制权的是 go runtime。goroutine 可以被 runtime 调度，在执行过程中也可以被抢占。

当 goroutine 需要被中断然后切换到另一个 goroutine 时，runtime 只需要修改当前的 stack frame 即可。每个 goroutine 对应的栈其实是存在堆上的，所以可以随时打断随时复原。

网络库也是配合这一套 runtime 的。syscall 都是非阻塞的，并可以自动地挂在 netpoll 上。

有栈协程配合 runtime，解耦了 Task 级的用户代码和线程的对应关系。

### [](#基于-Future-的无栈协程 "基于 Future 的无栈协程")基于 Future 的无栈协程

有栈协程的上下文切换开销不可忽视。因为可以被随时打断，所以我们有必要保存当时的寄存器上下文，否则恢复回去时就不能还原现场了。

无栈协程没有这个问题，这种模式非常符合 Rust 的 Zero Cost Abstraction 的理念。Rust 中的 `async + await` 本质上是代码的自动展开，`async + await` 代码会基于 llvm generator 自动展开成状态机，状态机实现 Future 通过 poll 和 runtime 交互（具体细节可以参考[这篇文章](https://hsqstephenzhang.github.io/2021/11/24/rust/future-explained0/)）。

## [](#Rust-异步机制原理 "Rust 异步机制原理")Rust 异步机制原理

Rust 的异步机制设计较为复杂，标准库的接口和 Runtime 实现是解耦的。

Rust 的异步依赖于 Future trait。

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

Future 是如何被执行的？根据上述 trait 定义很显然，是通过 `poll` 方法。返回的结果是 `Poll`，要么 `Pending` 要么 `Ready(T)`。

那么 poll 方法是谁来调用的？

1.  用户。用户可以手动实现 Future，如用户对某个 Future 做包装，显然它需要实现 `poll` 并调用 `inner.poll`。
2.  Runtime。Runtime 是最终 `poll` 的调用者。

作为 Future 的实现者，需要保证的是，一旦返回了 `Poll::Pending`，就要在未来该 Future 依赖的 IO 就绪后能够唤醒它。唤醒一个 Future 是通过 Context 内的 Waker 做到的。至于说唤醒之后要做什么，这个由 Runtime 所提供的 cx 自行实现（如将这个 Task 加入到待执行队列）。

所以任何会产生事件的东西都要负责存储 Waker 并在 Ready 后唤醒它；而提供 cx 的东西会接收到这次唤醒操作，要负责重新调度它。这两个概念分别对应 Reactor 和 Executor，这两套东西靠 Waker 和 Future 解耦。

![](https://www.ihcblog.com/images/062812972a4bd0e09fdce4bff62204bc1973e5ac.png)

### [](#Reactor "Reactor")Reactor

举例来说，你甚至可以在 Tokio 之上再实现一套自己的 Reactor（约等于自己做了一套多路复用）。事实上 Tokio-uring 就是这么做的：它本身注册在 Tokio(mio) 上了一个 uring fd，而基于这个 fd 和一套自己的 Pending Op 管理系统又对外作为 Reactor 暴露了事件源的能力。在 tokio-tungstenite 中，也通过 `WakerProxy` 来解决了读写唤醒问题。

另一个例子是计时器驱动。显然 IO 事件的信号来自于 epoll/io-uring 等，但计时器并不是，其内部维护了时间轮或 N 叉堆之类的结构，所以唤醒 Waker 一定是 Time Driver 的职责，所以它一定需要存储 Waker。Time Driver 在这个意义上是一个 Reactor 实现。

### [](#Executor "Executor")Executor

说完了 Reactor 我们来科普一下 Executor。Executor 负责任务调度和执行。以 Thread-per-core 场景为例（为啥用这个例子？没有跨线程调度写起来多简单啊），完全可以实现成一个 `VecDeque`，Executor 做的事就是从里面拿任务，调用它的 poll 方法。

### [](#IO-组件 "IO 组件")IO 组件

看到这里你可能会好奇，既然 Reactor 负责唤醒对应 Task，Executor 负责执行唤醒的 Task，那我们看一下源头，是谁负责将 IO 注册到 Reactor 上的呢？

IO 库（如 TcpStream 实现）会负责将未能立刻完成的 IO 注册到 Reactor 上。这也是为什么你在使用 Tokio 的时候会非要用 `Tokio::net::TcpStream` 而不能用标准库的原因之一；以及你想异步 sleep 时也需要用 Runtime 提供的 sleep 方法。

## [](#实现一个极简的-Runtime "实现一个极简的 Runtime")实现一个极简的 Runtime

出于简便目的，我们使用 epoll 来写这个 Runtime。这并不是我们的最终产品，只是为了演示如何实现最简单的 Runtime。

本小节的完整代码在 [github.com/ihciah/mini-rust-runtime](https://github.com/ihciah/mini-rust-runtime) 。

对应前文的三部组成部分：Reactor、Executor 和 IO 组件，我们分别实现。我们从 Reactor 入手吧。

## [](#Reactor-1 "Reactor")Reactor

由于裸操作 epoll 的体验有点差，并且本文的重点也并不是 Rust Binding，所以这里使用 polling 这个 crate 来完成 epoll 操作。

polling crate 的基础用法是创建 `Poller`，以及向 `Poller` 添加或修改 fd 和 interest。这个包默认使用 oneshot 模式（`EPOLLONESHOT`），需要在 event 触发后重新注册。在多线程场景下这可能是有用的，不过在我们的单线程的最简版本中似乎没有必要这么做，因为会带来额外的 `epoll_ctl` syscall 开销。不过处于简便起见，我们仍旧使用它。

作为 Reactor，我们在向 `Poller` 注册 interest 时，需要提供一个其对应的标识符，这个标识符在很多其他地方会被叫做 Token 或 UserData 或 Key。当 Event Ready 后，这个标志符会被原样返回。

所以我们需要做的事情大概是这样的：

1.  创建 `Poller`
2.  当需要关注某个 fd 的 Readable 或 Writable 时，向 `Poller` 添加 interest Event，并将 Waker 存下来
3.  添加 interest Event 前需要分配一个 Token，这样当 Event Ready 后我们才知道这个 Event 对应的 Waker 在哪。

于是我们可以将 Reactor 设计为：

```rust
pub struct Reactor {
    poller: Poller,
    waker_mapping: rustc_hash::FxHashMap<u64, Waker>,
}
```

在其他 Runtime 实现中往往会使用 slab，slab 同时处理了 Token 分配和 Waker 的存储。

简便起见，这里保存 Token 和 Waker 的关系时直接使用了 HashMap。Waker 的存储比较 trick 的方式解决：由于我们只关心读和写，所以我们将读对应的 MapKey 定义为 `fd * 2`，写对应的 MapKey 定义为 `fd * 2 + 1`（因为 TCP 连接是全双工的，同一个 fd 上的读写无关，可以在不同的 Task 上，有各自独立的 Waker）；而 Event 的 UserData（Token）仍旧使用 fd 本身。

```rust
impl Reactor {
    pub fn modify_readable(&mut self, fd: RawFd, cx: &mut Context) {
        println!("[reactor] modify_readable fd {} token {}", fd, fd * 2);

        self.push_completion(fd as u64 * 2, cx);
        let event = polling::Event::readable(fd as usize);
        self.poller.modify(fd, event);
    }

    pub fn modify_writable(&mut self, fd: RawFd, cx: &mut Context) {
        println!("[reactor] modify_writable fd {}, token {}", fd, fd * 2 + 1);

        self.push_completion(fd as u64 * 2 + 1, cx);
        let event = polling::Event::writable(fd as usize);
        self.poller.modify(fd, event);
    }

    fn push_completion(&mut self, token: u64, cx: &mut Context) {
        println!("[reactor token] token {} waker saved", token);

        self.waker_mapping.insert(token, cx.waker().clone());
    }
}
```

要将 fd 挂在 Poller 上或摘掉也十分简单：

```rust
impl Reactor {
    pub fn add(&mut self, fd: RawFd) {
        println!("[reactor] add fd {}", fd);

        let flags =
            nix::fcntl::OFlag::from_bits(nix::fcntl::fcntl(fd, nix::fcntl::F_GETFL).unwrap())
                .unwrap();
        let flags_nonblocking = flags | nix::fcntl::OFlag::O_NONBLOCK;
        nix::fcntl::fcntl(fd, nix::fcntl::F_SETFL(flags_nonblocking)).unwrap();
        self.poller
            .add(fd, polling::Event::none(fd as usize))
            .unwrap();
    }

    pub fn delete(&mut self, fd: RawFd) {
        println!("[reactor] delete fd {}", fd);

        self.completion.remove(&(fd as u64 * 2));
        println!("[reactor token] token {} completion removed", fd as u64 * 2);
        self.completion.remove(&(fd as u64 * 2 + 1));
        println!(
            "[reactor token] token {} completion removed",
            fd as u64 * 2 + 1
        );
    }
}
```

一个注意事项是，在挂上去之前要设置为 Nonblocking 的，否则在做 syscall 时，如果出现误唤醒（epoll 并没有保证不会误唤醒）会导致线程阻塞。

然后我们会面临一个问题：什么时候 `epoll_wait`？答案是没有任务的时候。如果所有任务都等待 IO 了，那么我们可以安全地陷入 syscall。所以我们的 Reactor 需要暴露一个 wait 接口来供 Executor 在没有任务时等待。

```rust
pub struct Reactor {
    poller: Poller,
    waker_mapping: rustc_hash::FxHashMap<u64, Waker>,

    buffer: Vec<Event>,
}

impl Reactor {
    pub fn wait(&mut self) {
        println!("[reactor] waiting");
        self.poller.wait(&mut self.buffer, None);
        println!("[reactor] wait done");

        for i in 0..self.buffer.len() {
            let event = self.buffer.swap_remove(0);
            if event.readable {
                if let Some(waker) = self.waker_mapping.remove(&(event.key as u64 * 2)) {
                    println!(
                        "[reactor token] fd {} read waker token {} removed and woken",
                        event.key,
                        event.key * 2
                    );
                    waker.wake();
                }
            }
            if event.writable {
                if let Some(waker) = self.waker_mapping.remove(&(event.key as u64 * 2 + 1)) {
                    println!(
                        "[reactor token] fd {} write waker token {} removed and woken",
                        event.key,
                        event.key * 2 + 1
                    );
                    waker.wake();
                }
            }
        }
    }
}
```

接收 syscall 结果时需要提供预分配好的 buffer（`Vec<Event>`），为了避免每次都分配，我们直接在结构体中保存下来这个 buffer，通过 `Option` 包装可以让我们临时地拿到它的所有权。

`wait` 需要做的事情是：

1.  陷入 syscall
2.  syscall 返回后，处理所有就绪的 Event。如果事件是 readable 或者 writable 的，那么就分别从 HashMap 里找到并删除其对应的 Completion，然后唤醒它（这个 fd 和 Map Key 的对应规则我们前面也说到了，readable 对应 `fd * 2`，writable 对应 `fd * 2 + 1`）。

最后将 Reactor 的创建函数补齐：

```rust
impl Reactor {
    pub fn new() -> Self {
        Self {
            poller: Poller::new().unwrap(),
            waker_mapping: Default::default(),

            buffer: Vec::with_capacity(2048),
        }
    }
}

impl Default for Reactor {
    fn default() -> Self {
        Self::new()
    }
}
```

这时我们的 Reactor 就写完了。总的来说就是包装了 epoll，同时额外做了 Waker 的存储和唤醒。

## [](#Executor-1 "Executor")Executor

Executor 需要存储 Task 并执行。

### [](#Task "Task")Task

Task 是什么？Task 其实是 Future，但因为 Task 需要共享所有权，所以这里我们使用 Rc 来存储；并且我们只知道用户丢进来一个 Future，并不知道它的具体类型，所以我们需要把它 Box 起来，这里使用 `LocalBoxFuture`。再加上内部可变性，所以 Task 的定义如下：

```rust
pub struct Task {
    future: RefCell<LocalBoxFuture<'static, ()>>,
}
```

### [](#TaskQueue "TaskQueue")TaskQueue

设计 Task 的存储结构，简便起见直接使用 VecDeque。

```rust
pub struct TaskQueue {
    queue: RefCell<VecDeque<Rc<Task>>>,
}
```

这个 TaskQueue 需要能够 push 和 pop 任务：

```rust
impl TaskQueue {
    pub(crate) fn push(&self, runnable: Rc<Task>) {
        println!("add task");
        self.queue.borrow_mut().push_back(runnable);
    }

    pub(crate) fn pop(&self) -> Option<Rc<Task>> {
        println!("remove task");
        self.queue.borrow_mut().pop_front()
    }
}
```

### [](#Waker "Waker")Waker

Executor 需要提供 Context，其中包含 Waker。Waker 需要能够在被 wake 时将任务推入执行队列等待执行。

```rust
pub struct Waker {
    waker: RawWaker,
}

pub struct RawWaker {
    data: *const (),
    vtable: &'static RawWakerVTable,
}
```

Waker 通过自行指针和 vtable 自行实现动态分发。所以我们要做的事情有两个：

1.  拿到 Task 结构的指针并维护它的引用计数
2.  生成类型对应的 vtable

我们可以这么定义 vtable：

```rust
struct Helper;

impl Helper {
    const VTABLE: RawWakerVTable = RawWakerVTable::new(
        Self::clone_waker,
        Self::wake,
        Self::wake_by_ref,
        Self::drop_waker,
    );

    unsafe fn clone_waker(data: *const ()) -> RawWaker {
        increase_refcount(data);
        let vtable = &Self::VTABLE;
        RawWaker::new(data, vtable)
    }

    unsafe fn wake(ptr: *const ()) {
        let rc = Rc::from_raw(ptr as *const Task);
        rc.wake_();
    }

    unsafe fn wake_by_ref(ptr: *const ()) {
        let rc = mem::ManuallyDrop::new(Rc::from_raw(ptr as *const Task));
        rc.wake_by_ref_();
    }

    unsafe fn drop_waker(ptr: *const ()) {
        drop(Rc::from_raw(ptr as *const Task));
    }
}

unsafe fn increase_refcount(data: *const ()) {
    let rc = mem::ManuallyDrop::new(Rc::<Task>::from_raw(data as *const Task));
    let _rc_clone: mem::ManuallyDrop<_> = rc.clone();
}
```

手动管理引用计数：我们会通过 `Rc::into_raw` 来取得 `Rc<Task>` 的裸指针并使用它和 vtable 构建 RawTask 然后构建 Task。在 vtable 实现中，我们需要小心地手动管理引用计数：如 `clone_waker` 时，虽然我们只 clone 一个指针，但它在含义上有了一次拷贝，所以我们需要手动让其引用计数加一。

Task 实现 `wake_` 和 `wake_by_ref_` 来重新调度任务。重新调度任务做的事情只是简单地从 thread local storage 拿到 executor 然后向 TaskQueue push。

```rust
impl Task {
    fn wake_(self: Rc<Self>) {
        Self::wake_by_ref_(&self)
    }

    fn wake_by_ref_(self: &Rc<Self>) {
        EX.with(|ex| ex.local_queue.push(self.clone()));
    }
}
```

### [](#Executor-2 "Executor")Executor

有了前面这些组件以后，构建 Executor 是非常简单的。

```rust
scoped_tls::scoped_thread_local!(pub(crate) static EX: Executor);

pub struct Executor {
    local_queue: TaskQueue,
    pub(crate) reactor: Rc<RefCell<Reactor>>,

    
    _marker: PhantomData<Rc<()>>,
}
```

当用户 spawn Task 的时候：

```rust
impl Executor {
    pub fn spawn(fut: impl Future<Output = ()> + 'static) {
        let t = Rc::new(Task {
            future: RefCell::new(fut.boxed_local()),
        });
        EX.with(|ex| ex.local_queue.push(t));
    }
}
```

其实做的事情就是将传入的 Future Box 起来然后构建 `Rc<Task>` 之后丢到执行队列里。

那么 Executor 的主循环在哪呢？我们可以放在 `block_on` 里。

```rust
impl Executor {
    pub fn block_on<F, T, O>(&self, f: F) -> O
    where
        F: Fn() -> T,
        T: Future<Output = O> + 'static,
    {
        let _waker = waker_fn::waker_fn(|| {});
        let cx = &mut Context::from_waker(&_waker);

        EX.set(self, || {
            let fut = f();
            pin_utils::pin_mut!(fut);
            loop {
                
                if let std::task::Poll::Ready(t) = fut.as_mut().poll(cx) {
                    break t;
                }

                
                while let Some(t) = self.local_queue.pop() {
                    let future = t.future.borrow_mut();
                    let w = waker(t.clone());
                    let mut context = Context::from_waker(&w);
                    let _ = Pin::new(future).as_mut().poll(&mut context);
                }

                
                if let std::task::Poll::Ready(t) = fut.as_mut().poll(cx) {
                    break t;
                }

                
                self.reactor.borrow_mut().wait();
            }
        })
    }
}
```

这段有点复杂，可以分成以下几个步骤：

1.  创建一个 dummy_waker，这个 waker 其实啥事不做。
2.  (in loop)poll 传入的 future，检查是否 ready，如果 ready 就返回，结束 block_on。
3.  (in loop) 循环处理 TaskQueue 中的所有 Task：构建它对应的 Waker 然后 poll 它。
4.  (in loop) 这时已经没有待执行的任务了，可能主 future 已经 ready 了，也可能都在等待 IO。所以再次检查主 future，如果 ready 就返回。
5.  (in loop) 既然所有人都在等待 IO，那就 `reactor.wait()`。这时 reactor 会陷入 syscall 等待至少一个 IO 可执行，然后唤醒对应 Task，会向 TaskQueue 里推任务。

至此 Executor 基本写完了。

## [](#IO-组件-1 "IO 组件")IO 组件

IO 组件要在 WouldBlock 时将其挂在 Reactor 上。以 TcpStream 为例，我们需要为其实现 `tokio::io::AsyncRead`。

```rust
pub struct TcpStream {
    stream: StdTcpStream,
}
```

在创建 TcpStream 时需要将其添加到 Poller 上，销毁时需要摘下：

```rust
impl From<StdTcpStream> for TcpStream {
    fn from(stream: StdTcpStream) -> Self {
        let reactor = get_reactor();
        reactor.borrow_mut().add(stream.as_raw_fd());
        Self { stream }
    }
}

impl Drop for TcpStream {
    fn drop(&mut self) {
        println!("drop");
        let reactor = get_reactor();
        reactor.borrow_mut().delete(self.stream.as_raw_fd());
    }
}
```

在实现 AsyncRead 时，对其做 read syscall。因为在添加到 Poller 时已经设置 fd 为非阻塞，所以这里 syscall 是安全的。

```rust
impl tokio::io::AsyncRead for TcpStream {
    fn poll_read(
        mut self: std::pin::Pin<&mut Self>,
        cx: &mut std::task::Context<'_>,
        buf: &mut tokio::io::ReadBuf<'_>,
    ) -> Poll<io::Result<()>> {
        let fd = self.stream.as_raw_fd();
        unsafe {
            let b = &mut *(buf.unfilled_mut() as *mut [std::mem::MaybeUninit<u8>] as *mut [u8]);
            println!("read for fd {}", fd);
            match self.stream.read(b) {
                Ok(n) => {
                    println!("read for fd {} done, {}", fd, n);
                    buf.assume_init(n);
                    buf.advance(n);
                    Poll::Ready(Ok(()))
                }
                Err(ref e) if e.kind() == std::io::ErrorKind::WouldBlock => {
                    println!("read for fd {} done WouldBlock", fd);
                    
                    let reactor = get_reactor();
                    reactor
                        .borrow_mut()
                        .modify_readable(self.stream.as_raw_fd(), cx);
                    Poll::Pending
                }
                Err(e) => {
                    println!("read for fd {} done err", fd);
                    Poll::Ready(Err(e))
                }
            }
        }
    }
}
```

read syscall 可能会返回正确的结果，也可能会返回错误。其中错误中有一个错误需要特殊处理，就是 `WouldBlock`。当 `WouldBlock` 时，我们便需要将其挂在 Reactor 上，这里通过我们前面定义的函数 `modify_readable` 表示我们对 readabe 是关心的。在挂 Reactor 动作完成后，我们可以放心地返回 `Poll::Pending`，因为我们知道，它后续会被唤醒。

本文科普了一些 **并发**、**epoll**、**io-uring** 等基础概念，并简要对比了 **面向 callback 编程模式**、**有栈协程模式** 和 **Future 抽象**；并针对 Rust 的异步模型做了深入解读。最后，基于 epoll 一步步地写了一个极简的 Runtime。

下一章我们我将介绍 Monoio 的模型、事件驱动、IO 接口等，以及在设计过程中的一些考量与思考。 
 [https://www.ihcblog.com/rust-runtime-design-1/](https://www.ihcblog.com/rust-runtime-design-1/)
