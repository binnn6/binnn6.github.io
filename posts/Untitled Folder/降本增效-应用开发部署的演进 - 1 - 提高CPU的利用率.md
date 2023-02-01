# 提高CPU的利用率

## cpu virtualization - process vs thread (LWP)

[Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/)中提到cpu通过时间片轮转(time sharing)实现了一个cpu上可以同时执行多个进程，其实从程序员的视角来说就是操作系统将一个cpu虚拟化成多个cpu，实现进程业务逻辑的时候无需考虑cpu资源的事情，每个进程都可以认为自己是独占cpu。

> cpu虚拟化(多进程)提升了机效 + 人效，同样虚拟内存也是类似。
>
> - 提高cpu利用率
> - 降低理解成本

可以想象一下，如果没有这个时间片轮转的技术，一个cpu同时只能执行一个进程，那一台计算机同时执行的进程数将受到cpu个数的限制、同时cpu也难以得到充分利用。

由于创建进程需要涉及分配、回收资源，比如内存、文件、文件系统等，导致其开销比较大。因此又出现了线程的概念。

在Linux中，线程其实就是轻量级的进程，其实对于调度系统来说，进程和线程是一样的，都是对应一个task_struct结构体，而Linux实际上调度正是task_struct。

<img src="http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-01-06-160006.jpg" alt="img" style="zoom:67%;" />

怎么理解轻量级呢？

1. 其实进程和线程都是通过[clone系统调用](https://man7.org/linux/man-pages/man2/clone.2.html)创建的，只是参数不同，创建线程时，会共享父线程的各种资源、比如VM、fd table等，不像进程一样拥有自己的VM、fd table等资源。所以才有了**进程是资源分配的最小单位，而线程是调度的最小单位**的说法。

2. 当然值得一提的是创建进程也不是创建时就把所有的父进程的资源都copy过来，而是采用COW，确保系统可以高效的创建进程。

## cpu scheduling

有了进程和线程，那么如何分配cpu呢？

非抢占式调度和抢占式调度

### non-preemptive scheduling

非抢占式调度就是完全依赖用户程序自行让出cpu，对程序员的要求比较高，很容易导致某个程序长时间占用cpu，导致其他程序得不到执行。

<img src="http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-01-25-164557.png" alt="img" style="zoom:100%;" />

### preemptive scheduling

抢占式调度则会强制让长时间执行的进程让出cpu，给予其他进程一定的运行时间。

抢占式调度让每个进程都得到了执行的机会，副作用则是产生了上下文切换的开销、同时增加内核调度系统的复杂度。 其中进程A让出cpu、保存进程A的现场[寄存器、缓存]、进程B调度到cpu、恢复进程B的现场就是所谓的上下文切换。

<img src="http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-01-25-164628.png" alt="img" style="zoom:100%;" />

根据抢占发生的时机, 抢占又分为**用户抢占**和**内核抢占**。<img src="http://devops-1255386119.cos.ap-beijing.myqcloud.com/2023-01-25-165748.png" alt="image-20230126005745672" style="zoom:100%;" />

图片摘自[Real-Time-Linux 第9页](http://nyx.skku.ac.kr/wp-content/uploads/2019/06/ESW11-Real-Time-Linux.pdf)

用户抢占发生在从中断或者系统调用返回用户态时；如果一个系统调用或者中断耗时很长，就会导致其他进程长时间无法获取cpu。因此Linux系统自2.6内核开始支持内核抢占式调度。内核抢占相比用户抢占，允许从中断返回内核态时发生抢占，使高优先级的进程更及时的获取到时间片。

> ##### 内核抢占的时机
>
> When an interrupt handler exits, before returning to kernel-space
>
> When kernel code becomes preemptible again
>
> If a task in the kernel explicitly calls schedule()
>
> If a task in the kernel blocks (which results in a call to schedule() )

## 上下文切换 - context switch overhead

正是因为Linux选择了抢占式调度、并且是内核抢占，导致进程和线程存在**[上下文切换(Context Switch)](https://en.wikipedia.org/wiki/Context_switch)的开销**。

同一应用的不同线程间切换，由于共享内存的缘故，避免了TLB flush，所以成本会更低一些。([进程和线程的context switch耗时](https://stackoverflow.com/questions/5440128/thread-context-switch-vs-process-context-switch))其实差距不大。

**这里的提高cpu利用率并不是说top中显示的cpu利用率越高越好，而是提高user cpu time，毕竟context switch其实会提高system cpu time。**

有没有办法避免Context Switch?

> ##### 上下文切换分类
>
> - 进程切换
>
> - 内核态和用户态相互切换
>
> - 中断处理
>
> ##### 一些优化context switch overhead的措施
>
> io多路复用 
>
> 协程
>
> spin lock、 futex、vDSO
>
> spdk

## 异步非阻塞 - async no-blocking 

**采用async no-blocking方式取代blocking sync system call,不创建那么多进程或者线程，开启cpu affinity** 。

> 以上主要是从充分利用cpu的立场考虑的，但实际场景中要考虑更多的因素，比如内存、磁盘，然后根据具体的场景进行取舍。

比如nginx利用event-driven architecture(**epoll,multiplexing noblocking**)实现了一个进程可以同时处理多个请求，使其性能远远超过了[apache(one thread per request)](https://www.hostinger.com/tutorials/nginx-vs-apache-what-to-use/)。

然而异步的编程方式其实是不符合人类的思维习惯,因此会导致代码复杂且易出现bug。

> 异步编程的出现，提升了机效，但实际增加程序员的心智负担，提升机效的同时，不损失程序员的体验呢？

## 用户态调度 - userspace schedule

而[协程](https://en.wikipedia.org/wiki/Coroutine)则可以解决这个问题，协程让程序员可以利用同步的方式编写异步的服务(程序员可以像编写多进程的同步服务一样通过协程进行编程，同时避免context switch overhead)，将复杂的回调逻辑屏蔽在runtime中。**协程将原本交给进程调度的任务交给了用户态的应用**，跟操作系统的调度给程序员带来类似的体验提升，代码实现中无需关注调度细节，认为自己是独占cpu，因此协程也可以被用户态的调度。

其实挺有意思，Linux内核采用了内核抢占的调度方式，解决了非抢占式调度的任务可能长时间占用cpu的问题，但程序员无需感知调度方式；go runtime又通过支持goroutine、支持了用户态的抢占式任务调度，同样也是程序员透明的。当然go不支持yield，某种意义上讲其实yield相当于用户态度的schedule()。

从用户态和内核态的角度来说，golang算是用户态的非抢占式调度，毕竟goroutine是用户态实现了调度；从程序员的角度，golang又是抢占式调度，毕竟go runtime支持了抢占，写代码时无需过度关注。

而golang的runtime中的协程的底层thread又依赖内核抢占的调度方式，因此有时我们也要确保golang底层不要启动太多thread([Go gomaxprocs 调高引起调度性能损耗](https://mp.weixin.qq.com/s/k9Zg3ji4Hp8dmYEkqx-gAg)。

同时一个比较有意思的点、golang中的sync.Mutex底层是直接依赖futex么？[ITT 2019 - Kavya Joshi - Let's talk locks!](https://www.youtube.com/watch?v=tjpncm3xTTc&ab_channel=IstanbulTechTalks)

协程这个概念本身比较模糊，甚至有人说[goroutine不算协程](https://stackoverflow.com/questions/18058164/is-a-go-goroutine-a-coroutine)，因为协程中有要支持yield, 争论这个意义不大，了解原理即可。

go算是完全自动化的调度，是否让出cpu完全由runtime决定，而且[从Go 1.14开始支持抢占式](https://go.dev/doc/go1.14#runtime)；

[lua的协程](https://www.lua.org/pil/9.1.html)则支持手动的yield和resume,只是程序员手动调度。

>关于协程的一些资料
>
>[Kavya Joshi - The Scheduler Saga](https://www.youtube.com/watch?v=YHRO5WQGh0k&ab_channel=GopherAcademy)
>
>[Dmitry Vyukov — Go scheduler: Implementing language with lightweight concurrency](https://www.youtube.com/watch?v=-K11rY57K7k&t=2769s&ab_channel=Hydra)
>
>[浅谈协程](https://mp.weixin.qq.com/s/SyWjLg3lYx3pIJQfEtik8Q)
>
>[当谈论协程时，我们在谈论什么](https://mp.weixin.qq.com/s/IO4ynnKEfy2Rt-Me7EIeqg)


> 另外一些脑暴的内容后续填坑
> 1. stackful和stackless实现的区别，libco、goroutine等实现方式、semi-coroutine
> 3. 另外一些性能优化关键词 [spdk/dpdk](https://www.zhihu.com/question/313210254/answer/618781828)、io-uring、zero-copy、futex、DMA、sendfile、splice、cow、cpu affinity

> C语言的unsafe特性致命打击了操作系统进化过程；导致mmu出现，kernel和userspace严格隔离和高成本切换，用户空间几乎彻底失去调度权，催生面向对象，又糊上GC，这些全部都是走了弯路。如果一开始就safe，后面会完全不同。 
>
> ​                                                                -- 微博上看到的观点、脑洞比较大、仅引用、不代表赞成

---

**[查看原文](https://oops-oom.github.io/posts/%E9%99%8D%E6%9C%AC%E5%A2%9E%E6%95%88-%E5%BA%94%E7%94%A8%E5%BC%80%E5%8F%91%E9%83%A8%E7%BD%B2%E7%9A%84%E6%BC%94%E8%BF%9B/%E9%99%8D%E6%9C%AC%E5%A2%9E%E6%95%88-%E5%BA%94%E7%94%A8%E5%BC%80%E5%8F%91%E9%83%A8%E7%BD%B2%E7%9A%84%E6%BC%94%E8%BF%9B%20-%200%20-%20%E5%AF%BC%E8%AF%AD.html)**
