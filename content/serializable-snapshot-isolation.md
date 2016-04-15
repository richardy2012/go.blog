从浅往深的讲吧，先是一些基础知识，数据库教材里面都会有，网上一搜一大把。数据库会说到ACID，其中I是隔离性，是关于数据库在事务并行执行的时候的一些规定。隔离性是有一定的严格程度的，这个就是**事务的隔离级别**。SQL标准里面定义了下面四种隔离级别：

* 读未提交
* 读已提交
* 可重复读
* 可串行化

隔离级别的定义都是跟事务并发执行时的一些异常现象相关的，包括：读脏、丢失写、幻读、不可重复读等。读未提交就是事务1可能读到事务2修改过但还没提交的数据(包括修改后abort的情况)，这就是脏读。读未提交的隔离级别基本上是毫无意义的。读已提交的隔离级别不会读到未提交事务的数据，也就是没有脏读问题。

已经提交的事务就应该是持久的，如果提交后还会丢失，就是丢失写。英文应该叫update lose，翻译的不算准确，明白是那个意思就行。我举个例子，x当前是5，事务1把它改成了7并提交了。事务2跟事务1同时进行着，后面事务2执行时出现冲突abort了，但它的abort动作把x的值恢复成了5而不是7，那事务1的操作就丢失了。

不可重复读是指一次事务之内的读操作读出不同的结果。这里也举个例子，满足读已提交级别，却不满足可重复读的场景：事务1读x是5，然后其它事务将x改为6并提交，事务再读一次x发现它不是5了。这就是不可重复读的，但是满足了读已提交。

幻读(phantom)是指对表的结构的变化读到了不一致，前面是数据的值的不一致。比如表加了一行，同时执行的几个事务，有的读到19行，有的读到的是20行。

可串行化是最严格的级别，只有满足多个事务并发操作的结果，与这些事务按照某个顺序串行执行的结果相同时，我们说这个隔离级别是可串行化。实际上当说到事务，我们期望的都是可串行化，只是很多时候出于性能折衷，达到可能是读已提交或者可重复读。

## 快照隔离

快照隔离(snapshot isolation或者简写为SI)算一种事务隔离级别，但是却没有归类到上面的标准里面。原因是制订标准的时候，大家对事务的认识都还是基于锁的，不知道有其它的实现方式。基于锁的事务实现是有性能开销的。后来有了多版本并发控制(MVVC)，就超出来原来的分类。

简单的理解就是，写一个值时，不要在这个值上面做修改，而是创建一个新的版本，也就是所谓快照。而读一个值会读取最近提交成功了的那个版本的数据。读读肯定是没问题的，读写也没问题，读到的是最后提交的版本。至于写写，如果在提交的时候检测到有冲突，将这个事务abort掉就行了。所有的操作完全不阻塞，相比加锁等待的方案，性能一下子就上去了。

快照隔离可以解决可重复读这个级别的一些问题，但是还达不到可串行化。为什么说达不到可串行化呢？主要是存在write skew问题。为了方便说明，下面用wnx这种记法，w表示是写操作，n是事务的编号，x是操作的数据。比如w1x表示事务1对x执行写操作，r3y表示事务3对y执行读操作。用c/a分别表示commit和abort操作。

    r1x r2y w1y w2x c1 c2

这个顺序在快照隔离中是可以执行下来的，事务1读x写y，事务2是读y写x，没有冲突。但是这个结果不等价于任何一种串行执行结果：

    r1x w1y c1 r2y w2x c2
    rwy w2x c2 r1x w1y c1
    
事务1和事务2读到的x和y两个事务执行以前的x和y。而串行化要求读到的至少是其中一个更新了。

## 可串行化快照隔离

可串行化快照隔离(serializable snapshot isolation或SSI)是在快照隔离级别之上，支持串行化。

如果是事务1读x写y，事务2是读y写z会不会有write skew问题呢？不会。下面执行结果是等价于事务2先执行，然后事务1执行的串行顺序的。

    r1x r2y w1y w2z c1 c2
    r2y w2z c2 r1x w1y c2
    
仔细想一下，单独的读x写y，读y写x都可以，但两者一起，就出问题了。

总结出来，快照隔离存在的write skew的问题，本质上需要至少两个条件：

* 有读写冲突
* 依赖成环

如果可以破坏这些条件，就可以避免write skew，达到可串行化了。所以要做的就是检测不同事务之间的读写依赖是否形成环。x的值依赖于事务2，而被事务1依赖，y的值依赖于事务1，成环了。学过数据结构我们知道，检测图中存在环我们可以使用深度优先遍历遇到之前走过的结点。但是对性能有一定的影响。可以做一个简化的措施，允许误判以换取性能。

实现是这样子的，为每个值维护一个出边和一个入边。如果某个值即存在出边又存在入边，则**可能**是存在环的。我们宁可错杀一千，不要放过一个，事务操作中，只要检测到这种情况，就abort掉。还是看上面的例子：

    r1x r2y w1y w2x c1 c2

事务2读y，y被事务2依赖，于是在y的值设置出边。执行到w1y时，事务1写y，于是y依赖于事务1，设置y的入边。这时我们发现y的入边出边都设置了，有潜在成环的可能性。于是让abort掉事务1。

这里说的是可串行化快照实现方式之一。还有一些其它的方式也可以实现，本质上都是要对读写冲突进行处理。对于OLTP类型的业务，大部分的流量都是在读操作，而读是不会被abort掉的，所以这类场合，实现可串行化快照隔离引入的性能开销可以接受。

## 参考资料

[Serializable Isolation for Snapshot Databases](https://drive.google.com/file/d/0B9GCVTp_FHJIcEVyZVdDWEpYYXVVbFVDWElrYUV0NHFhU2Fv/edit)

[CockroachDB设计文档](https://github.com/cockroachdb/cockroach/blob/master/docs/design.md)