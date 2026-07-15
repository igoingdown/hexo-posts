---
style: summer

title: 再谈 CAP：从 Redis、etcd、MongoDB 看透分布式一致性

date: 2026-07-06 15:30:00

tags:
  - 分布式
  - CAP
  - Redis
  - etcd
  - MongoDB
  - Raft
  - 分布式锁
  - 一致性
---

这篇文章源于我对 [《分布式理论: 再谈CAP》](https://blog.pdjjq.org/post/distributed-theory-talk-about-cap-again-12jnyp) 这篇博客的一次深入学习。我按照"先抽象后具体、先总结后细化"的方式，从原博客出发，顺着一连串追问走完了一条完整的学习路径：

1. 原博客到底讲了什么？（CAP 是滑块，不是选择题）
2. Redis Cluster 里 SETNX / Lua 脚本的"原子性"是怎么实现的？它和 AP 选型是什么关系？
3. Kleppmann 与 antirez 关于 Redlock 的著名论战，到底在吵什么？
4. etcd / Raft 是怎么做到"已确认的写入永不丢失"的？
5. "用不用一致性哈希"跟 AP 取向有关系吗？
6. MongoDB 的可调一致性是怎么回事？
7. 读写关注（writeConcern / readConcern）组合爆炸怎么选？在阿里云上落地有哪些坑？

每一节都遵循同一个套路：**先给抽象框架，再给具体机制，最后用同一个"银行账户 / 电商订单"的示例串起来**。全文的技术论断都核对过一手资料（Redis 官方规范、Raft 论文、etcd / MongoDB 官方文档、Jepsen 报告、阿里云官方文档），关键处附原文引用。

<!-- more -->

# 一、原博客讲了什么：CAP 不是选择题，是滑块

## 1.1 教科书版 CAP

在一个多节点、共享数据的分布式系统中，读写操作最多同时满足以下两项：

- **C（一致性）**：特指**强一致性 / 线性一致性**——写操作一旦返回成功，之后任何节点上的任何读操作都必须读到这个新值。整个集群表现得"像一台单机"。
- **A（可用性）**：每一个请求都要在**有限时间内**得到**非错误的返回结果**，且是 100% 的请求都如此。
- **P（分区容忍）**：网络把集群分隔成互相不通的几块时，系统仍能继续工作。

## 1.2 原博客的推理链

原博客的核心论点是：**"三选二"这个说法在工程上过于粗糙**。它的推理分四步：

1. **P 没得选**。现实网络永远存在"不可靠的网络"和"概率性的宕机"，分区一定会发生。于是"三选二"退化为：**分区发生时，在 C 和 A 之间二选一**（CP 还是 AP）。
2. **A 这一端被"软化"**。100% 可用是伪命题：就算集群本身 100% 可用，客户端是通过网络访问它的（博客举例：K8s 里 etcd 通过 Service 暴露，连通性依赖 Service 的可用性），这段链路会直接拉低外部实际观测到的可用性。工程上追求的从来是**高可用**（若干个 9）+ 调用方的**重试、重连、降级**兜底。
3. **C 这一端也被"软化"**。一致性不是开关而是梯度：**强一致 > 弱一致 > 最终一致**。所谓 AP 系统"放弃了 C"，其实只是把 C 降级到"最终一致"，一致性保证依然存在。
4. **推论**：既然 A 和 C 两端都是连续可调的，CAP 就变成了一个**滑块**。**BASE 理论**（基本可用 Basically Available + 软状态 Soft state + 最终一致 Eventually consistent）就是把滑块推向 AP 一端时的系统化方法论——BASE（碱）这个名字就是故意跟 ACID（酸）对着起的。

> 这个观点和学术界的 PACELC 定理、Brewer 本人 2012 年《CAP Twelve Years Later》的修正一脉相承："三选二"是被误传的简化，真正的权衡只在分区发生的那一刻才出现，且是程度问题而非有无问题。

## 1.3 贯穿全文的示例：一个银行账户

一个余额服务部署在 3 个节点（N1、N2、N3）上，每个节点都有副本，你的余额是 100 元。

**正常运行时**：向 N1 存入 50 元，N1 同步给 N2、N3 后返回成功，之后从任何节点读都是 150。C 和 A 都满足——**CAP 的矛盾只在分区发生时才爆发**。

**分区发生时**：N3 与 N1/N2 断连，你恰好连在 N3 上存钱，N3 面临抉择：

- **选 C（放弃 A）**：N3 说"我联系不上多数节点，拒绝服务"。请求失败，但数据绝不打架——这是 **etcd** 的做法；
- **选 A（降级 C）**：N3 说"行，先记下"，两边数据分叉，等网络恢复后再收敛（最终一致）——这是 **Redis Cluster** 的路线。

原博客最后用三个系统印证：偏 AP 的 Redis Cluster（哈希槽 + gossip）、架构居中的 MongoDB Sharding（shards / mongos / config servers）、偏 CP 的 etcd（Raft）。下面几节就是对这三个系统的逐一深挖。

# 二、Redis Cluster：原子性与 AP，是两根不同的轴

第一个追问：**为什么 SETNX、Lua 脚本能实现"整个集群的原子性"？这和 AP 选型有什么关系？**

这个问题里埋着一个几乎所有人都会踩的概念陷阱。先把结论亮出来：

> **SETNX 和 Lua 脚本从来没有实现过"整个集群的原子性"，它们实现的是"单个节点上的原子性"。Redis Cluster 的高明之处在于：它通过路由设计，把"集群原子性"这个难题退化成了"单机原子性"——不是解决了跨节点原子性，而是绕开了它。**

而且，"原子性"和 CAP 里的"一致性"是两个完全不同的东西，对抗的是两个不同的敌人：

| | 原子性（SETNX / Lua） | CAP 一致性（AP vs CP） |
|---|---|---|
| 对抗的敌人 | **并发**：多个客户端同时操作互相踩踏 | **故障**：宕机、分区导致副本数据打架 |
| 发生的范围 | 单个 master 节点内部 | master 与 replica 之间、分区两侧之间 |
| Redis 的答案 | 单线程执行，天然解决 | 异步复制，**故意不解决**（这就是 AP 选型） |

## 2.1 三块基石

**基石 1：单线程 → 每条命令天然原子。** Redis 命令执行是单线程排队的（Redis 6 的 I/O 多线程只管读写网络数据）。SETNX 的"检查 + 写入"两步之间不可能被插队。原子性没有魔法，就是"排队 + 不许插队"。

顺带一提：Redis 原生恰恰**没有**通用的 CAS 命令（没有"如果值等于 X 就改成 Y"），正是这个空缺，才轮到 Lua 登场。

**基石 2：Lua → 把多条命令打包成一条。** [Redis 官方文档](https://redis.io/docs/latest/develop/programmability/eval-intro/) 的原话非常斩钉截铁：

> "Redis guarantees the script's atomic execution. While executing the script, all server activities are blocked during its entire runtime."
> （Redis 保证脚本的原子执行。脚本运行期间，服务器的所有其他活动全部被阻塞。）

实现手段还是那个土办法：阻塞整个服务器。Lua 没有引入锁、没有 MVCC，只是让单线程跑完你的整段脚本之前不理任何人。

**基石 3：Cluster 路由 → 把集群问题退化成单机问题。** 每个 key 通过 `HASH_SLOT = CRC16(key) mod 16384` 映射到唯一的槽；集群稳定时一个槽只由一个 master 服务；**多 key 操作（含 Lua）强制要求所有 key 在同一个槽**，否则报 `CROSSSLOT` 错误。配套工具是 **hash tag**：key 里写 `{...}` 就只对花括号内的部分做哈希，`{order:42}:lock` 和 `{order:42}:status` 必然落在同一台机器。

把三块基石拼起来：**Redis Cluster 不是"实现了跨节点原子操作"，而是从制度上禁止了跨节点操作。** 解决问题的最高境界是让问题不存在。

## 2.2 示例：分布式锁的生与死

**加锁**（SETNX 的原子性在干活）：

```
SET lock:order:42 "instance-A-token" NX PX 30000
```

实例 A 和 B 同一毫秒发出这条命令，在 key 所属的 master N2 的单线程里排队：A 成功，B 返回 nil。互斥达成——注意全程只有 N2 一台机器参与。

**解锁**（为什么必须 Lua）：不能直接 DEL——如果 A 处理超时、锁已过期被 B 拿走，A 的 DEL 会删掉 B 的锁。正确语义是"值还是我的 token 才删"，而 Redis 没有这条原生命令，GET 完再 DEL 中间又有缝隙。Lua 补上：

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

**灾难现场**（AP 的账单到期）：[Redis Cluster 官方规范](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/) 原话：

> "Because of the use of asynchronous replication, nodes do not wait for other nodes' acknowledgment of writes."
> "Usually there are small windows where acknowledged writes can be lost."
> （由于使用异步复制，节点不等待其他节点对写入的确认。……通常存在一些小的时间窗口，已确认的写入可能丢失。）

时序：A 加锁成功，N2 返回 OK → 复制包还在网线上飞，N2 宕机 → replica 被提升为新 master，它的数据里**没有这把锁** → B 加锁成功 → **A 和 B 同时持锁**。

请仔细品：**没有任何环节违反原子性**——旧 master 上的 SETNX 是原子的，新 master 上的也是。坏掉的是跨副本的一致性。SETNX/Lua 解决的是"并发"这个敌人，而杀死这把锁的是"故障"这个敌人——AP 选型意味着 Redis 故意不为后者买单。

一个帮助记忆的类比：SETNX/Lua 像一位公证员，在他的办公室里所有文件签署绝对严格有序、绝无插队（原子性）；但这间办公室的档案是**事后异步**抄送到备份处的（异步复制）——办公室失火时，最后几份刚签完、还没抄送的文件就永远消失了，哪怕当事人已经拿到了"签署成功"的回执。

# 三、Redlock 论战：Kleppmann vs antirez

上面那个"锁在 failover 时失效"的场景，正是 2016 年分布式系统圈一场著名论战的主角。

## 3.1 起因

为了解决单实例/主从 Redis 锁的丢锁问题，Redis 作者 antirez 设计了 **Redlock**：部署 5 台完全独立的 Redis（无主从无复制），加锁时向 5 台同发 `SET NX PX`，在**多数派（≥3 台）成功且总耗时小于锁有效期**才算持锁。antirez 在文档里写了"欢迎专家来分析"。正在写《Designing Data-Intensive Applications》（DDIA）的 **Martin Kleppmann** 应约写了 [《How to do distributed locking》](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)——他自述"这个算法本能地在我脑中拉响了警报"。第二天 antirez 发表反驳 [《Is Redlock safe?》](http://antirez.com/news/101)。

## 3.2 Kleppmann 的两记重拳

**第一拳：先问锁保什么——效率还是正确性？** 判断方法一句话："锁偶尔失效了，会死人吗？"

- **效率锁**：失效后果 = 重复干活，多付 5 美分账单、用户收到两封重复邮件。无所谓。
- **正确性锁**：失效后果 = 数据永久不一致、"给病人用错药的剂量"。绝不能发生。

由此得出全文最著名的判决——Redlock **"非鱼非禽"（neither fish nor fowl）**：对效率锁太重太贵（一台 SETNX 就够了），对正确性锁又不够安全。

**第二拳：就算锁服务完美，锁也保不住你。** 经典时序（后来成为 DDIA 最广为流传的插图）：

```
客户端A: 拿到锁(30s过期) → 【STW GC 暂停 40 秒】────→ 醒来，继续写入存储 💥
锁:                       ← 第30秒过期自动释放
客户端B:                     拿到锁 → 写入存储
```

A 醒来后**根本不知道自己被冻结过**。写入前再检查一次也没用——Kleppmann 原话："GC 可以在任意点暂停线程，包括对你最不利的那个点——检查之后、写入之前。" 没有 GC 也逃不掉：缺页中断、CPU 抢占、`SIGSTOP`，"Whatever. Your processes will get paused."；GitHub 出过数据包在网络里延迟 90 秒的著名事故；HBase 真的踩过这个 bug。

**解药：Fencing Token（栅栏令牌）。** 既然管不住过期的客户端，就让存储侧当门卫：锁服务每发一次锁附带一个**单调递增编号**，存储记住见过的最大编号，**拒绝一切编号倒退的写入**：

```
A: 拿锁 token=33 → 【GC】────────→ 带 33 写入 → 存储:"我见过34了，拒绝" ❌
B:        拿锁 token=34 → 带 34 写入 → 存储: 收下 ✓
```

互斥被破坏的瞬间，系统给出的是**显式失败**而不是静默的数据损坏。然后是对 Redlock 的具体判决：**Redlock 生成不了 fencing token**——它的随机值没有顺序；想造全局递增计数器？"很可能你需要一个共识算法才能生成 fencing token"——把 Redlock 修好的前提，是先引入一个它本想替代的东西。

**补刀：时钟假设。** 分布式理论铁律：好算法在时序全乱时可以变慢（活性受损），但绝不能出错（安全性无恙）。而 Redlock 的**安全性本身**依赖时钟：5 节点例子里，客户端 1 在 A/B/C 加锁成功 → **节点 C 时钟前跳**导致锁提前过期 → 客户端 2 在 C/D/E 加锁成功 → 两个客户端都"合法"持有多数派锁。Redlock 只在"网络延迟有界、进程暂停有界、时钟误差有界"的同步系统模型下才正确——这种保证"通常只在汽车气囊系统之类的硬实时系统里才有"。

## 3.3 antirez 的三条防线

**防线一：fencing 是个悖论。** "如果你的存储能做到'只接受 token 更大的写入'，那它就已经是线性一致存储了——你还要分布式锁干什么？" 且 Redlock 的随机 token 可以做 Check-and-Set 达到类似效果。另外很多被锁保护的资源（外部 API、物理设备）根本没有能检查 token 的"存储服务器"——这一点后来被公认是 fencing 方案的真实局限。

**防线二：我要的时钟假设没那么苛刻。** 不需要各服务器对准绝对时间，只需要"数 5 秒误差 10% 以内"的相对速率；时钟跳变用运维守则解决（别手动改时间、NTP 配平滑校时）。他同时大方承认："Martin 说得对，Redis 应该改用单调时钟 API，我几周内实现。"

**防线三：锁过期问题是所有人的问题。** GC 导致锁过期后继续操作，对 ZooKeeper、对一切自动释放的锁都存在，凭什么单说 Redlock？（这话对了一半——问题确实普遍，但 Kleppmann 的原论点恰恰是"所以光有锁不够，必须 fencing"，而 Redlock 恰恰给不出。）

## 3.4 结局

没有裁判宣判，但历史给出了判决：

- Kleppmann 在原文加注："他提出了一些不错的观点，**但我坚持我的结论**。" 整套论证沉淀进 DDIA 第 8、9 章，随书成为行业圣经。
- Hacker News 风向偏 Kleppmann。最有力的两击：tptacek 指出"健全的系统失败时不该**静默**损坏数据"；Tokio 作者 carllerche 戳破防线一——**接收 fencing token 的一方不需要线性一致存储**，连最终一致的 Cassandra 都能做单调 token 检查，"悖论"不成立。
- antirez 呼吁的 Jepsen 测试至今没人做。Jepsen 作者 Kyle Kingsbury 后来有句流传很广的话："谁想卖给你分布式锁，谁就是在卖木屑和谎言"——所有分布式"锁"本质上都只是租约（lease），必须配 fencing 才安全。
- **最有说服力的证据：Redis 官方文档自己"投降"了一半。** 今天的 [Redlock 官方页面](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/) 加入了免责声明，原话是 "**You should implement fencing tokens**"，并承认墙钟漂移可能导致锁被多个进程同时持有，还并排贴出了双方文章的链接。
- 但 antirez 也没全输：Redlock 至今在"锁失效代价可接受"的场景大量使用。而讽刺的闭环是：如果你只需要效率锁，按 Kleppmann 的建议连 Redlock 都不用搭，单台 SETNX 就够了。

一句话总结遗产：**Kleppmann 赢了框架**（efficiency/correctness 二分 + fencing 成为教科书结论），**antirez 赢了市场**（大多数场景只需要效率锁）。

# 四、etcd / Raft：已确认的写入为什么永不丢失

论战的结论是"正确性场景请找共识系统"。那共识系统凭什么做到 Redis 做不到的事？以下机制全部出自 [Raft 论文](https://raft.github.io/raft.pdf) 和 [etcd 官方文档](https://etcd.io/docs/v3.5/learning/api_guarantees/)。

## 4.1 三条纪律

Raft 要解决的问题一句话：**让一群会宕机、网络会断的机器，对"发生过什么"永远保持同一份账本，且已确认的账目永不丢失。** 它的设计哲学是三条纪律：

1. **一切听班长的（Strong Leader）**：任何时刻至多一个 Leader，所有写入只从 Leader 流向其他节点；
2. **多数派说了算（Quorum）**：选班长要过半数同意，记账要过半数抄录。数学保证：**任何两个多数派必有交集**；
3. **辈分压死人（Term）**：单调递增的任期号，谁的 term 大谁说了算，term 小的消息一律当废纸——防止"旧班长诈尸"的照妖镜。

## 4.2 5 节点集群的一生

设 etcd 集群 {A, B, C, D, E}。

**第一幕：选举。** 开机时全员 Follower，各自持一个**随机**倒计时（论文示例 150–300ms）。A 最先超时 → term 升为 1，变 Candidate，投自己一票并群发拉票 → 其他节点检查"这个 term 我投过没有（一人一票）"和"候选人的账本至少和我一样新吗" → A 集齐多数票当选，立刻发心跳压制其他人的选举。随机化保证大概率只有一人先醒来，避免选票瓜分；论文实测 5 节点平均约 35ms 选出新主。

**第二幕：写入的七步生命周期。** 客户端 `put 退款单42 已处理`：

1. 请求到达 Leader A；
2. A 追加到本地日志（此刻是"铅笔字"，未提交）；
3. A 并行向所有 Follower 发 AppendEntries；
4. 每个 Follower **先把日志持久化到磁盘 WAL，才回复确认**；
5. A 收到 B、C 确认——加上自己 3/5 **过半**，标记为 **committed**（钢笔字）；
6. A 应用到状态机，**然后才回复客户端 OK**；
7. commit 进度随后续心跳传播给 D、E。

对照 Redis：Redis 是"我自己写完就回 OK，副本回头慢慢传"；etcd 是"**多数派已落盘，我才敢回 OK**"。论文原话：

> "Raft guarantees that committed entries are durable and will eventually be executed by all of the available state machines."

代价也很诚实：每次写入多一轮网络往返 + fsync。**一致性是花钱买的。**

**第三幕：分区。** 集群裂成 {A, B}（旧 Leader 在少数派）和 {C, D, E}（多数派）：

- 少数派侧：A 仍自认 Leader，可以收写请求，但永远凑不齐 3 票、**永远无法提交、永远不会回复 OK**——这些写入从未被"确认"，丢了不算违约；
- 多数派侧：C 超时发起选举，term 升为 2，拿 3 票当选，照常营业。**对外服务不间断**；
- 分区愈合：A 看到 term 2 > 1，**立即无条件退位**；它的铅笔字与新账本冲突，被删除覆盖——被回滚的全部是从未确认过的写入。

etcd 官方对此的表述干脆利落："There is no 'split-brain' in etcd."

**第四幕：最精妙的一环——选举限制。** 最坏情况：Leader C 刚提交一笔写入（C+D+E 三份），下一毫秒就宕机。会丢吗？投票规则里有一条铁律：**"你的账本比我旧？我不投你。"**（拉票请求携带最后一条日志的 index 和 term，投票者拒绝日志更旧的候选人。）多数派交集论证：

- 那笔写入在多数派 {C, D, E} 手里，C 死了还剩 {D, E}；
- 新 Leader 要当选需要 4 个活人里至少 3 票，**任何 3 人组合必然包含 D 或 E**；
- 账本落后的候选人会被 D、E 拒投，永远选不上；能选上的只能是 D 或 E——**它们本来就有那笔账**。

这就是论文的 **Leader Completeness** 性质：新班长从当选那一刻起，账本里必然带着所有已提交的账目。Redis 的故障切换是"哪个副本活着就提拔哪个"，副本有没有最新数据看运气；Raft 是**制度性地保证"没有完整账本的人根本选不上"**。

## 4.3 etcd 加的三件武器

**武器一：线性一致读（ReadIndex）。** 防"自以为还是 Leader 的旧主用旧数据答题"：Leader 服务读请求前，先和多数派交换一轮心跳确认"我还没被废黜"。不在乎读到稍旧数据可以显式选 `serializable` 读——**又是那个滑块**，etcd 让你在每次读上自己选 C 还是性能。

**武器二：revision——官方盖章的 fencing token。** etcd 有全集群单调递增的版本号 revision。还记得 Kleppmann 说"生成 fencing token 可能本身就需要共识算法"吗？etcd 就是那个共识算法，revision 就是那个 token——这不是引申，[etcd 官方文档](https://etcd.io/docs/v3.5/learning/why/) 原话：

> "In *How to do distributed locking*, Martin Kleppmann introduced the idea of fencing token. The authors interpret that fencing token is revision number in the case of etcd."

论战的闭环在这里完成：Kleppmann 指出的病，etcd 开出了官方处方。

**武器三：Lease + 锁配方，以及一句诚实的告诫。** etcd 的锁：客户端建带 TTL 的租约（自动续约），在锁前缀下写入 key，**revision 最小者持锁**，崩溃则租约到期 key 自动删除。但官方文档自己写着：

> "The lease mechanism itself doesn't guarantee mutual exclusion."
> （租约机制本身并不保证互斥。）

租约的 TTL 是物理时间，服务端认为已过期、客户端还自以为持锁的窗口依然存在（GC 老剧本换个舞台照样上演）。所以 etcd 的标准姿势是：**持锁不算数，写入时带条件**——用事务附上获锁时的 revision 做条件写，过期的旧持锁者必然条件不满足被拒绝。这正是整场论战的最终答案：**锁只是排队机制，安全性由 fencing 兜底。**

## 4.4 对照表

| 问题 | Redis (Cluster/Redlock) | etcd (Raft) |
|---|---|---|
| 写入确认的含义 | 一台 master 收到了 | **多数派已写入磁盘** |
| 已确认写入会丢吗 | 会（官方承认的窗口） | 不会（选举限制 + 多数派交集） |
| 脑裂可能吗 | 短窗口内可能 | 不可能（term + 多数派唯一性） |
| fencing token | 造不出来 | revision，官方盖章 |
| 代价 | —— | 每写多一轮 RTT + fsync |
| 适合当什么 | 缓存、效率锁、限流 | 元数据、选主、配置、正确性锁 |

# 五、一致性哈希与 AP：一个不存在的因果关系

下一个追问："原博客说 Redis Cluster 不用一致性哈希而用哈希槽，这跟它倾向 AP 有什么关系？我没看出来。"

**这个怀疑本身就是对的一半答案：直接因果关系本来就不存在。** 但顺着挖下去，能把"分布式系统里哪些决定管什么"彻底理清。

## 5.1 两个互相独立的问题

把分布式存储想象成图书馆，它要回答两个不同的问题：

- **问题一：这本书放哪个书架？（数据放置 / Sharding）**——一致性哈希、哈希槽、范围分片都是这个问题的答案；
- **问题二：这本书的复印件之间怎么同步？（数据复制 / Replication）**——异步复制、多数派写入、Raft 都是这个问题的答案。

**CAP 的 C/A 之争完全发生在问题二里。** 两个证伪实验：

1. **反例**：AP 系统的鼻祖 Amazon Dynamo 和 Cassandra，用的**恰恰是一致性哈希**。若"不用一致性哈希"与 AP 有因果，Dynamo 应该是 CP 才对。
2. **思想实验**：保持 16384 个槽一根毛不动，只把异步复制换成"多数派落盘才返回"——Redis 立刻变成偏 CP。**拧动 C/A 旋钮的手柄在复制机制上，不在哈希算法上。**

Redis 不用一致性哈希的真实原因全是工程便利：固定槽位让数据迁移可精确控制（按槽迁移）；客户端一行 `CRC16 mod 16384` 就能算路由；能支持 hash tag 实现多 key 原子操作——没有一条和 C/A 有关。

## 5.2 但间接联系真实存在：三条线

**联系一：博客说的"可用性"是另一种可用性——爆炸半径。** "通过 sharding 提供良好的可用性"里的可用性是运维意义上的**故障隔离**：不分片时一台挂 = 100% 数据不可用；分 3 片后一台 master 挂只影响 1/3 的槽位。这提升的是"部分故障下还有多少服务活着"，不是在 C/A 天平上压砝码。两种"可用性"混在同一段里，正是困惑的来源。

**联系二（最深）：槽位表本身就是用 AP 的方式管理的。** "哪个槽归哪台机器"这张表本身也是需要全体达成一致的数据，存哪？谁说了算？MongoDB 把它放进 Config Servers（多数派复制集，CP 式管理）；Kafka 新版用 KRaft（CP）；而 **Redis Cluster 的槽位表没有中心存储、不走任何共识协议**——每个节点各持一份，靠 gossip 扩散 + config epoch + "最后一次故障切换获胜"来最终收敛。客户端的路由表也可能过期，所以才有 MOVED/ASK 重定向来事后纠错。官方规范明确把"持有过期路由表的客户端把写入发给旧 master"列为丢写场景之一。所以准确的说法是：**"用不用一致性哈希"与 AP 无关，但"槽位表用 gossip 而非共识协议管理"与 AP 大大有关**——同一种设计哲学在元数据层的复现。

**联系三：它们是亲兄弟——共同的爹叫"性能优先"。** 官方规范一句话说透动机：

> "you can expect the same performance as a single Redis instance multiplied by N... Very high performance and scalability while preserving weak but reasonable forms of data safety and availability."
> （N 个 master 的集群性能 = 单机 × N……以"弱但合理"的数据安全和可用性为代价，换取极高的性能和扩展性。）

哈希槽 + 客户端直连（路由层性能）、异步复制（AP 取向）、gossip 管元数据（元数据层最终一致）——**不是父子关系（谁导致谁），而是兄弟关系**，都是"性能优先、一致性够用就行"这个爹生出来的孩子。气质相似，所以博客把它们放在一起讲不算错，但相似不等于因果。

## 5.3 两轴矩阵

| 系统 | 放置机制（问题一） | 复制/元数据机制（问题二） | CAP 性格 |
|---|---|---|---|
| Redis Cluster | 哈希槽 | 异步复制 + gossip 元数据 | **AP** |
| Dynamo / Cassandra | **一致性哈希** | 可调 quorum、最终一致 | **AP** |
| MongoDB | 范围/哈希 chunk | 复制集多数派 + CP 元数据 | 偏 CP / 可调 |
| etcd | 不分片 | Raft 多数派 | **CP** |

用一致性哈希的是 AP，不用的也是 AP——哈希方案这一列对 CAP 性格没有任何预测力。

# 六、MongoDB：把 CAP 滑块做成旋钮，交到每个请求手里

原博客对 MongoDB 只画了架构图（Shards / mongos / Config Servers / 分片键 / Balancing），没有给它贴 CAP 标签。这不全是偷懒——**MongoDB 难以被钉在 CP 或 AP 上，因为它把滑块做成了两个参数**：

- **writeConcern（写关注）**：这笔写要几个副本确认才算成功？`w:1`（主节点收到就行）还是 `w:"majority"`（多数派落盘才行）？
- **readConcern（读关注）**：这次读能接受什么级别的数据？`local`（可能读到将来被回滚的数据）、`majority`（只读已确认的）、`linearizable`（线性一致读）？

一句话抽象：**同一个集群，`w:1` 的请求走 Redis 的剧本，`w:"majority"` 的请求走 etcd 的剧本。CAP 从"系统级选型"变成了"每个请求的运费选项"。** 底座是：每个分片内部是一主多从的复制集，选举协议是"从 Raft 汲取灵感的自研共识协议"（Jepsen 报告原话）——**CP 的骨架，开了一扇 AP 的门**。

## 6.1 同一笔存款，两种运费，两种命运

余额服务换成 MongoDB 分片集群，user_id=7788 归分片 S2（一主两从）管，存入 50 元。

**A 线，`w:1`（不买保价）**：S2 主节点 P 写本地 oplog → **立即回复"成功"** → 0.5 秒后 P 宕机，oplog 还没被任何从节点拉走 → 选举出的新主没有这笔存款 → 旧 P 修好归队，**回滚**这段日志向新主看齐 → 余额还是 100。**你收到过"成功"，钱却没了。** [官方文档](https://www.mongodb.com/docs/manual/reference/write-concern/) 白纸黑字："Data can be rolled back if the primary steps down before the write operations replicate to any of the secondaries."（[Jepsen 报告](https://jepsen.io/analyses/mongodb-4.2.6) 的挖苦："rollbacks, which is a fancy way of saying 'data loss'"——回滚，不过是"数据丢失"的花式说法。）

**B 线，`w:"majority"`（买足额保价）**：P 写本地 oplog → **等**至少一个从节点也持久化（1 主 + 1 从 = 3 人中的 2 人）→ 才回复"成功" → 同样的宕机 → 选举时**持有更新 oplog 的从节点赢得选举**（多数派交集，与 Raft 同构）→ 新主有这笔存款，余额 150。✓

同一个集群、同一笔写、同一场宕机，差别只是客户端代码里的一个参数。代价同样具象：B 线延迟高于 A 线。**天下没有免费的 C。**

## 6.2 默认值的黑历史

"旋钮交给用户"的阴暗面是**用户根本不知道旋钮的出厂位置**：

- MongoDB 4.x 及之前，**默认 writeConcern 是 `w:1`**。Jepsen 报告原话毫不客气："MongoDB's default level of write concern was (and remains) acknowledgement by a single node, which means **MongoDB may lose data by default**."
- Jepsen 2020 年测 4.2.6 时发现：即使开到最强设置，多文档事务在网络分区下依然丢已确认的写（事务重试机制 bug，4.2.8 修复）；同一份报告还点名批评 MongoDB 官网一边宣传"业界最强一致性"，一边在"MongoDB and Jepsen"页面只字不提这些发现。
- 结局是建设性的：**MongoDB 5.0（2021）起默认 writeConcern 改为 `w:"majority"`**。出厂位置从 AP 端挪到了 CP 端。

这段历史是最好的教材：**可调一致性系统的最大风险从来不是"调错"，而是"根本不知道要调"。**

## 6.3 三个系统，三种人生态度

| | Redis Cluster | MongoDB | etcd |
|---|---|---|---|
| 副本怎么同步 | 异步，焊死 | 异步底座 + **writeConcern 旋钮** | Raft 多数派，焊死 |
| 已确认的写会丢吗 | 会 | **看你拧到哪档** | 不会 |
| CAP 性格 | AP（焊死） | **可调**（5.0 起出厂在 CP 端） | CP（焊死） |
| 一句话人设 | "快就完事了" | "都行，你说了算" | "错一个字都不行" |

三个系统恰好构成一条光谱：AP 焊死 → 可调 → CP 焊死。这正是原博客核心论点（CAP 是连续的权衡空间）最完美的工程注脚——MongoDB 干脆把这个权衡空间做成了 API。

# 七、组合怎么选：四个套餐 + 阿里云落地避坑

最后一个追问：writeConcern × readConcern × readPreference 的笛卡尔积有 125 种组合，业务怎么选？

## 7.1 组合爆炸是纸老虎

三条规则把空间砍到只剩几格：

1. **大部分组合无效或自相矛盾**：`linearizable` 只能配读主节点；`w:0` 生产等于自杀；`snapshot` 基本只在事务里用。
2. **读写关注必须配对，单边加强是浪费钱**。[官方因果一致性表格](https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/) 的结论一句话：**只有 `readConcern: majority` + `writeConcern: majority` 同时开，四项因果保证（含"读到自己刚写的"）才全部成立，任何一边降级全部塌掉**。`w:majority` 写配 `local` 读从节点——从节点没同步到照样读不到自己写的；`w:1` 写配 `majority` 读——严谨地读一个会消失的值没有意义。
3. **套餐跟着数据的身价走**：还是 Kleppmann 那把尺子——丢了/读旧了会不会造成不可挽回的损失？

## 7.2 业界实际在用的四个套餐

| 套餐 | writeConcern | readConcern | readPreference | 典型场景 |
|---|---|---|---|---|
| **① 默认套餐** | `majority` | `local` | `primary` | 订单、用户资料等一般核心数据（**5.0+ 出厂默认，80% 场景直接用**） |
| **② 性能套餐** | `w:1` | `local` | `secondaryPreferred` | 日志、埋点、点赞数 |
| **③ 强一致套餐** | `majority` (+`j:true`) | `majority` / `linearizable` | `primary` | 余额、扣库存、分布式锁、状态机流转 |
| **④ 因果会话套餐** | `majority` | `majority` + causal session | 可读从 | 改完资料立刻回显、评论后立刻可见 |

**业务选择的核心答案：选择的方式是不选择，用默认（套餐①）；只有明确知道自己为什么要偏离默认时才动它**——向上（钱、锁 → ③）或向下（日志 → ②）。

## 7.3 阿里云 ApsaraDB for MongoDB 现状（2026-07，均核对官方文档）

**能买什么**：在售大版本 4.2 / 4.4 / 5.0 / 6.0 / 7.0 / 8.0 / 8.3（阿里云是 MongoDB 官方中国战略合作伙伴，8.x 均"国内首发"）。新业务无脑选 **8.0**。各版本值得记住的：4.2 跨分片事务；4.4 Hidden Index、refineCollectionShardKey；5.0 **默认 writeConcern 改 majority**、时序集合、在线重分片；7.0 analyzeShardKey（分片键体检）；8.0 性能大版本（majority 写改为"多数派写完 oplog 即确认，不等回放"，把 majority 的税降低了）。

**三种形态**：

1. **单节点**：无副本集、**无 oplog** → 不支持事务、不支持时间点恢复、无 SLA，官方定位仅"开发、测试、学习"。**生产禁用。**
2. **副本集（主力）**：三节点 = **1 主 + 1 从 + 1 Hidden**——三个都是数据节点，majority=2 总能凑齐。这个设计避开了自建圈经典的 PSA 架构坑（arbiter 不存数据，数据节点挂一个后 majority 确认点停滞、缓存压力雪崩）。
3. **分片集群**：mongos（最多 32）+ Shard（每个是一套副本集，最多 32）+ 三节点 ConfigServer——原博客画的架构图在阿里云上就是这个产品形态。

**默认配置**：writeConcern/readConcern 跟随社区版（5.0+ 默认 `w:majority`）。阿里云文档还特意提醒反向坑：从 4.x 升 5.0+ 默认写关注变强，"可能出现性能退化"——不是 bug，是原来没保价的包裹默认全保价了。

**常见的坑（按踩中率排序）**：

1. **用 Primary 单点直连地址连生产库**。主备切换后"Primary 地址"指向的机器变成从节点，应用瞬间不能写（官方原话"严重影响业务正常运行"）；更阴的反向坑：直连 Secondary 的只读应用，切换后那台可能升主，**"只读"应用突然有了写权限**。永远用高可用连接串（ConnectionStringURI / SRV）。
2. **应用没有重连机制**。主备切换、变配、升级都有 30 秒内闪断，官方在七八处文档反复强调。
3. **每个请求 new 一个 MongoClient**。连接数按节点限制（低配每节点 500），MongoClient 必须全局单例。这是阿里云社区第一大工单来源。
4. **oplog 窗口不够，从节点同步断裂**。批量灌数据时 oplog 暴涨，从节点落后超过窗口就 "too stale to catch up" 进入 RECOVERING **且无法自愈，要提工单**。官方建议窗口 ≥24 小时、主备延迟配 ≥10 秒告警。
5. **分片键选错**。低区分度 → jumbo chunk；单调递增（时间戳）→ 写热点。正解参考官方物联网案例：`(deviceId, timestamp)` 复合键。补救逐版本变强：4.4 加后缀、5.0 在线重分片、7.0 事前体检。
6. **长事务**。默认 60 秒强杀、锁等待仅 5 毫秒；未提交事务积压 WiredTiger 缓存导致整实例卡顿。阿里云参数指南写着事务超时"可以适当调小，**不建议调大**"。
7. **双可用区 + majority 的容灾窗口**：一个可用区整体故障时，切换窗口内同步延迟范围的数据可能丢，且双可用区默认**人工切换**。要更强容灾上三可用区。

## 7.4 完整示例：电商系统上云的全决策链路

三类数据落 MongoDB：**订单**（丢一条赔一条）、**商品目录**（读多写少，读旧 3 秒无所谓）、**行为日志**（丢一批无所谓，量大）。

1. **选形态和版本**：生产排除单节点；先买三节点副本集，版本 8.0；日志单独一个便宜实例，与订单隔离。
2. **接入**：复制高可用连接串（不是直连地址）；MongoClient 全局单例，`maxPoolSize` 按"应用实例数 × 池大小 < 每节点上限"倒推；上线前用控制台主备切换演练一次 30 秒闪断恢复。
3. **配套餐**：订单写入什么都不配（8.0 默认就是套餐①，宕机时选举限制保护已确认的订单）；支付状态流转升级套餐③（`readConcern: majority` 防"基于幻影状态做决策"+ 事务，事务里绝不调外部接口）；商品目录走 `secondaryPreferred` + 只读节点（心里默念：异步复制，大促时延迟可到秒级，**价格计算、库存扣减绝不走这条路**）；行为日志 `w:1` 批量写。
4. **用户体验补丁**：产品要求"改完收货地址立刻可见"但读走从节点 → causal session（套餐④），并确认读写都在 majority 档（官方表格：不配对则因果保证失效）。
5. **运维保险**：主备延迟 ≥10 秒告警；确认时间点恢复可用；大促灌数据盯 oplog 窗口。
6. **一年后扩分片集群**：先用 analyzeShardKey 体检候选键，选 `(userId, orderId)` 复合哈希键避开时间戳写热点；mongos 至少 2 个都写进连接串；选错了还有 8.0 的 reshardCollection 兜底。

# 八、全文总结

从原博客出发，这条学习路径最后连成了一个完整的闭环：

1. **CAP 不是三选二的选择题，是连续的权衡空间**：P 是前提；A 的工程含义是"高可用 + 客户端兜底"；C 是可以分级降价出售的商品。设计系统时问的不是"我选 CP 还是 AP"，而是"**分区发生时，我把一致性降到哪一档、把可用性保到哪几个功能**"。
2. **原子性和一致性是两根不同的轴**：Redis 的 SETNX/Lua 用单线程排队完美解决"并发"这个敌人，但 AP 选型（异步复制）意味着已确认的原子写入可能在 failover 时整体蒸发——"故障"这个敌人它故意不管。
3. **任何带自动过期的锁都防不住暂停的客户端**（Kleppmann），这个问题对所有人都存在（antirez），所以答案是**锁只管排队，安全性由 fencing 兜底**——而生成 fencing token 需要共识，etcd 的 revision 就是官方盖章的 fencing token。
4. **数据放置（sharding）和数据复制（replication）是两个正交的问题**，CAP 之争全部发生在后者；一致性哈希与 AP 没有因果关系。
5. **MongoDB 把滑块做成了每个请求的旋钮**，而它的默认值变迁史（w:1 → w:majority）告诉我们：可调系统的最大风险不是调错，而是不知道要调。
6. **落地时，形态选错 > 参数选错**；组合爆炸是纸老虎，业界的标准答案是四个配对套餐，80% 的场景用默认就是正确答案。

# 参考

- 原博客：[分布式理论: 再谈CAP](https://blog.pdjjq.org/post/distributed-theory-talk-about-cap-again-12jnyp)
- [Redis Cluster Specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [Redis: Scripting with Lua](https://redis.io/docs/latest/develop/programmability/eval-intro/)
- [Redis: Distributed Locks (Redlock)](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- Martin Kleppmann: [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- antirez: [Is Redlock safe?](http://antirez.com/news/101)
- Ongaro & Ousterhout: [In Search of an Understandable Consensus Algorithm (Raft)](https://raft.github.io/raft.pdf)
- [etcd: API guarantees](https://etcd.io/docs/v3.5/learning/api_guarantees/) / [etcd versus other key-value stores](https://etcd.io/docs/v3.5/learning/why/)
- [MongoDB: Write Concern](https://www.mongodb.com/docs/manual/reference/write-concern/) / [Read Concern](https://www.mongodb.com/docs/manual/reference/read-concern/) / [Causal Consistency and Read/Write Concerns](https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/)
- Jepsen: [MongoDB 4.2.6](https://jepsen.io/analyses/mongodb-4.2.6)
- 阿里云：[MongoDB 版本及存储引擎](https://help.aliyun.com/zh/mongodb/product-overview/mongodb-versions-and-storage-engines) / [事务与 Read/Write Concern 最佳实践](https://help.aliyun.com/zh/mongodb/use-cases/transactions-and-read-write-concern) / [oplog 设置最佳实践](https://help.aliyun.com/zh/mongodb/use-cases/best-practices-and-risks-for-oplog-settings) / [副本集读写分离和高可用](https://help.aliyun.com/document_detail/57670.html)
