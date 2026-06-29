---
style: summer

title: Xray Reality 突然连不上？一次 connection reset by peer 的完整排查

tags:
  - xray
  - reality
  - vpn
  - gfw
  - 网络排查
  - tls
---

某天早上梯子突然挂了。本地客户端一直报 `connection reset by peer`，第一反应是"完了，IP 被封了，得换机器"。但真按流程一步步查下来，发现根本不是 IP 的问题，而是 Reality 借用的伪装域名（SNI）被污染了——改一个字段、重启进程，两分钟就好了。

这篇把整个排查过程原样记下来。比起最后那个一行的结论，**排查时怎么一层层缩小范围、怎么用一个细节推翻错误假设**，才是真正有参考价值的部分。

<!-- more -->

## 一、先看懂症状：RST 不等于"IP 被封"

排查的第一步不是动手，是**先读懂报错**。很多人一看连不上就认定"IP 被墙了"，但 TCP 层的不同失败方式，含义完全不一样：

| 现象 | 含义 | 典型原因 |
| --- | --- | --- |
| `timeout` / 无响应（SYN 石沉大海） | 包被直接丢弃 | 整个 IP 被 GFW 黑洞路由 / 防火墙 DROP |
| `connection refused`（SYN 立即收到 RST） | 端口没在监听 | 服务挂了 / 端口配错 |
| `connection reset by peer`（握手后才被 RST） | TCP 三次握手成功了，但在 TLS 握手或传数据时收到 RST | GFW 基于流量特征的 RST 注入，或服务端协议/认证不匹配主动断开 |

我遇到的是**第三种**。这个细节很关键，它当场就排除了一半可能：

- 整个 IP 被黑洞，表现应该是**全端口 timeout**，而不是 RST；
- 端口没监听，应该是**立即 refused**，也不是握手后才 RST。

所以 `connection reset by peer` 说明：**TCP 层是通的，能连到服务器**，问题出在更上层——要么 GFW 识别出代理流量主动打断，要么服务端协议/握手出了问题。

> 一句话：纯粹的"IP 被封"通常是全端口 timeout，而不是 RST。所以"我 IP 被封了"这个怀疑，从症状上看就不太成立。

## 二、排查总思路：分层定位

把可能性从最容易排除的往下排：

```
本地 / ISP 问题
  → 服务器死活
    → Xray 服务
      → 网络可达性（国外对照）
        → 墙的封锁（国内多地）
          → 封 IP / 封端口 / 封协议
```

逐层往下，每一层都用一个命令做"分水岭判断"。

## 三、动手排查

### Step 0 — 先排除"是自己这边的问题"

封 IP 之前，先确认不是本地网络抽风：

- **换网络**：用手机蜂窝流量开热点再连一次。手机能连、家里宽带不能连，那就是本地 ISP / 路由器 / DNS 的事，跟服务器无关。
- **换时间**：GFW 在某些时段会临时加强，过几小时可能自己恢复。

### Step 1 — 确认 TCP 到底通不通

直接用 `nc` 探一下代理端口（这一步不需要真的代理）：

```bash
nc -vz YOUR_SERVER_IP 443
# Connection to YOUR_SERVER_IP 443 port [tcp/https] succeeded!
```

国内国外都 `succeeded`。**这一下就把"IP 被封"基本推翻了**：

- ❌ 整个 IP 被黑洞 → 排除（那样国内会 timeout）
- ❌ 端口被封 → 排除（443 国内能连上）

`nc` 成功 = TCP 三次握手完成 = IP 和 443 端口从国内完全可达。

### Step 2 — 确认服务端进程是活的

SSH 登上服务器看 Xray。先按官方安装路径找，结果扑了个空：

```bash
systemctl status xray
# Unit xray.service could not be found.

cat /usr/local/etc/xray/config.json
# No such file or directory
```

别慌，换 `ps` 直接看进程：

```bash
ps aux | grep xray
# root  <PID>  ...  bin/xray-linux-amd64 -c bin/config.json
```

两个信息很关键：

1. 进程从前一天一直跑到现在，没崩，`*:443` 在监听，能 accept 连接 → **Xray 服务本身是好的**。
2. 它**不是 systemd 装的**，是手动跑的裸进程，二进制叫 `xray-linux-amd64`（GitHub release 原始名），配置用的是**相对路径** `bin/config.json`——所以官方路径当然找不到。

> 顺带一个实用技巧：裸进程的配置在哪？用 `/proc/<PID>/cwd` 反查它的工作目录最准——这是个符号链接，指向进程启动时的当前目录，比 `find` 满盘搜更可靠，不会认错到某个残留的示例 / 备份配置。
>
> ```bash
> PID=$(pgrep -f xray-linux-amd64)
> CFG="$(readlink /proc/$PID/cwd)/bin/config.json"
> ls -l "$CFG"   # 这才是正在跑的进程真正加载的那份
> ```

到这里，IP 没被封、端口通、服务活着、进程健康。那 RST 从哪来？

## 四、关键转折：RST 一定来自更上层

既然 TCP 握手能完成，`connection reset by peer` 一定发生在 **TLS / 代理协议握手阶段**。此时只剩两个嫌疑：

| 嫌疑 | 说明 | 特征 |
| --- | --- | --- |
| A. 客户端↔服务端配置不匹配 | UUID / 协议 / 密钥 / SNI 对不上，服务端 accept 后主动 reset | 最近改过客户端配置 |
| B. GFW 协议指纹 RST 注入 | 墙放过 TCP 握手，等看到代理流量特征后注入 RST | 长期能用、突然就挂；`nc` 通但真正代理被重置 |

这里有个**容易被忽略的盲点**：`nc` 只做 TCP 握手、不发任何代理数据，所以它通过**不能**排除 B。GFW 的协议层 RST 恰恰是"TCP 放行、数据流被打断"，表现就是 `nc` 成功但实际代理 `connection reset by peer`。

### 一个能直接定性的推理

这台服务端跑的是 **VLESS + XTLS-Reality**。Reality 有个特性能帮我们一刀切开 A 和 B：

> **如果客户端的 key / SNI 跟服务端对不上，Reality 服务端不会 reset，而是把你当成"普通访客"，直接把流量转发给真实的伪装网站（fallback）。**

也就是说，如果是配置不匹配（嫌疑 A），看到的应该是**正常握手 + 拿到真实伪装站的页面/证书**，**而不是 RST**。

我拿到的是 RST → **基本排除配置不匹配，就是 GFW 在 TLS 层动手**。剩下的两种都指向 GFW：

| 真正原因 | 机制 | 修复 |
| --- | --- | --- |
| ① 借用的 SNI 被墙了 | GFW 看 ClientHello 里的 SNI 域名，命中重置名单就 RST，不管你连哪个 IP | 换 SNI，1–2 分钟，免费 |
| ② IP 被重点标记 | TCP 能连（所以 `nc` 通），但 GFW 对这个 IP 在 TLS 层做选择性重置 | 换 IP，~15 分钟 |

修复速度差很多，所以要先分流。

## 五、30 秒分流：测 SNI 有没有被污染

服务端配置里借用的伪装域名（`realitySettings` 的 `serverNames` / `target`）当时是 `www.microsoft.com`。思路是：**在国内直接连这个真实域名本身**，看会不会被重置。

这里踩了个小坑，**值得专门记一下**，第一次的命令是错的：

```bash
curl -v --max-time 10 www.microsoft.com   # ← 错误示范
# * Host www.microsoft.com:80 was resolved.   ← 走了 80 端口
# * Trying [2600:...]:80...                    ← 还走了 IPv6
# > GET / HTTP/1.1
# * Operation timed out ...                    ← 在 HTTP 层超时
```

漏了 `https://`，curl 默认走了 **80 端口的明文 HTTP**，还解析到了 IPv6。而 Reality 的 SNI 是写在 **443 端口 TLS 握手的 ClientHello** 里的，**80 端口的 HTTP 根本不带 SNI**——这个 timeout 跟"SNI 有没有被污染"完全是两码事。这一步必须打到 443 的 TLS 层才有意义：

```bash
# 国内：看这个域名的 443 TLS 握手会不会被重置
curl -v --max-time 10 https://www.microsoft.com/
openssl s_client -connect www.microsoft.com:443 -servername www.microsoft.com -tls1_3 </dev/null

# 服务器：确认能直连候选的新域名、且支持 TLS1.3
openssl s_client -connect www.apple.com:443 -servername www.apple.com -tls1_3 </dev/null 2>/dev/null | grep -i 'Verify return'
# Verify return code: 0 (ok)
```

判读：

- 连真实域名就被 reset / 卡死 → **SNI 被污染了**，这就是元凶 → 换 SNI（最快）。
- 连真实域名正常 → SNI 没问题，那是 IP 被标记 → 换 IP。

## 六、根因与修复：换 SNI

结论落在 ①：`www.microsoft.com` 这个伪装域名被污染了。顺带说，`www.microsoft.com` 本来就不是个好的 Reality SNI——它挂在 CDN 上、节点行为不稳定，社区早就不太推荐拿它做伪装。

修复只动**两个字段**，Reality 的 `privateKey` / `shortIds` 完全不用动：

```jsonc
"realitySettings": {
  "show": false,
  "target": "www.apple.com:443",        // ← 改（原 www.microsoft.com:443）
  "xver": 0,
  "serverNames": ["www.apple.com"],     // ← 改（必须和 target 域名一致）
  "privateKey": "……保持不动……",
  "shortIds": [ "……保持不动……" ]
}
```

> **最关键的一点：不能只改 `target`，必须连 `serverNames` 一起改。**
> 真正决定"GFW 看到什么 SNI"的是 `serverNames`——客户端发出去、墙实际看到的就是它。`target` 只是服务端去借真实站点握手的地方。只改 `target`、留着 `serverNames` 是 microsoft，客户端 SNI 还是被污染的那个，照样 RST。

校验配置语法后重启裸进程（没有 systemd，用 `nohup`）：

```bash
PID=$(pgrep -f xray-linux-amd64)
DIR=$(readlink /proc/$PID/cwd)
cd "$DIR"
./bin/xray-linux-amd64 -test -c ./bin/config.json   # 必须看到 Configuration OK
kill $PID
nohup ./bin/xray-linux-amd64 -c ./bin/config.json >/var/log/xray.log 2>&1 &
sleep 1; ss -tlnp | grep ':443'                     # 确认重新监听上
```

最后**客户端同步把 SNI 改成同一个** `www.apple.com`，`publicKey` / `shortId` / `flow`（`xtls-rprx-vision`）/ UUID / 端口 / IP 全部不动。

连上了。整个修复，从确诊到恢复，两分钟。

## 七、如果换 SNI 还是 RST（备用路线）

那就不是 SNI 而是 IP 在 TLS 层被针对了，换 IP：

- VPS 后台打 **Snapshot → 用快照 Deploy 新实例**，拿到新 IP；客户端只改地址，Reality 配置全不变。
- 顺便**换机房**（东京 / 大阪对大陆线路更好），别再用同一个被扫烂的 IP 段；换完建议用 `xray x25519` 生成一对**新 Reality 密钥**，等于全新身份。
- 老是被封就上**中转机**或 **Cloudflare CDN** 套前面，让墙永远看不到真实落地 IP。

## 八、复盘：这次排查最值钱的几个判断

1. **先读报错再动手。** `connection reset by peer`（握手后 RST）和 `timeout`（全丢）、`refused`（没监听）含义完全不同，光这一点就排除了"IP 被封"。
2. **`nc` 通 ≠ 没被墙。** `nc` 只做 TCP 握手，GFW 的协议层 RST 正好是"放过 TCP、打断数据"，所以 `nc` 成功反而符合"被墙"的特征。
3. **用 Reality 的 fallback 特性区分配置问题和墙。** 配置不匹配会 fallback 到真实站（不会 RST），收到 RST 就基本锁定是 GFW。
4. **测 SNI 一定要打到 443 的 TLS 层。** 漏了 `https://` 测成 80 端口 HTTP，结论完全无效。
5. **改 SNI 要 `target` 和 `serverNames` 一起改，客户端同步改。** 漏一个就白改。

最后那个修复只是一行字段，但能两分钟定位到它，靠的是前面每一步都用一个命令把范围砍掉一半。排查比修复值钱，记下来。
