### 一、Paxos协议简介
Paxos算法由Leslie Lamport在1990年提出，它是少数在工程实践中被证实的强一致性、高可用、去中心的分布式协议。Paxos协议用于在多个副本之间在有限时间内对某个决议达成共识。Paxos协议运行在允许消息重复、丢失、延迟或乱序，但没有拜占庭式错误的网络环境中，它利用“大多数 (Majority)机制”保证了2F+1的容错能力，即2F+1个节点的系统最多允许F个节点同时出现故障。
```
拜占庭式错误释义：
一般地把出现故障但不会伪造信息的情况称为“非拜占庭错误”(Non-Byzantine Fault)或“故障错误”(Crash Fault)；而伪造信息恶意响应的情况称为“拜占庭错误”(Byzantine Fault)。
```

#### 1、核心概念

+ Proposal：提案（提案 = 提案编号acceptNumber + 提案值acceptValue）;
+ Proposal Number：提案编号;
+ Proposal Value：提案值;

#### 2、参与角色

+ Proposer（提案者）：处理客户端请求，主动发起提案；
+ Acceptor (投票者)：被动接受提案消息，参与投票并返回投票结果给Proposer以及发送通知给Learner；
+ Learner（学习者）：不参与投票过程，记录投票相关信息，并最终获得投票结果；

在实际的分布式业务场景中，一个服务器节点或进程可以同时扮演其中的一种或几种角色，而且在分布式环境中往往同时存在多个Proposer、多个Acceptor和多个Learner。

#### 3、基础逻辑

Paxos算法是指一个或多个提案者针对某项业务提出提案，并发送提案给投票者，由投票者投票并最终达成共识的算法。
<img src="https://lighthousedp-1300542249.cos.ap-nanjing.myqcloud.com/4915/1.jpg!dtstep" />

”达成共识“过程的特点：
（1）、可以由一个或多个提案者参与；
（2）、由多个投票者参与；
（3）、可以发起一轮或多轮投票；
（4）、最终的共识结果是一个值，且该值为提案者提出的其中某个值；

### 二、Basic Paxos

#### 1、两个阶段

Basic Paxos算法分为两个阶段：Prepare阶段和Accept阶段。

(1). Prepare阶段

该阶段又分为两个环节：
* a、Proposer发起广播消息给集群中的Acceptor发送一个提案编号为n的prepare提案请求。
* b、Acceptor收到提案编号为n的prepare提案请求，则进行以下判断：
  如果该Acceptor之前接受的prepare请求编号都小于n或者之前没有接受过prepare请求，那么它会响应接受该编号为n的prepare请求并承诺不再接受编号小于n的Accept请求，Acceptor向Proposer的响应数据包含三部分内容：接受编号n的提案状态信息，之前接受过的最大提案编号和相应提案值；如果该Acceptor之前接受过至少一个编号大于n的prepare请求，则会拒绝该次prepare请求。

通过以上prepare阶段处理流程可以知道：
* a、prepare请求发送时只包含提案编号，不包含提案值；
* b、集群中的每个Acceptor会存储自己当前已接受的最大提案编号和提案值。

假设分布式环境中有一个Proposer和三个Acceptor，且三个Acceptor都没有收到过Prepare请求，Prepare阶段示意图如下：
<img src="https://lighthousedp-1300542249.cos.ap-nanjing.myqcloud.com/4915/3.jpg!dtstep" />

假设分布式环境中有两个Proposer和三个Acceptor，ProposerB成功发送prepare请求，在发送Accept请求时出现故障宕机，只成功给Acceptor1发送了accept请求并得到响应。当前各个Acceptor的状态分别为：
Acceptor1，同意了ProposerB发送的提案编号2的Accept请求，当前提案值为：orange；
Acceptor2，接受了ProposerB发送的提案编号2的Prepare请求；
Acceptor3，接受了ProposerB发送的提案编号2的Prepare请求；
此时ProposerA发起Prepare请求示意图如下：
<img src="https://lighthousedp-1300542249.cos.ap-nanjing.myqcloud.com/4915/4.jpg!dtstep" />
流程说明：
* a、ProposerA发起prepare(1)的请求，由于该编号小于提案编号2，所以请求被拒绝；
* b、ProposerA发起prepare(3)的请求，该编号大于编号2，则被接受，Accetpor1返回Promised(3,2,'orange')，表示接受编号3的提案请求，并将之前接受过的最大编号提案和提案值返回。
* c、Acceptor2和Acceptor3均返回Promised(3)，表示接受编号3的提案请求。

(2). Accept阶段

如果Proposer接收到了超过半数节点的Prepare请求的响应数据，则发送accept广播消息给Acceptor。如果Proposer在限定时间内没有接收到超过半数的Prepare请求响应数据，则会等待指定时间后再重新发起Prepare请求。

Proposer发送的accept广播请求包含什么内容：
* a、accept请求包含相应的提案号；
* b、accept请求包含对应的提案值。如果Proposer接收到的prepare响应数据中包含Acceptor之前已同意的提案号和提案值，则选择最大提案号对应的提案值作为当前accept请求的提案值，这种设计的目的是为了能够更快的达成共识。而如果prepare返回数据中的提案值均为空，则自己生成一个提案值。

Acceptor接收到accept消息后的处理流程如下：
* a、判断accept消息中的提案编号是否小于之前已同意的最大提案编号，如果小于则抛弃，否则同意该请求，并更新自己存储的提案编号和提案值。
* b、Acceptor同意该提案后发送响应accepted消息给Proposer，并同时发送accepted消息给Learner。Learner判断各个Acceptor的提案结果，如果提案结果已超过半数同意，则将结果同步给集群中的所有Proposer、Acceptor和所有Learner节点，并结束当前提案。
  当Acceptor之前没有接受过Prepare请求时的响应流程图：
  <img src="https://lighthousedp-1300542249.cos.ap-nanjing.myqcloud.com/4915/5.jpg!dtstep" />

当Acceptor之前已存在接受过的Prepare和Accept请求时的响应流程图：
<img src="https://lighthousedp-1300542249.cos.ap-nanjing.myqcloud.com/4915/12.jpg!dtstep" />
该示例中prepare请求返回数据中已经包含有之前的提案值（1,'apple'）和（2,'banana'），Proposer选择之前最大提案编号的提案值作为当前的提案值。

#### 2、关于提案编号和提案值

+ 提案编号

在Paxos算法中并不自己生成提案编号，提案编号是由外部定义并传入到Paxos算法中的。
我们可以根据使用场景按照自身业务需求，自定义提案编号的生成逻辑。提案编号只要符合是“不断增加的数值型数值”的条件即可。
比如：
在只有一个Proposer的环境中，可以使用自增ID或时间戳作为提案编号；
在两个Proposer的环境中，一个Proposer可以使用1、3、5、7...作为其编号，另一个Proposer可以使用2、4、6、8...作为其提案编号；
在多Proposer的环境中，可以为每个节点预分配固定ServerId(ServerId可为1、2、3、4...)，使用自增序号 + '.'  + ServerId或timestamp + '.' + ServerId的格式作为提案编号，比如：1.1、1.2、2.3、3.1、3.2或1693702932000.1、1693702932000.2、1693702932000.3；
每个Proposer在发起Prepare请求后如果没有得到超半数响应时，会更新自己的提案号，再重新发起新一轮的Prepare请求。

+ 提案值

提案值的定义也完全是根据自身的业务需求定义的。在实际应用场景中，提案值可以是具体的数值、字符串或是cmd命令或运算函数等任何形式，比如在分布式数据库的设计中，我们可以将数据的写入操作、修改操作和删除操作等作为提案值。

#### 3、最终值的选择
Acceptor每次同意新的提案值都会将消息同步给Learner，Learner根据各个Acceptor的反馈判断当前是否已超过半数同意，如果达成共识则发送广播消息给所有Acceptor和Proposer并结束提案。
在实际业务场景中，Learner可能由多个节点组成，每个Learner都需要“学习”到最新的投票结果。关于Learner的实现，Lamport在其论文中给出了下面两种实现方式：

（1）、选择一个Learner作为主节点用于接收投票结果（即accepted消息），其他Learner节点作为备份节点，Learner主节点接收到数据后再同步给其他Learner节点。该方案缺点：会出现单点问题，如果这个主节点挂掉，则不能获取到投票结果。
（2）、Acceptor同意提案后，将投票结果同步给所有的Learner节点，每个Learner节点再将结果广播给其他的Learner节点，这样可以避免单点问题。不过由于这种方案涉及很多次的消息传递，所以效率要低于上述的方案。

### 三、活锁问题

#### 1、什么是活锁?
“活锁”指的是任务由于某些条件没有被满足，导致一直重复尝试，失败，然后再次尝试的过程。 活锁和死锁的区别在于，处于活锁的实体是在不断的改变状态，而处于死锁的实体表现为等待（阻塞）；活锁有可能自行解开，而死锁不能。
#### 2、为什么Basic-Paxos可能会出现活锁?
由于Proposer每次发起prepare请求都会更新编号，那么就有可能出现这种情况，即每个Proposer在被拒绝时，增大自己的编号重新发起提案，然后每个Proposer在新的编号下不能达成共识，又重新增大编号再次发起提案，一直这样循环往复，就成了活锁。活锁现象就是指多个Proposer之间形成僵持，在某个时间段内循环发起preapre请求，进入Accept阶段但不能达成共识，然后再循环这一过程的现象。
活锁现象示例图：
<img src="https://lighthousedp-1300542249.cos.ap-nanjing.myqcloud.com/4915/6.jpg!dtstep" />

#### 3、活锁如何解决？
活锁会导致多个Proposer在某个时间段内一直不断地发起投票，但不能达成共识，造成较长时间不能获取到共识结果。活锁有可能自行解开，但该过程的持续时间可长可短并不确定，这与具体的业务场景实现逻辑、网络状况、提案重新发起时间间隔等多方面因素有关。
解决活锁问题，有以下常见的方法：
（1）、当Proposer接收到响应，发现支持它的Acceptor小于半数时，不立即更新编号发起重试，而是随机延迟一小段时间，来错开彼此的冲突。
（2）、可以设置一个Proposer的Leader，集群全部由它来进行提案，等同于下文的Multi-Paxos算法。

### 四、Multi-Paxos
上文阐述了Paxos算法的基础运算流程，但我们发现存在两个问题：
（1）、集群内所有Proposer都可以发起提案所以Basic Paxos算法有可能导致活锁现象的发生；
（2）、每次发起提案都需要经过反复的Prepare和Accept流程，需要经过很多次的网络交互，影响程序的执行效率。
考虑到以上两个问题，能不能在保障分布式一致性的前提下可以避免活锁情况的发生，以及尽可能减少达成共识过程中的网络交互，基于这种目的随即产生了Multi-Paxos算法。

首先我们可以设想一下：在多个Proposer的环境中最理想的达成共识的交互过程是什么样子的？
就是这样一种情况：集群中的某个Proposer发送一次广播prepare请求并获得超半数响应，然后再发送一次广播accept请求，并获得超过半数的同意后即达成共识。但现实中多个Proposer很可能会互相交错的发送消息，彼此之间产生冲突，而且在不稳定的网络环境中消息发送可能会延迟或丢失，这种情况下就需要再次发起提案，影响了执行效率。Multi-Paxos算法就是为了解决这个问题而出现。

Multi-Paxos算法是为了在保障集群所有节点平等的前提下，依然有主次之分，减少不必要的网络交互流程。
Multi-Paxos算法是通过选举出一个Proposer主节点来规避上述问题，集群中的各个Proposer通过心跳包的形式定期监测集群中的Proposer主节点是否存在。当发现集群中主节点不存在时，便会向集群中的Acceptors发出申请表示自己想成为集群Proposer主节点。而当该请求得到了集群中的大多数节点的同意后随即该Proposer成为主节点。
集群中存在Proposer主节点时，集群内的提案只有主节点可以提出，其他Proposer不再发起提案，则避免了活锁问题。由于集群中只有一个节点可以发起提案，不存在冲突的可能，所以不必再发送prepare请求，而只需要发送accept请求即可，因此减少了协商网络交互次数。

### 五、Paxos应用场景示例

上文对Paxos算法的处理流程就行了阐述，为了加深理解，下面以一个分布式数据库的使用案例来阐述Paxos算法在实际业务场景中的使用。

场景描述：分布式数据库中假设包含3个节点，客户端访问时通过轮询或随机访问的方式请求到其中的某个节点，我们要通过Paxos算法保证分布式数据库的3个节点中数据的一致性。
实际的分布式数据一致性流程更为复杂，我这里为了方便阐述将这个过程进行一些简化。

<img src="https://lighthousedp-1300542249.cos.ap-nanjing.myqcloud.com/4915/8.jpg!dtstep" />

分布式数据库中的每个节点都存储三份数据，一是事务日志数据，二是DB数据，三是事务日志执行位置。
事务日志表存储着数据库的操作日志记录，包括：写入Put、修改Update和删除Delete等相关的操作日志，有些文章资料将事务日志表称为状态机其实是一个意思。
DB数据表存储具体的业务数据。
事务日志执行位置用于记录当前节点执行到了哪一条操作记录；

<img src="https://lighthousedp-1300542249.cos.ap-nanjing.myqcloud.com/4915/9.jpg!dtstep" />
整体设计思想：我们只要通过Paxos算法保证各个节点事务日志表数据一致就可以保证节点数据的一致性。

假设，当前各个节点的事务日志表和数据表均为空，现在客户端1对数据库发起写入操作请求：{'Op1','Put(a,'1')'}，这里的Op1代表操作的ID(为了简单起见，直接使用自增ID表示，该数值对应Paxos算法中的提案编号)，Put(a,'1')代表操作内容，对应Paxos中的提案值，假设该请求被随机分配到了Server1处理。

<img src="https://lighthousedp-1300542249.cos.ap-nanjing.myqcloud.com/4915/10.jpg!dtstep" />

流程说明：
1、Server1接受到Put(a,'1')请求，并不是直接写入数据表，而是首先通过Paxos算法判断集群节点是否达成写入共识；
2、当前三个节点的OperateIndex均为0，事务日志表和数据表均为空，Server1的Proposer首先向三个节点发起Prepare(OperateIndex + 1)，即Prepare(1)请求。
3、接收到过半数的Prepare请求反馈后，发送Accept(1,'Put(a,'1')')请求，并得到Accepted请求反馈，则此时三个节点达成共识，当前三个节点的事务日志表均为：{'Op1','Put(a,'1')'}，数据表均为空。
4、达成共识后，Server1执行写入操作并更新当前节点的OperateIndex，此时Server1的OperateIndex为1，其他节点仍为0，Server1的数据表为：a = 1，另外两个节点为空，三个节点的事务日志表相同，当前写入流程结束。

假设，此时Server2节点接收到Put(b,'1')的请求，处理流程如下：
1、Server2接收到Put(b,'1')请求，由于当前Server2的OperateIndex仍为0，则首先发起Prepare(1)的请求，
2、由于当前三个节点的Acceptor的提案编号均为1，所以会拒绝Server2的Prepare(1)请求.
3、Server2未能得到超过半数的prepare响应，则会查看当前事务日志表发现已存在Op1操作，则从当前节点的事务日志表中取出相应操作并执行，然后将当前节点OperateIndex修改为1；
4、Server2随即再次发起Prepare(OperateIndex+1)，即Prepare(2)的请求。
5、此时三个节点达成共识，并更新各自的事务日志表。
6、Server2执行写入操作，此时Server1节点状态为OperateIndex：1，数据表：a=1；Server2节点状态为OperateIndex:2, 数据表：a=1和b=1；Server3的节点状态为OperateIndex：0，数据表为空；三个节点的事务日志表相同，均为：
{'Op1','Put(a,'1')'}；{'Op2','Put(b,'1')'}。当前流程执行结束。

假设，此时Server3接收到Get(a)请求，处理流程如下：
1、Server3接收到Get(a)请求，并不是直接查询数据表然后返回，而是要将当前节点的OperateIndex和事务日志表中的记录进行比对，如果发现有遗漏操作，则按照事务日志表的顺序执行遗漏操作后再返回。
由于Get请求并不涉及对数据的写入和修改，所以理论上不需要再次发起Paxos协商。
2、此时Server1节点的状态为OperateIndex：1，数据表：a=1；Server2的节点状态为OperateIndex:2,  数据表：a=1和b=1；Server3的节点状态为OperateIndex：2，数据表为a=1和b=1；三个节点的事务日志表相同，均为：
{'Op1','Put(a,'1')'}；{'Op2','Put(b,'1')'}。当前流程执行结束。

执行流程示意图如下：
<img src="https://lighthousedp-1300542249.cos.ap-nanjing.myqcloud.com/4915/11.jpg!dtstep" />

随着数据的不断写入，事务日志表的数据量不断增加，可以通过快照的方式，将某个时间点之前的数据备份到磁盘（注意此处备份的是数据表数据，不是事务日志数据，这样宕机恢复时直接从快照点开始恢复，可以提高恢复效率，如果备份事务日志数据，宕机恢复时需要从第一条日志开始恢复，导致恢复时间会比较长），然后将事务日志表快照前的数据清除即可。

### 六、对一些问题的解释

#### 1、投票过程为什么要遵循选择最大提案号的原则？

Paxos投票虽然叫作“投票”，但其实与我们现实中的“投票”有很大的区别，因为它的运算过程中并不关心提案内容本身，而完全依据哪个提案号大就选择哪个的原则，因为只有这样才能达成共识。

#### 2、为什么Proposer每次发起prepare都要变更提案号？

这个问题其实很容易理解，也是为了达成共识。假设ProposerA、ProposerB、ProposerC分别同时发起了prepare(1)、prepare(2)、prepare(3)的提案，而此时ProposerC出现故障宕机，如果ProposerA、ProposerB在后续的每一轮投票中都不变更提案号，那永远都不可能达成共识。

#### 3、为什么Paxos算法可以避免脑裂问题？
Paxos算法可以避免分布式集群出现脑裂问题，首先我们需要知道什么是分布式集群的脑裂问题。
脑裂是指集群出现了多个Master主节点，由于分布式集群的节点可能归属于不同的网络分区，如果网络分区之间出现网络故障，则会造成不同分区之间的节点不能互相通信。而此时采用传统的方案很容易在不同分区分别选出相应的主节点。这就造成了一个集群中出现了多个Master节点即为脑裂。
而Paxos算法是必须达到半数同意才能达成共识，这就意味着如果分区内的节点数量少于一半，则不可能选出主节点，从而避免了脑裂状况的发生。

### 七、开发、运维超实用工具推荐
接下来向大家推荐一款对日常开发和运维，极具有实用价值的好帮手XL-LightHouse。

**一键部署，一行代码接入，无需大数据相关研发运维经验就可以轻松实现海量数据实时统计，使用XL-LightHouse后：**
* 再也不需要用Flink、Spark、ClickHouse或者基于Redis这种臃肿笨重的方案跑数了；
* 再也不需要疲于应付对个人价值提升没有多大益处的数据统计需求了，能够帮助您从琐碎反复的数据统计需求中抽身出来，从而专注于对个人提升、对企业发展更有价值的事情；
* 轻松帮您实现任意细粒度的监控指标，是您监控服务运行状况，排查各类业务数据波动、指标异常类问题的好帮手；
* 培养数据思维，辅助您将所从事的工作建立数据指标体系，量化工作产出，做专业严谨的职场人，创造更大的个人价值；

**XL-LightHouse简介**

* XL-LightHouse是针对互联网领域繁杂的流式数据统计需求而开发的一套集成了数据写入、数据运算、数据存储和数据可视化等一系列功能，支持超大数据量，支持超高并发的【通用型流式大数据统计平台】。
* XL-LightHouse目前已涵盖了常见的流式数据统计场景，包括count、sum、max、min、avg、distinct、topN/lastN等多种运算，支持多维度计算，支持分钟级、小时级、天级多个时间粒度的统计，支持自定义统计周期的配置。
* XL-LightHouse内置丰富的转化类函数、支持表达式解析，可以满足各种复杂的条件筛选和逻辑判断。
* XL-LightHouse是一套功能完备的流式大数据统计领域的数据治理解决方案，它提供了比较友好和完善的可视化查询功能，并对外提供API查询接口，此外还包括数据指标管理、权限管理、统计限流等多种功能。
* XL-LightHouse支持时序性数据的存储和查询。
  GitHub搜索XL-LightHouse了解更多！

如本文有所疏漏或您有任何疑问，欢迎访问dtstep.com与我本人沟通交流！