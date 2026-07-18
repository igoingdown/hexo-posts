---
style: summer

title: 分布式锁的边界：从 etcd 与 Redis 之争，到 fencing token 与 CAS

date: 2026-07-19 15:30:00

tags:
  - 分布式
  - 分布式锁
  - etcd
  - Redis
  - Redlock
  - fencing token
  - CAS
  - 一致性
  - Jepsen
---

这篇文章源于我对 [《etcd 以及 redis分布式锁的实现优劣比较》](https://blog.pdjjq.org/post/etcd-and-redis-distributed-locks-realize-the-advantages-and-disadvantages-z17y7uy) 这篇博客的一次深入学习。和上一篇 [《再谈 CAP》](/2026/07/06/再谈CAP-从Redis-etcd-MongoDB看透分布式一致性/) 一样，我从原博客出发，顺着一连串追问走完了一条完整的学习路径。这次我把这条路径原样保留成 **Q&A 对话体**——因为每一个后续问题都是被前一个答案逼出来的，这个"被逼出来"的过程本身就是内容：

1. 原博客讲了什么？"强一致场景用 etcd、性能场景用 Redis"这个结论站得住吗？
2. etcd 的 bug #11456 修复之后，为什么在进程暂停/消息延迟下依然无法保证严格互斥？要严格互斥该怎么办？是不是压根没有真正的方案？
3. 既然锁、fencing token、CAS、事务都需要，它们到底各自负责什么？
4. 既然锁只管效率，绝大多数场景是不是 SETNX 就够了？MySQL 的 version 列能不能替代 etcd 的 fencing token？
5. "跨系统统一发号"的真实例子是什么？电商跨领域的一致性问题该怎么解？换 etcd 会更好吗？

每一节都遵循同一个套路：**先给抽象结论，再给具体机制，最后用同一个"SKU-8848 库存扣减"的示例串起来**。全文的技术论断都核对过一手资料（etcd 官方文档与 concurrency 包源码、Jepsen 对 etcd 3.4.3 的正式分析、Redis 官方分布式锁文档、antirez 的原始 Redlock 规范、Kleppmann 与 antirez 的公开论战），关键处附原文引用。

<!-- more -->

# Q1：原博客讲了什么？它的结论站得住吗？

## 原博客的内容

原博客很短，论证链只有三步：

1. **Redis 是 AP**：哨兵和切片集群两种部署模式都选择了可用性 + 分区容忍，故障转移期间客户端可能读到未同步的旧数据，只能保证最终一致；
2. **etcd 是 CP**：基于 Raft 保证线性一致性，网络分区凑不齐多数派时宁可停止服务也不牺牲一致性；
3. **选型结论**：业务对锁的要求非常严格（任何情况下不允许多实例同时持锁，哪怕没有实例持锁）且并发要求不高 → etcd/zk；业务有幂等兜底、性能优先 → Redis。

另外它给出了分布式锁的三个基本特征：**互斥性/安全性**（绝不允许多个 client 同时获得锁）、**受限存活**（client 崩溃或分区时锁要能自动过期释放，避免死锁）、**高性能/高可用**。

## 我的评价：结论碰巧是对的，但理由不对、也不全

先说对的部分。它的 CAP 定性准确——Redis Sentinel/Cluster 的异步复制确实是最终一致，etcd 默认对除 watch 外的所有操作提供线性一致性（[etcd 官方 API guarantees](https://etcd.io/docs/v3.5/learning/api_guarantees/) 逐字可查）。最终的选型方向也与 Kleppmann 本人的建议一致：仅为提效可以用单实例 Redis，对正确性有硬要求应该用 ZooKeeper 类共识系统。

但它有一个**站不住脚的隐含论断**：只要肯牺牲可用性，etcd 锁就能做到"任何情况下都不允许多个实例获取到锁"。这与 etcd 官方认可的 [Jepsen 2020 年测试报告](https://jepsen.io/analyses/etcd-3.4.3) 直接矛盾：

> "etcd locks (like all distributed locks) do not provide mutual exclusion — multiple processes can hold an etcd lock concurrently, even in healthy clusters with perfectly synchronized clocks."
> （etcd 锁——和所有分布式锁一样——不提供互斥：即使在时钟完全同步的健康集群中，多个进程也可能同时持有同一把 etcd 锁。）

Jepsen 实测发现，lease TTL 设为 1～3 秒时，健康集群跑几分钟就能复现互斥失效。其中一部分是具体实现 bug（[issue #11456](https://github.com/etcd-io/etcd/issues/11456)：服务端把锁交给排队的下一个客户端之前，没有重新校验它的 lease 是否还有效），但 etcd 维护者在官方回应里说得很清楚：bug 修完，结构性的局限依然在。

它还有**两个关键遗漏**：全文没有出现 Redlock（Redis 分布式锁的"正经"多实例算法，N 个独立 master、多数派、`MIN_VALIDITY = TTL - (T2-T1) - CLOCK_DRIFT`），也没有出现 fencing token——而这恰恰是 Kleppmann、antirez、Redis 官方文档三方论战之后沉淀下来的共识性方案。[Redis 官方文档](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/) 罕见地在正文里点名引用了 Kleppmann 的批评和 antirez 的回应，并直接建议"你应该实现 fencing token"。

一句话总结这一节：**原博客让读者以为"选 etcd = 绝对安全，选 Redis = 拿性能换风险"，而真相是两者都不能单独保证正确性——这就逼出了下一个问题。**

# Q2：bug 修完了，为什么 etcd 还是保证不了严格互斥？到底怎么办？

## 先给结论

#11456 修复的是"实现细节"，修不掉的是"模型假设"。**互斥性和受限存活在数学上是一对矛盾**：

> 在一个消息延迟无上界、进程暂停无上界的异步网络里，任何只依赖超时来判断"对方是否还活着"的机制，都无法 100% 正确地区分"对方真的死了"和"对方只是很慢"。

etcd 的 lease 到期，只能证明"etcd 在过去 TTL 秒内没收到续约"，**不能**证明"那个客户端已经停止访问共享资源"。GC Stop-The-World、虚拟机迁移、宿主机 CPU 抢占、网络抖动，都能让一个客户端在应用层"冻结"几秒——它自己的代码逻辑完全没变，它以为自己还持有锁，一旦恢复就继续往下执行。而 etcd 那边 lease 早已过期，锁已经判给了下一个排队者。这两件事——"客户端主观认为自己持锁"和"etcd 客观上已把锁给了别人"——在纯异步模型下无法被任何一方可靠检测。这不是 etcd 的缺陷，Redis/Redlock、ZooKeeper 的 ephemeral node 全部同理，只是窗口大小和触发概率不同。

## 示例：SKU-8848 的第一次双写

服务器 A 持有 etcd 锁（lease TTL 2 秒）正在扣减 SKU-8848 的库存，被宿主机 CPU 争抢卡住 3 秒。这 3 秒里 etcd 判定 A 失联，把锁给了服务器 B，B 完成扣减并释放。A 醒来后并不知道自己已经"过期"，继续执行扣减——**两次扣减都发生了，超卖**。注意：A 手里的锁是 etcd 发的还是 Redis 发的，对这个事故没有任何影响。

## 那怎么办：把裁决权从锁挪到资源

**第一层：fencing token。** 拿锁的同时拿一个严格单调递增的编号（etcd 的 revision 号天然就是），真正写资源时把编号带过去，让**资源自己**校验："这次请求的编号是否比我上次接受的更大？不是就拒绝。" A 带着编号 100 迟到，资源已见过 B 的 101，100 < 101，拒绝。

**第二层：token 校验必须和写入焊成原子操作。** 有人用形式化验证工具（FizzBee）严格建模后发现 fencing token 有一个真实的缝隙：如果旧编号的请求**先于**新编号到达资源，资源当时没见过更大的编号，没有理由拒绝它——两次写入都会成功。所以 token 比较不能"先查后写"分两步做，必须用资源层的原子条件写（CAS）一步完成。

**第三层：对无法条件写的操作（调用不幂等的第三方接口、发短信），只能靠业务幂等键/去重表兜底。**

## 直接回答"是不是压根没有真正的方案"

分两层：

- 如果"真正的方案"指**纯粹靠锁本身**在任意进程暂停/消息延迟下 100% 保证互斥——**确实没有**，这是异步分布式系统理论（FLP 类不可能性）的硬性结论，不是工程能力问题。任何声称做到的方案，要么偷偷加了同步性假设（"进程暂停不超过 X 秒"），要么是错的。
- 如果指**工程上把风险压到可接受、可审计、可纠正**——完全做得到，且是业界标准实践：短锁窗口降概率 + fencing token 把裁决权交给资源 + 资源层原子条件写兜底 + 业务幂等/对账做最后防线。四层叠加后双写概率可以压到极低，但理论非零——这也是为什么大厂的账务、库存系统即便用了 etcd/zk，对账体系依然是标配。

# Q3：锁、fencing token、CAS、事务，到底各自负责什么？

## 先给抽象框架：两层不同性质的问题

"库存和资源的互斥"看似一个问题，拆开是两层：

- **调度层的互斥**：真正触碰资源之前，减少无意义的并发争抢，并保证不死锁。本质是**效率和存活性**。
- **结果层的互斥**：不管调度层的假设（"持锁者还活着"）是否失效，最终写入必须不可分割、顺序可追溯。本质是**安全性**，且只有这一层能给出数学上站得住的保证。

分布式锁天生只能解决第一层；fencing token、CAS、事务都在补第二层。

| 机制 | 它回答的问题 | 它不能回答的问题 |
|---|---|---|
| **分布式锁** | "现在轮到谁去处理这个资源？处理者失联多久后换人？" | "换人之后，原持有者的操作还会不会生效？"——锁只能观察续约，观察不到对方的动作 |
| **fencing token** | "这次写请求是不是来自已被取代的过时持有者？" | "如何让这个判断和写入不可分割？"——先查后写中间仍有竞态 |
| **CAS** | "把'检查 token'和'执行写入'合并成一次不可分割的操作" | "涉及多张表/多个资源时怎么整体生效或整体不生效？" |
| **事务** | "把 CAS 的原子性范围从单行扩展到一组关联对象，整体可回滚" | "链路里有外部不可回滚的副作用（已发出的短信、已扣款的第三方支付）怎么办？"——只能业务幂等/补偿 |

四者不是四选一，而是**从"谁先来"到"最终写没写对"层层递进的流水线**，每一环挡住的失败模式都不同：没有锁 → 海量并发怼到资源层拼 CAS，大量冲突重试，纯浪费；有锁没 token → 资源层无法识别迟到的旧持有者；有 token 没 CAS → 两个并发都读到"我的编号更大"然后都写成功；有 CAS 没事务 → 单行安全，但"扣了库存没建订单"的半成品出现。

## 示例：SKU-8848 的完整时间线

- **t0**：A 请求加锁，etcd 按 revision 排队判定 A（revision=100）持锁。——**锁在起作用**：B 的请求在 etcd 里排队，而不是直接冲向数据库。
- **t1**：A 读到库存行 `fence_token=99`，100 > 99，可以往下走（内存比较，未写库）。
- **t2**：A 被 CPU 抢占卡住 3 秒。
- **t3**：etcd 2 秒没收到续约，lease 过期，锁转给 B（revision=101）。——**锁的"受限存活"在起作用**：没有它 B 永远拿不到锁；代价是 A 不知道自己已出局。
- **t4**：B 执行 `UPDATE inventory SET stock=stock-1, fence_token=101 WHERE sku='8848' AND stock>=1 AND fence_token<101`，成功。——**CAS 在起作用**：检查条件和更新是 MySQL 单条语句内的原子操作，这是整条链路里唯一"数学上不可分割"的动作。
- **t5**：A 醒来执行自己那条 SQL（`fence_token<100`），此时 `fence_token=101`，WHERE 匹配不到任何行，影响行数 0。——**fencing token 在起作用**：etcd 拦不住一个已经在跑的进程，但"A 已过期"这个事实被编码成了数据库能在写入瞬间校验的条件。
- **t6**：如果这次操作还要同时建订单、核销优惠券，把三个写包进同一个数据库事务。——**事务在起作用**：把 t4 那次 CAS 的原子范围从一行扩展到一组行。

一眼可以看穿的分工：**锁负责 t0→t3 这段"谁先谁后、掉线换人"的调度，它的失效边界正是 t3→t5 之间的缝隙；fencing token + CAS 负责在 t4/t5 真正落笔的瞬间把缝隙缝上。**

## 反问：既然 CAS 才是兜底，锁能不能干脆不要？

单纯"扣库存"这种一条 SQL 的操作，确实可以不要（很多秒杀系统就是纯数据库乐观锁）。但现实业务流程往往是多步骤的：读多张表算业务规则、调风控/支付等外部接口、耗时几百毫秒到几秒。如果 1000 个并发都各自跑完整套流程再在最后一步拼 CAS，999 个会"跑到最后才发现白干"。**锁的价值是把整个多步骤流程序列化，让绝大多数请求在最早期就排队或快速失败，减少浪费性并发。** 这是纯粹的效率问题——正确性依然由流程末尾的 CAS/事务兜底。

# Q4：绝大多数场景是不是 SETNX 就够了？MySQL 的 version 列能替代 etcd 吗？

## SETNX 够不够，取决于锁是不是"最后一道防线"

既然锁只管调度效率、正确性靠资源层 CAS 兜底，那锁自己犯点小错就无所谓：Redis 主从故障转移导致两个客户端同时以为持锁，后果只是**两边都跑到数据库门口**，然后只有一个能通过 CAS 写进去，另一个影响行数 0、走重试或返回"库存不足"。这跟 etcd lease 过期后两个客户端跑到数据库门口的处境，**从数据库视角看完全一样**——数据库不关心你是被谁的锁放进来的。

所以：**只要资源层老老实实做了 CAS/乐观锁，锁用 SETNX 还是 etcd Mutex 对正确性没有任何影响，差别只在浪费掉的并发量。** 而 Redis 的性能、部署成本、运维熟悉度全面占优——电商秒杀、限流、防重复消费这些场景，现实中就是 `SET key value NX PX` + TTL 打天下。

**锁本身的强弱什么时候才真正重要？只有一种情况：锁下游没有能做 CAS 的资源层。**

- leader election 式的锁（"同一时刻只能有一个进程跑这个定时任务"）——锁本身就是唯一裁决者，没有第二道防线；
- 调用不支持幂等、不支持条件写的第三方接口——没有 `WHERE version=xxx` 可用；
- 服务发现、配置分发——这本来就是 etcd/zk 的主场，不是"锁"问题。

判断标准不是"这把锁重不重要"，而是"**下游有没有别的机制能兜底**"。

## MySQL 的 version 列本身就是 fencing token

Fencing token 的本质要求只有一条：**严格单调递增 + 资源层能对它做原子的比较-写入**。MySQL 自己完全能生成并维护这个数字：

```sql
-- etcd revision 版：token 由外部权威分配
UPDATE inventory SET stock = stock - 1, fence_token = 101
WHERE sku = '8848' AND stock >= 1 AND fence_token < 101;

-- MySQL version 版：token 由资源自己维护，效果等价
UPDATE inventory SET stock = stock - 1, version = version + 1
WHERE sku = '8848' AND stock >= 1 AND version = 100;
```

两条 SQL 本质是同一件事。对"资源就是数据库一行"的场景，**version 列更简单、无外部依赖、无额外网络往返，完全不需要 etcd**。

外部统一发号（etcd/zk）真正不可替代的只有两种情况：

1. **被保护的资源不是数据库行**——写文件、操作物理设备，没有 SQL 行可挂 version；
2. **需要跨多个异构系统保持同一套顺序**——多个下游必须对"谁的操作更新"达成一致判断，各自维护的编号互不相识，没法比较。

第二种听起来抽象，于是有了最后一问。

# Q5："跨系统统一发号"的真实例子？电商跨领域一致性怎么办？etcd 会更好吗？

## 真实例子在基础设施的控制面，不在电商里

这个场景的准确画像：**一个可能随时被更换的"权威身份"（leader/primary），要往多个互相不认识的下游写数据；身份一旦更换，所有下游必须一致地拒绝旧身份的迟到写入。**

**例子一：Kafka 的 controller epoch。** Kafka 的 controller 由 ZooKeeper（新版 KRaft）选举，它要向几十上百个 broker 下发指令。旧 controller GC 卡死被替换后醒来继续发指令，broker 们必须一致拒绝。做法：每次 controller 变更 epoch 加一（由 ZooKeeper/KRaft 这个**单一权威**维护），指令都带 epoch，broker 只接受不小于已见最大值的指令。多个 broker 就是"多个互不通信的下游"，它们不可能各自维护 version——那样对"旧 controller 是否过期"的判断会不一致。

**例子二：HDFS NameNode 主备切换的 epoch fencing。** 新 active NameNode 从协调服务拿到更大的 epoch，先广播给所有 JournalNode，此后旧 active 带旧 epoch 的写入被所有 JournalNode 一致拒绝。

共同点：**要保护的不是一行业务数据，而是"谁是合法写入者"这个身份本身，且多个下游必须对身份新旧达成一致。**

## 电商的"订单+库存+优惠券"是另一类问题：分布式事务

支付完成 → 改订单状态、扣库存、核销优惠券，三个系统。它和上面例子的本质区别：**这里没有"可能被替换的权威写入者"，三个系统不需要对任何身份的新旧达成一致。** 各系统防的是自己域内的问题：订单用状态机（`WHERE status='待支付'`）防重复回调，库存用幂等键 + 条件更新防重复扣减，优惠券校验券状态——全是 Q3/Q4 讲过的资源自带 CAS。

真正的难题是**原子性**：三个操作分布在三个数据库，怎么保证要么都成功要么都不生效。这是教科书意义上的分布式事务，而 **etcd 对此无能为力**——它能排序、能选主，但没有任何机制能让三个独立的 MySQL 同时提交或回滚。就算发一个全局序号，半成品状态照样发生，因为失败出在某个服务自己的提交环节，与序号无关。

业界的标准答案（字节、阿里都是这套）是**放弃强原子性，改用最终一致 + 可补偿**：本地消息表/Transactional Outbox 把"支付成功"和待发消息在同一个本地事务里原子落库，MQ 可靠投递给下游幂等消费；Saga/TCC 拆成一串本地事务、失败走补偿（库存加回去、券恢复）；对账定时修复中间状态做最后兜底。**不同领域服务自治、RPC/消息互联、域内靠状态机和幂等、跨域靠事件+补偿+对账——这不是妥协，而是对"跨库原子性在大规模下不可行"的正确回应。**

## 硬上 etcd：一致性收益约等于零，性能是灾难

一致性上：缺的是跨库原子提交能力，etcd 不提供；每笔订单的操作天然以 order_id 为边界隔离，不同订单之间根本不需要全局排序。性能上：etcd 每次写入都走 Raft 共识（fsync + 多数派往返），整个集群写吞吐在**万级 QPS** 且不可水平扩展；电商大促峰值是几十万到百万级 QPS——把 etcd 放进订单关键路径，整个系统的吞吐上限被压掉两三个数量级。

反过来验证：etcd/zk 在这些公司确实被大量使用，但全部用在**低频、小数据量的控制面**——服务发现、配置中心、选主；没有人把它放到交易这种数据面的每请求路径上。

> **一句话收拢**：etcd 这类协调服务解决控制面问题（谁是 leader、谁的身份过期、集群成员是谁），低频但要求强一致；电商交易是数据面问题，高频但允许短暂中间状态，正确性靠每个资源自带的状态机/CAS/幂等，跨系统靠消息+补偿+对账。"etcd 一致性好但性能差，到底用不用"之所以纠结，是因为拿控制面的工具去审视数据面的问题——放回各自领域，答案是清晰的。

# 全文总结

把五问串成一个闭环：

1. **原博客的选型结论方向没错，但论证是残缺的**：它用 CAP 标签替代了真正的分析，隐含的"etcd 锁绝对互斥"不成立（Jepsen 实锤），且完全没提 Redlock 和 fencing token 这两块业界共识的核心拼图。
2. **互斥性与受限存活存在结构性矛盾**：只要为了防死锁引入超时自动释放，就必然存在一个时间窗口，其中多个持有者可能同时"认为"自己持锁。这个矛盾对 etcd、Redis、ZooKeeper 一视同仁，修 bug 修不掉。
3. **正确性的裁决权必须从锁挪到资源**：fencing token 把"过时"变成资源层可校验的信息，CAS 把校验和写入焊成原子动作，事务把原子范围扩展到一组关联数据，业务幂等兜住不可回滚的副作用。锁只负责调度效率。
4. **绝大多数业务场景：Redis SETNX 做调度 + MySQL 事务内的 CAS（version 列）做兜底，就是完整答案**。version 列本身就是 fencing token，不需要外部发号。etcd 的号只有在"没有资源行可挂 version"（leader election）或"跨异构系统统一顺序"（Kafka controller epoch、HDFS epoch fencing 这类基础设施控制面）时才不可替代。
5. **跨领域的一致性是分布式事务问题，不是锁问题**：电商场景下强一致不可行也不必要，最终一致（领域事件 + 重试 + 对账 + 回补）是正确答案。这个 topic 值得单独展开，本文不再深入。

# 参考

- 原博客：[etcd 以及 redis分布式锁的实现优劣比较](https://blog.pdjjq.org/post/etcd-and-redis-distributed-locks-realize-the-advantages-and-disadvantages-z17y7uy)
- Jepsen: [etcd 3.4.3 analysis](https://jepsen.io/analyses/etcd-3.4.3) / etcd 官方回应: [Jepsen 343 results](https://etcd.io/blog/2020/jepsen-343-results/)
- etcd: [issue #11456 — Locks return without checking whether the lease is still held](https://github.com/etcd-io/etcd/issues/11456)
- [etcd: API guarantees](https://etcd.io/docs/v3.5/learning/api_guarantees/) / [etcd: KV API](https://etcd.io/docs/v3.6/learning/api/) / [clientv3 concurrency 包](https://pkg.go.dev/go.etcd.io/etcd/client/v3/concurrency)
- [Redis: Distributed Locks with Redis（含 Redlock 与"Analysis of Redlock"）](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- Martin Kleppmann: [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- antirez: [Is Redlock safe?](http://antirez.com/news/101)
- Lorin Hochstein: [Locks, leases, fencing tokens, FizzBee!](https://surfingcomplexity.blog/2025/03/03/locks-leases-fencing-tokens-fizzbee/)（fencing token 的形式化验证与缝隙）
