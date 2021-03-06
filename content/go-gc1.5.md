Go1.5的垃圾回收是一个非分代，非移动的，并发的，三色的标记清扫垃圾回收。

简单解释一下，非分代是指Go1.5的垃圾回收没有使用分代垃圾回收算法，非移动是指没有做内存的整理和紧缩，这里的"并发"是指在垃圾回收的时候，用户代码可以同时运行。三色标记清扫是一个经典的垃圾回收算法，更多基础知识相关可以看[另一篇博客](gc.md)，本文假设读者对于垃圾回收有一些基本的了解。

Go1.5垃圾回收的实现被划分为五个阶段：

- GCoff 垃圾回收关闭状态
- GCscan 扫描阶段
- GCmark 标记阶段，write barrier生效
- GCmarktermination 标记结束阶段，STW，分配黑色对象
- GCsweep 清扫阶段

![](static/gc.png)

从比较宏观的角度，描述Go1.5的GC流程：

0. 从GCoff切换到GCscan阶段
1. 等待所有P都获知这个变化。这时所有的goroutine都会经过一个GC安全点，并且知道当前进入了GCscan阶段
2. 扫描各个goroutine栈，将所有遇到的指针标记并插入队列
3. 切换为GCmark阶段
4. 等待所有P获知这处变化
5. write barrier生效，所有的黑色/灰色/白色对象到白色对象的修改都会被捕获，标记并插入队列。malloc会分配白色对象
6. 同时，GC会遍历并标记所有可达的对象
7. GC完成对堆的标记之后，会依次处理P，从队列中拿出一些对象
8. 一旦GC将队列中所有的对象都处理完了，将进入到下一个阶段：GCmarktermination
9. 等待所有P获知这处变化
10. malloc现在分配黑色对象，未标记的可达对象会不断地减少
11. 再一次地，从队列中拿出对象并标记所有没被标记到的可达对象
12. 当所有P都处理完毕，并且没有新的灰色对象了(意味着所有可达对象都标记了)，切换到GCsweep阶段
13. 等待所有P获知这处变化
14. 从现在开始，malloc分配白色对象(使用之前需要sweep span)。不再需要write barrier
15. GC后台sweep
16. sweep完成之后，切换回GCoff，等待下一轮

扫描阶段到标记阶段切换是当没有更多的指针需要扫描时触发。扫描阶段只是将对象标记为灰色，但是不会标记为黑色。切换到标记阶段之后，write barrier生效。

-------------------------

Go1.5的设计目标是，尽量缩短STW(stop the world)的时间，提高应用程序的实时性。不stop the world，意味着垃圾回收过程将和用户代码同时运行。垃圾回收是在整理内存，而用户代码在分配和修改内存，让它们同时工作是很有技术挑战的。

并发的三色标记算法是一个经典算法，通过write barrier，维护"黑色对象不能引用白色对象"这条约束，就可以保证程序的正确性。Go1.5会在标记阶段开启write barrier。在这个阶段里，如果用户代码想要执行操作，修改一个黑色对象去引用白色对象，则write barrier代码直接将该白色对象置为灰色。去读源代码实现的时候，有一个很小的细节：原版的算法中只是黑色引用白色则需要将白色标记，而Go1.5实现中是不管黑色/灰色/白色对象，只要引用了白色对象，就将这个白色对象标记。这么做的原因是，Go的标记位图跟对象本身的内存是在不同的地方，无法原子性地进行修改，而采用一些线程同步的实现代价又较高，所以这里的算法做过一些变种的处理。

只有标记阶段是加了write barrier的。标记结束阶段stop the world了，没有用户代码同时运行，不加write barrier容易理解，但是扫描阶段为什么不加write barrier呢？其实，这个阶段会依次扫描各个goroutine，而当扫描其中某个goroutine时，那个goroutine是需要暂停运行的，只是不算stop the world而已。暂停后用户代码和垃圾回收不会同时修改对象，因此不需要write barrier。扫描阶段不需要进行递归地标记处理，所以其实停顿goroutine开销也不大。

扫描阶段会将各个goroutine的栈对象放到一个队列里。到标记阶段，就会从队列里取对象，并将算法执行下去。注意到还有一个标记结束阶段，为什么需要这个阶段呢？还有，为什么标记结束阶段会stop the world呢？其实从技术角度，完全不stop the world的算法是有的。Go1.5目前这么做只是考虑保持实现的简单。在标记期间，由于write barrier的存在，不停地有对象放到工作队列中，而垃圾回收则不停地从工作队列中取数据，明确一个“完成”的状态并不容易。如果stop the world，当生产者停止，消费者将队列消费完后，就是一个“完成”的状态。

扫描过程是把root集合进队列；标记过程是不停地从队列中拿数据，然后标记新的数据进队列；在标记结束阶段，stop the world之后，GC会重新执行一次扫描和标记过程。因为在扫描阶段并没有扫描所有的root集合，只扫描了goroutine的栈。而这一次，会将其它root集合进行扫描和标记。

关于分配对象的颜色问题，标记结束阶段是分配黑色对象，而其它阶段都是分配白色对象。分配白色对象，是只要接下来有活跃对象引用到它，标记的过程中算法都能保证这个对象会被打上正确的颜色。在标记结束阶段直接分配黑色对象，是因为接下来也没有标记过程了。

--------------------

垃圾回收的算法需要用到一些额外的信息，比如对象标记的颜色，对象中是否包含指针（到其它对象的引用）。Go1.5使用位图来记录这些信息，每个内存地址都对应有一个位图。

有两种不同的位图表示，其中栈，数据区，bss区的位图，只需要用1位表示。因为这些是root集合对象，不需要标记是否活跃，只需要用1位来表示GC过程中是否需要访问指针。
另一种是堆位图，每个机器字长要用2个位表示。低位跟之前一样，0表示跳过，1表示需要访问指针。高位有其它含义。如果两位都是0，表示这是垃圾对象，垃圾回收不要处理它。

高位的含义由标记位对应的地址，相对于分配对象的首地址所在位置决定：

* 如果是第一个，则高位是GC标记位
* 如果是第二个，高位是GC的checkmarked位（用于调试）
* 如果是第三个或者更后，高位表示对象还在被描述中

Go1.5对2位表示的位图进行了分组处理，前面4个高位放一起，后面4个低位放一起。

标记位只是有1位，如何区分黑白灰三种颜色呢？Go1.5约定，标记并且在队列中的是灰色对象，标记了但是不在队列中的黑色对象，末标记的是白色对象。

这里面也还是有很多细节，像对象的修改并不是简单的一处内存改动，而相应的标记位也可能位到影响。还有比如memmove也是有writebarrier的，涉及到了一块bitmap的改动。memmove会发生什么事？引用这块对象的对象，都要修改。但是如何跟踪哪些对象引用这个对象呢？？？

-------------------

Go1.5希望GC延迟在10ms以前，并且应用代码第50秒钟至少可以运行40秒。拿出25%的GOMAXPROCS的计算能力来执行GC。如何控制GC的频率和CPU占用率呢？Go1.5有一个GC控制器和对应的[算法](http://golang.org/s/go15gcpacing)。

在1.5之前，垃圾回收是STW的，GC触发时机很容易捕获。当堆大小达到上一次GC完成后堆大小的两倍时，触发GC。比如上次GC完成，堆大小是4M，那么当堆增长到8M的时候会触发GC。但是在Go1.5中，分配量的计算就变得模糊了，因为应用程序可能在GC的过程中，分配对象。所以需要一个算法进行预估（GC控制器可以类比操作系统的进度调度器，控制算法可以类比进程调度的算法）。

算法比较复杂，只提一点点基本的思想，具体细节见原文。首先需要估算扫描的工作量。标记阶段消耗的CPU也是由扫描的数量控制的。然后需要用户代码协助。如果只有一个后台垃圾回收线程，而用户代码的分配速度快于标记的速度，最糟糕的情况垃圾回收将永远没机会完成。为了解决这种问题，用户代码在分配的时候可能会需要协助执行扫描的工作。接下来还需要进行CPU调度，用户代码协助会导致高估或者低估GC的CPU预算，所以要监控用户协助使用和后台回收使用的CPU，如果低于25%，切换后台扫描运行来提升GC的CPU使用率，否则可以不要运行后台收集以降低CPU。最后是触发频率控制，通过用户协助和CPU调度以及触发控制一起，创建一个反馈回路使得CPU使用率和堆增长都能达到优化目标。

本文写的超前了，目前Go1.5正式版还没放出来，不过git的代码已经可以去体验了^_^
