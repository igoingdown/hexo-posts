---
style: summer
title: 从 Kitex 到 ALB：一个大厂微服务老兵眼里的"原始"架构，其实是教科书
date: 2026-07-05 17:22:04
tags:
  - 微服务
  - 服务发现
  - 阿里云
  - Kitex
---

> 副标题：Server-side Discovery vs Client-side Discovery——当你从注册中心+直连+Mesh 的世界，走进"一个负载均衡打天下"的小团队

## 一、开场：这也太原始了吧？

我在字节待过几年，写服务的肌肉记忆是这样的：定义 IDL、生成代码、`NewClient("user-service")`、框架自动去注册中心拉实例列表、加权负载均衡、直连、熔断限流打点全内建。后来 Mesh 铺开，连这些选项都不用配了，治理逻辑整个下沉到 sidecar，业务代码干净得像本地函数调用。

后来我加入了一个小团队——服务数不到二十，跑在阿里云上。我问："服务之间怎么互相调用？注册中心用的什么？"

答案是：没有注册中心。服务 A 调服务 B，就是往一个固定的内网域名发 HTTP 请求，域名解析到一个叫 ALB 的东西，ALB 把请求转给 B 的某台虚拟机。完了。

我的第一反应有两层。表层是大厂人的本能不适：这也太原始了吧？没有服务发现、没有 RPC、没有治理、实例列表靠在控制台里手动维护——这不是 2014 年的架构吗？但真正让我记到今天的是脱口而出的第二句："发 HTTP 请求到一个域名？那流量不会要出公网绕一圈才到对面的服务实例吗？一跳几十甚至上百毫秒的延迟，内部调用怎么受得了？"

同事当场反问内部调用延迟 100ms 的量级是怎么来的，我意识到这个数据是我潜意识的猜测，没有任何直接证据！于是我花了些时间把这套东西系统性地研究了一遍，结论让我更惭愧：**这不是"没有服务发现"，这是教科书上白纸黑字写着的 Server-side Discovery（服务端服务发现）模式，而且在这个规模下，它大概率是最优解。** 我在大厂用的那套，只是同一个问题的另一种解法——Client-side Discovery（客户端服务发现）——它的一切复杂度，都是被 5 万+服务、300 万容器的规模逼出来的。

这篇文章写给和我一样的人：熟悉 Kitex/注册中心/Mesh 那一套，但对阿里云的 SLB/ALB/PrivateZone/ECS 全家桶一脸茫然的工程师。

## 二、先扫盲：阿里云那些产品，映射到你熟悉的概念

大厂内部基建是自研黑盒，阿里云则是把同样的能力拆成标价商品卖。做一个粗暴的概念映射：


| 阿里云产品           | 它是什么                                                                                               | 你熟悉的对应物         |
| --------------- | -------------------------------------------------------------------------------------------------- | --------------- |
| **VPC**         | 私有网络，一个隔离的内网地址空间。内网域名解析到的是 VPC 内的私网地址（如私网型 ALB 的 VIP），流量从发出到落地全程在 VPC 内流转，一步都不出公网——对，就是我开场没搞明白的那件事 | 机房内网/网络平面       |
| **ECS**         | 裸虚拟机。注意：不是 K8s Pod，没有编排，IP 相对稳定                                                                    | 物理机/虚拟机时代的"机器"  |
| **SLB 家族**      | 负载均衡产品线的总称                                                                                         | 内部四层/七层接入体系     |
| **CLB**         | 老一代负载均衡，四层为主                                                                                       | L4 LB           |
| **NLB**         | 新一代四层负载均衡                                                                                          | L4 LB           |
| **ALB**         | 新一代七层负载均衡，能看懂 HTTP，按域名/路径/Header 路由                                                                | 内部统一接入层 / L7 网关 |
| **PrivateZone** | 内网 DNS，把 `user-service.internal` 这类域名解析成 VPC 内的 IP                                                 | 内部 DNS          |


这张表你真正需要记住的只有两个：**ALB（L7）和 PrivateZone**。CLB/NLB 是四层产品，列出来一是防止你在控制台里迷路，二是它们会在第五节 gRPC 长连接的坑里再次出场。

几个对大厂人反直觉的点：

1. **ECS 是长寿命的**。没有 K8s 的 Pod 生灭，一台 ECS 的 IP 可能几年不变。"实例 IP 高度动态"这个我们默认的前提，在这里不成立。
2. **ALB 解析出来的是 VIP（虚拟 IP）**。VIP 不属于任何一台具体机器，由负载均衡集群共同持有，节点挂了 VIP 不变、流量自动漂移。所以那个"固定域名"是真的可以焊死在配置文件里一辈子不改。
3. **健康检查、故障摘除、转发调度，全部由阿里云运维，SLA 厂商兜底**。这三样在字节内部各自都是有专职团队的系统。



## 三、主轴：Server-side Discovery vs Client-side Discovery

好，进入正题。服务发现要回答的问题只有一个："调用方如何找到被调方的活实例？"两种模式的分野，在于**这个问题在链路的哪一侧被回答**。

### 3.1 两条链路长什么样

小团队的链路（Server-side Discovery，A 方案）：

```
[服务 A] ──① DNS 查询──▶ [内网 DNS/PrivateZone]
   │            ◀── 返回 ALB 的 VIP（如 10.0.0.88）
   ├──② HTTP 请求发往 VIP──▶ [ALB 集群]
   │                            │ ③ 查健康检查结果，
   │                            │   从后端服务器组挑一台健康机器
   │                            ▼
   │                        [服务 B 的某台 ECS]
   ◀──────④ 响应原路返回──────┘
```

大厂的链路（Client-side Discovery，B 方案）：

```
              ┌────[注册中心]──────────┐
       ①注册+心跳                  ②订阅地址列表
              │                        │
  user 实例 ◀─┘        order ──③挑一台直连(省一跳)──▶ user 实例
```

先解释图里唯一的新名词：**后端服务器组**，就是你在 ALB 控制台/API 里维护的那份目标实例列表——一个 ALB 可以挂很多组，按域名/路径把流量分给不同的组。记住它，下面的类比里它是主角。

microservices.io（Chris Richardson 维护的微服务模式目录）对 Server-side Discovery 的定义原话是："客户端通过一个运行在固定已知位置的路由器（即负载均衡器）发请求……AWS ELB 就是一个服务端发现路由器的例子。"阿里云 ALB/SLB 和 AWS ELB 是同类产品。换句话说：

- ALB 的**后端服务器组** = 服务注册表；
- ALB 的**健康检查** = 心跳/探活；
- ALB 的**转发调度** = 负载均衡。

我们在字节说的"服务发现"，默认指的是 Client-side Discovery；而小团队"没有服务发现"的真相是——**把服务发现外包给了云厂商**。

### 3.2 逐维度拆解根本区别


|                | Server-side（ALB）                                           | Client-side（注册中心+直连）                                                    |
| -------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------- |
| **实例地址表在谁手里**  | LB 持有，客户端对实例列表一无所知                                         | 客户端订阅注册中心，本地缓存全量地址表                                                     |
| **负载均衡决策在哪一侧** | 服务端侧（LB 集群），连接/请求到了 LB 才挑后端，粒度到连接级/轮询为止                    | 客户端侧，发请求前自己挑实例，可做请求级加权、一致性哈希、感知实例实时负载                                   |
| **健康检查/摘除**    | LB 主动探测，周期性（秒～十秒级窗口），窗口内会有请求打到正在死掉的机器                      | 实例心跳上报（如 Nacos 每 5 秒一次，15 秒标记不健康，30 秒剔除），变化秒级推送；支持发布前主动摘流量的优雅下线，做到发布零报错 |
| **网络路径**       | 多一跳 L7 代理                                                  | 直连，少一跳                                                                  |
| **客户端复杂度**     | 极简，发个 HTTP 就行，任何语言零成本                                      | 要集成发现 SDK，每种语言都得实现一遍                                                    |
| **运维责任归属**     | 云厂商兜底，SLA 有承诺                                              | 注册中心集群（3～5 节点）自己养：监控、升级、防脑裂                                             |
| **成本模型**       | 典型做法是一个 ALB 挂多组监听规则/后端服务器组，按 Host/路径分流覆盖全部服务；成本≈固定实例费 + 流量 | 注册中心机器 + 专职人力                                                           |


几条对大厂人来说值得咀嚼的点：

**地址表的位置决定了一切。** Client-side 模式里客户端持有全量地址表，所以能做请求级的精细均衡、能做优雅上下线。Server-side 模式里客户端是"瞎"的，好处是零复杂度，代价是均衡粒度和摘除时效都受制于 LB。

**多语言在两种模式下是反向的论据。** 在字节，多语言意味着"框架团队要维护 Go/Python/Rust 多套 SDK"；在小团队，多语言恰恰是继续用 ALB 的理由——Server-side 模式对客户端零要求。

**注册中心本身是个要养的分布式系统。** microservices.io 原话："服务注册表必须高可用。"在字节这由专职中间件团队负责，感知不到成本；小团队为这个规模的服务自建一套 Nacos 集群，运维成本大于收益。

**还有一个隐藏前提：K8s。** 业界服务发现的普及，很大程度是容器编排让实例 IP 彻底动态化逼出来的——而 K8s 自己就内置了服务发现（Service + CoreDNS）。ECS + ALB 的世界里，实例 IP 本来就不怎么变，那个痛点根本不存在。

## 四、正面对比：阿里云 ALB 方案 vs 字节体系



### 4.1 规模：一切分野的根源

Kite 与 Thrift 耦合太深、2019 年从零重写、2021 年以 CloudWeGo 品牌开源——这段演进史你多半比我熟，不复述了（不熟的读者可看[官方回顾](https://www.cloudwego.io/zh/blog/2022/03/28/%E4%B8%80%E6%96%87%E4%BA%86%E8%A7%A3%E5%AD%97%E8%8A%82%E8%B7%B3%E5%8A%A8%E5%BE%AE%E6%9C%8D%E5%8A%A1%E4%B8%AD%E9%97%B4%E4%BB%B6-cloudwego/)）。这里只留两个对本文论证真正有用的事实：

- **规模**：在线微服务从 2018 年约 8000 个涨到 2021 年 **5 万+**；Kitex 支撑内部 **6 万+ 服务、峰值数亿 QPS**；Service Mesh 管理容器超 **300 万个**。
- **起点**：2016 年的字节，服务间用的也是简单的 HTTP/Thrift 直调。这套体系是随规模一步步长出来的，不是一开始设计出来的。



### 4.2 逐层对比

发现机制本身的对比见 3.2，这张表只看体系级能力：


| 维度   | ALB 方案          | 字节体系                                          |
| ---- | --------------- | --------------------------------------------- |
| 接口契约 | 口头/文档约定 JSON 字段 | IDL 合同文件，自动生成各语言代码，改了不同步编译期就报错                |
| 序列化  | JSON 文本         | Thrift/Protobuf 二进制，体积小 1/3～2/3、编解码快 3～10 倍 |
| 治理能力 | 基本靠业务代码自觉       | 超时/重试/熔断/限流/泳道/全链路 Trace 框架或 sidecar 内建       |
| 调试   | curl 一把梭        | 二进制协议需要专用工具                                   |
| 运维成本 | 月费固定，配置在控制台点点点  | 注册中心、Mesh 控制面/数据面都要专职团队                       |


关于最后一行的量级，字节公开分享过一个数字，我觉得所有羡慕 Mesh 的小团队都该看看：**sidecar 一次大版本全量发布曾平均耗时 4 个月以上，优化后仍要 2 个月。** 这就是养这套东西的运维量级。在大厂里这是平台团队的日常；在小团队里，这是一个不可能完成的任务。

大厂方案的收益在它的规模下都是真金白银：几万服务 × 每请求省 1～3ms × 调用链 10 层+，累积可观；数亿 QPS 下 Protobuf/Thrift 比 JSON 省的序列化 CPU（3～10 倍）直接等于少买几千台机器——字节公开数据是 Kitex 一年优化后吞吐 +30%、TP99 -67%。但同样这套东西放到这个规模的小团队，收益除以几个数量级，成本一分不少。

### 4.3 业界的共识站在哪一边

- Martin Fowler，**Microservice Premium**："除非系统复杂到单体管不动，否则根本不要考虑微服务"——治理设施是一种"税"，规模不到不要交；
- Oz Nova，**《You Are Not Google》**：UNPHAT 原则——先理解自己的问题，再看方案；Google/字节的方案是为你没有的规模问题设计的；
- O'Reilly，**《Do you need a service mesh?》**：mesh 常被"没想清楚就引入"，多数团队并不需要。



## 五、性能到底差多少：量化拆解

公网之问被打脸之后，我还留着一个更体面的质疑："就算走内网，多一跳代理、JSON 明文，性能不心疼吗？"把数据摆出来看。

### 5.1 逐项拆

先把框架摆正：这里其实混着两个常被捆绑、但彼此独立的决策——**发现在哪一侧被回答**（过 LB 还是直连），和**线上跑什么协议**（HTTP/JSON 还是二进制 RPC）。①是前者的代价，②③是后者的账，而 ④ 会解释它们为什么总是成对出现。

**① 网络路径：多一跳 LB vs 直连**

- 阿里云官方承诺：同可用区内网单向延迟 **P99 < 180 微秒**（RTT < 0.4ms）——云内网本身非常快。也在这里正式回收开头那次露怯：我当初担心的是"出公网绕一圈、一跳几十甚至上百毫秒"，实际是**单向 180 微秒、往返不到 0.4 毫秒**，差着两个数量级以上——这大概是全文最有戏剧性的一组数字；
- L7 代理新增延迟：可比对的数据是 Istio 官方称 Envoy sidecar 在 P99 增加约 0.18～0.25ms，生产经验值单跳 L7 代理约 1～5ms；
- 结论：直连省掉的就是这一跳，量级约 **0.3～3ms**。

**② 序列化：JSON vs Protobuf/Thrift**

- 体积：Protobuf 比 JSON 小约 34%～66%（Auth0 实测，同一结构 99B vs 214B 的经典例子）；
- 速度：编解码快 3～10 倍（Auth0：5 万对象 25ms vs 150ms）；
- 但注意量级：单次调用的编解码只有**微秒～亚毫秒级**。它省的主要是高 QPS 下的 CPU（机器钱），不是单次请求的延迟。

**③ 协议：HTTP/1.1 vs HTTP/2**

一句话：H2 的增量主要是 HPACK 头压缩（HTTP/1.1 每请求 500～800 字节明文头，Cloudflare 全网实测平均省 53%）和多路复用；建连开销两边都能用长连接摊薄，不是主要差异。

**④ gRPC 长连接过 L4 LB 的热点坑——RPC 和服务发现为什么是配套的**

gRPC 走 HTTP/2 长连接，一个客户端的**全部**请求跑在一条连接上。如果把 gRPC 服务挂在 L4 负载均衡（阿里云对应 CLB/NLB）后面，LB 按"连接"分配后端——这条连接建立时选了哪台机器，之后所有流量都砸到那一台，**一台过热、其余闲置**。gRPC 官方博客《gRPC Load Balancing》对此有权威论述，标准解法就是客户端负载均衡，也就是注册中心 + 直连。

这条解释了一个大厂人习以为常但很少深想的因果：**为什么"上了 RPC 就自然而然要上服务发现"——两者是配套的，不是两个独立决策。** 反过来，HTTP/1.1 过 ALB 恰好没有这个坑，所以 ALB 方案内部是自洽的。这也是全文主轴的闭环：协议的选择会反过来决定发现必须在哪一侧发生。

### 5.2 权威基准数据


| 数据                                   | 数值                                                                          | 出处                                   |
| ------------------------------------ | --------------------------------------------------------------------------- | ------------------------------------ |
| gRPC 官方 benchmark（8 核 GCE，实验室）       | unary 延迟 ～300μs；两台 8 核机间 ～15 万 QPS                                          | grpc.io                              |
| Kitex 官方 benchmark（1KB echo，并发 1000） | gRPC-go：11.4 万 QPS / TP99 30.6ms；Kitex：19.4 万 / 11.6ms（QPS 1.7x，TP99 约 1/3） | github.com/cloudwego/kitex-benchmark |
| Kitex Thrift 多路复用模式                  | 41 万 QPS（vs gRPC-go 2～3.6x）                                                 | 同上                                   |
| REST vs gRPC 独立压测（Ian Gorton）        | 小 payload 差距不大；大 payload + 500 并发时 gRPC 吞吐 ～10x                             | Medium                               |
| 字节生产经验值                              | 抖音某服务迁移后 CPU -19%、TP99 -29%                                                 | 字节技术公众号                              |




### 5.3 但对小规模来说，这些数字基本无感

直连 + RPC 比 ALB + HTTP/JSON 每跳大约省 0.3～5ms + 一些 CPU。业务接口 P99 几十毫秒，其中数据库查询 5～50ms、外部 API 数百 ms——省的那点淹没在噪音里。Kelsey Hightower（前 Google）说得直白："gRPC vs REST 的选择几乎从不是真实系统的瓶颈，API 设计纪律远比线上协议重要。"

性能差异要显著，需要三个前提：超高 QPS（序列化占 CPU 30～50% 时省 CPU = 省机器钱）、深调用链、大 payload（大对象列表场景 REST 吞吐可跌到 gRPC 的 1/10）。小团队通常一个都不满足。

### 5.4 调用链深度：大厂抠微秒的数学本质

Google 经典论文《The Tail at Scale》给出了大厂为什么必须优化这些的数学：

- 单个服务 99% 请求 10ms、1% 要 1s。如果一次用户请求要**并行扇出 100 个内部调用**并等最慢的那个：**63% 的用户请求会超过 1s**（1 − 0.99¹⁰⁰）；
- 延迟线性叠加：链深 10+ 时，每跳省 1～3ms = 端到端省 10～30ms+；链深 2～3 层时，只省 2～9ms；
- 可用性同理放大：10 个 99.9% 可用的服务串行调用，整体只剩 99.0%（月停机 43 分钟 → 7 小时量级）。

**字节动辄链深 10 层+、扇出几十路，所以每一跳的微秒都要抠；链浅的小系统，同样的优化收益直接除以 5。** 这才是两套架构分道扬镳的物理根源。

## 六、两段代码看清三种形态（第三种你天天写）

**A 方案：HTTP 过 ALB（Server-side Discovery）**

```python
import requests

# 域名固定，配置文件里写死，一辈子不用改
USER_SVC = "http://user-service.internal.example.com"

def get_user(user_id: int) -> dict:
    # 底层：DNS 解析到 ALB 的 VIP -> ALB 挑一台健康后端 ECS 转发
    resp = requests.get(f"{USER_SVC}/api/users/{user_id}", timeout=3)
    resp.raise_for_status()
    return resp.json()
```

调用方对"user-service 有几台机器、谁挂了"一无所知也不需要知道——全由 ALB 消化。

**B 方案：gRPC + Nacos 直连（Client-side Discovery）**

IDL 合同（`user.proto`）你天天写，略过不贴。真正值得看的是注册 + 订阅 + 直连这一段：

```python
import nacos, random, grpc
client = nacos.NacosClient("nacos.internal:8848", namespace="prod")

# 服务端启动时登记自己，之后 SDK 每 5 秒自动心跳，优雅退出时主动注销
client.add_naming_instance(service_name="user-service",
                           ip=get_local_ip(), port=50051)

# 客户端：拉地址表 -> 自己挑实例 -> 直连，不经过任何代理
instances = client.list_naming_instance("user-service")["hosts"]
target = random.choice([i for i in instances if i["healthy"]])
channel = grpc.insecure_channel(f"{target['ip']}:{target['port']}")
```

注意此刻发生的责任转移：Nacos 集群高可用（3 节点起）、监控、升级、每个服务接 SDK、客户端均衡/故障剔除/重试——在 A 方案里这些全是阿里云的活，现在全回到自己手里。

**第三种形态：Kitex/Mesh**

代码就不贴了——`NewClient` 加几个 option，发现、均衡、超时、熔断、打点全部内建；Mesh 模式下这些选项进一步下沉到 sidecar，连配置都不用写。这就是我们习以为常的"干净"。但现在你知道了，这份干净的背后是一整个平台部门。

## 七、结论：架构没有先进落后，只有匹配不匹配

回到开头那次露怯。一句"HTTP 不会出公网绕一圈吗"，把两件事同时摆上了台面：我"这也太原始了吧"的优越感，和这份优越感脚下的空洞——对云上网络的无知。调研完这一圈，我现在的答案是：

**用一个 ALB 打天下的小团队，像小区门口请了一个物业保安——谁来了登记一下、放行到对应楼栋，省心，月费固定。字节的体系，像一座百万人口城市的交通系统：红绿灯（限流）、环路（泳道）、交警队（治理团队）、导航系统（注册中心+Trace）。城市离不开它，但给一个 10 户的小区修八车道环路，只会得到账单和维护工作。**

如果给从 ALB 起步的团队一张演进路线图，触发信号和路径大致是：

**先不动架构能做的：** 内部调用统一超时和重试规范（比换协议重要 10 倍）；上全链路 Trace——阿里云 ARMS（托管 APM/全链路 Trace 产品，对标你熟悉的内部 Trace 平台）或自建 OpenTelemetry，排查收益立竿见影。

**触发升级的信号：** 服务数逼近 30～50 个，一个 ALB 上的监听规则和后端服务器组开始逼近配额上限、控制台手工维护成为主要出错源（AWS 上老一代"每服务挂一个 ELB"的玩法，正是被这种管理负担逼出了 Cloud Map）；引入自动扩缩容或高频滚动发布，手工维护后端服务器组跟不上；决定上 K8s。

**升级路径（从"白送"到"自建"，成本递增）：**

1. **K8s Service + CoreDNS**：上 K8s 就白送服务发现。注意用本文的二分法看，K8s Service 本质仍是 Server-side Discovery——ClusterIP 就是集群内的 VIP，kube-proxy 就是转发面——所以从 ALB 迁过去心智模型不变；只有 headless service + 客户端 LB/xDS 才切回 Client-side。多数团队到这一步就够了；
2. **阿里云 MSE**（微服务引擎，卖的就是托管版 Nacos）：需要秒级摘流量、动态配置时买托管版，别自建；
3. **gRPC**（配 headless service 或 xDS）：接口多团队协作混乱、或确有高 QPS 内部调用时，再引入 IDL 和 RPC——记住第五节的结论，RPC 和服务发现是配套的，这也是它要配 headless/xDS 的原因；
4. **Service Mesh**：留给"有专职平台团队"的那一天——大概率永远不需要。

字节的架构是被规模逼出来的，不是设计出来的。从大厂出来的人最需要卸载的一个心智负担就是：把"我熟悉的"当成"正确的"。在对的规模做对的事，"一个 ALB 打天下"本身就是一种正确的架构决策。

## 参考资料

**模式与定义**

- microservices.io — [Server-side Discovery](https://microservices.io/patterns/server-side-discovery.html) / [Client-side Discovery](https://microservices.io/patterns/client-side-discovery.html) / [Service Registry](https://microservices.io/patterns/service-registry.html)
- NGINX 官方博客 — [Service Discovery in a Microservices Architecture](https://www.f5.com/company/blog/nginx/service-discovery-in-a-microservices-architecture)
- 阿里云 — [SLB 产品概述](https://help.aliyun.com/zh/slb/product-overview/slb-overview) / [ECS 网络性能 FAQ（180μs 承诺）](https://help.aliyun.com/zh/ecs/network-performance-faqs) / [内网 DNS 解析](https://help.aliyun.com/zh/dns/pvtz-what-is-intranet-domain-name-resolution)

**字节跳动 / CloudWeGo**

- [一文了解字节跳动微服务中间件 CloudWeGo](https://www.cloudwego.io/zh/blog/2022/03/28/%E4%B8%80%E6%96%87%E4%BA%86%E8%A7%A3%E5%AD%97%E8%8A%82%E8%B7%B3%E5%8A%A8%E5%BE%AE%E6%9C%8D%E5%8A%A1%E4%B8%AD%E9%97%B4%E4%BB%B6-cloudwego/)
- [Kitex: Unifying Open Source Practice（6 万服务/数亿 QPS）](https://www.cloudwego.io/blog/2022/09/30/kitex-unifying-open-source-practice-for-a-high-performance-rpc-framework/)
- [字节跳动微服务架构体系演进（InfoQ）](https://www.infoq.cn/article/asgjevrm8islszo7ixzh) / [大规模 Sidecar 运维实践（InfoQ）](https://www.infoq.cn/article/dzey87ntr7znefhhl8us)
- [kitex-benchmark 官方数据](https://github.com/cloudwego/kitex-benchmark)

**性能**

- gRPC 官方 — [Load Balancing 博客（长连接热点坑）](https://grpc.io/blog/grpc-load-balancing/) / [Benchmarking](https://grpc.io/docs/guides/benchmarking/)
- [Auth0: Beating JSON performance with Protobuf](https://auth0.com/blog/beating-json-performance-with-protobuf/)
- [Cloudflare: HPACK](https://blog.cloudflare.com/hpack-the-silent-killer-feature-of-http-2/)
- [Istio 性能文档（sidecar 延迟）](https://istio.io/latest/docs/ops/deployment/performance-and-scalability/)
- Google — [The Tail at Scale（CACM）](https://cacm.acm.org/research/the-tail-at-scale/)

**"别过早优化"阵营**

- Martin Fowler — [Microservice Premium](https://martinfowler.com/bliki/MicroservicePremium.html) / [MonolithFirst](https://martinfowler.com/bliki/MonolithFirst.html)
- Oz Nova — [You Are Not Google](https://blog.bradfieldcs.com/you-are-not-google-84912cf44afb)
- O'Reilly — [Do you need a service mesh?](https://www.oreilly.com/content/do-you-need-a-service-mesh/)

