对于一个采用raft协议的分布式系统，raft实现的正确性至关重要。怎么保证raft实现出来是没问题的？要有的严格测试！对于一个稳定可靠的raft实现，用于测试的代码量应该是高于实现本身的，这可以作为开源实现选型时的参考依据。

那么，假设现在我们自己实现了一个raft，该如何测试它呢？

raft实现一般来讲有三大块，除了核心的算法本身，外围的两块分别是传输层和日志持久化。测试要做的事情，是一层一层的：单元测试，集成测试，错误注入。

# 算法实现的正确性

raft算法的正确性是有理论上证明的，这个具体参考论文。也就是说实现的是正确的，它就是正确的(排除不能满足raft假设的极端情况：消息广播耗时<选举超时<平均故障时间)。

为了方便测试，接口设计一定要清晰。为什么？因为实际上外围模块的error是简直就恶梦，耦合到一起后，就不方便单纯验证算法实现的正确性了。核心算法实现的正确是整个实现正确的基础，所以这个是第一步。

这个[C的实现](https://github.com/willemt/raft)就是一个很好的"接口清晰"例子。它把外围的模块都做成了函数回调，

send_requestvote，send_appendentries这些涉及到网络传输的丢出去，applylog，persist_vote，persist_term这些涉及到落盘持久化的也丢出去，剩下的就是很干净的纯算法实现。测试时只需要mock回调函数的部分。

    __push_entry(sys); //处理整个发包
    __poll_messages(sys); //收包

    // 运行每个raft节点的算法逻辑
    for (i = 0; i < sys->n_servers; i++)
        raft_periodic(sys->servers[i].raft, random() % 200);

    __ensure_election_safety(sys);
    __ensure_log_matching(sys);
    __ensure_leader_completeness(sys);

上面这一段代码可以看作算法实现正确性测试的一个大概骨架，一轮一轮的跑这段代码。发包由包都是mock出去，通过丢到队列和取队列实现，中间可以故意加一些丢包率，`raft_periodic`是raft的算法逻辑，这样子驱动raft逻辑往前走一点点，第二个参数是距离上次调用经过的时间。

最关键的是如何验证，看最后三行代码。选举安全性，日志匹配，leader的完备性，看过论文的就知道这正是论文中正确性证明的一些点，只要验证了这些，正确性就能得到理论上的保证了。mock实现将所有的log都记录下来，上面的函数会检查日志确认条件满足。具体来说，会检查确认：

* 选举安全：一个term内只能选出一个leader
* 日志匹配：如果两份日志中包含term和index相同的记录，那么这两个日志在这条记录之前的记录一定是完全匹配的
* leader完备性：如果在某个term提交了的记录，它一定出现在所有更高term的leader日志中

# 网络层

一个raft的实现，其实纯算法的那部分，只是冰山的一角。真正的复杂性都隐藏在外围模块，比如网络错误处理。建议把raft节点之间的交互抽象出一个传输层接口方便测试。这样可以模拟各种各样的网络错误，方便做错误注入以及测试一些corner case。

我们可以把传输抽象成一个这样的接口：

    type Transport interface {
        Send(RaftMsg) (Result, error)
    }

然后我们模拟各种网络错误。制造网络慢，丢包，包乱序。

我们可以写Transport实现，Send的时候故意给它sleep几秒才再调真实的发送操作...

可以故意的以一定概率随机丢掉其中一些包...

以及把这些情况都组合起来...

一个消息系统，要是能保证 不重、不丢、有序，那世界真是太美好了。可是测试，就是要模拟最恶劣的现实！这正是错误注入。

这里我们思考一下：模拟包乱序是否必要。

TCP是保证可靠性的，但业务层不是。假设一台机器与其它闪断了，它在业务层不停地重试发包多次，然后网络又恢复了。呐，对端就会收到重复的包。再比如说，一台机器A给另一台B发消息，B收到了又down掉，再起来时A以为B没收到，又给它发较早的包，那B收到的消息就乱序的。这里只是举些例子，也就是说，TCP层由TCP保证，业务层要由业务层保证。对于raft的实现，这些场景都需要测试的。

# 存储层

相比于网络层各种异常的普遍性，存储层的错误更加极端一些。但是真正健壮的raft实现，是需要测试这些的。这一块的模拟，建议在做持久化操作的时候插入一些hook。故意给调用返回一些错误，比如磁盘满了，磁盘挂了，无响应了。

通过故意插入这些hook，可以测试出实现对于各种error处理的是否健壮。比如一些代码的极端错误处理，那些写个TODO就不管了的，对于那亿分之一的概率，可能没事，但是通过这么一测，就傻逼了。

模拟突然断电了，是否会丢状态。重启时数据是否能恢复，是否一致等等。

# 关于集成测试以及其它

假设我们实现了一些更上层的业务，我们可以测试raft节点随便杀，是否无影响。集成测试是挺考验整个方方面面的，不仅仅是某一个点。工程不像算法，会有很多很多细节。比如说client是否自动重连，是否会尝试其它节点。比如说，由于raft的复杂性，通常可能是在单个线程里面去实现，而snapshot同步的时候可能就不能响应请求，那别人以为它挂了，就去重新选举之类。

单元测试一定要有，并且测试覆盖率要达到一定的比例。所有的测试都应该是可自动化的。项目要做持续集成和回归测试，不通过CI的代码不能合并。

市面上做分布式存储的产品，哪些敢说自己做了这种级别的测试？尤其是国内吹吹太多，没敢开源的根本不要瞎BB嘛。

关于Raft的实现，C的可以看看[这个](https://github.com/willemt/raft)，挺干净的实现，也有严格的[正确性测试](https://github.com/willemt/virtraft)。缺点是没有实现log压缩相关，另外，也没有外围的模块。更适合于作为纯粹算法实现的学习读一读。

js的可以看一看[这个](https://github.com/ongardie/raftscope)，优点是它是[可视化](http://raft.github.io/raftscope/)的。

如果是Rust，安利一下我们刚开源的[tikv](https://github.com/pingcap/tikv/tree/master/src/raft)，封装得还可以，基本上可以拿出去当一个caret使用。
