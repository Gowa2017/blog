---
title: Github上的MySQL高可用
categories:
  - 数据库
date: 2019-06-08 21:42:11
updated: 2019-06-08 21:42:11
tags: 
  - MySQL
---
Github 使用 MySQL 作为主要的数据库，其可用性是非常的严格的。对于 Github 网站自身，其 API 入登录认证等都需要访问数据库。我们必须运行多个 MySQL 群集来服务不同的业务和任务。我们的群集使用传统的​主从​设置，在这种设置中，一个群集中的一个节点（Master）接受写操作。剩下的群集节点异步的从 Master 重放变更并提供读流量。
<!--more-->

Master 的可用性非常的严格。如果没有了 Master，一个群集就无法接受写操作：所有需要持久化的数据都无法持久化。所有涉及道数据变更的操作如：提交、问题、用户创建、审阅等都会失败。

为了支持写操作我们很明显，需要一个可用的可写入节点。同样重要的是，我们需要能识别和发现那个节点。

在失败的情况下，在 Master 崩溃的情况下，我们必须确保一个新的 Master 的存在，且其能快速的宣告自身。用来检测失败，故障转移，宣告新 Master 身份的时间就构成了所有的停服时间。

这篇文章展示了 Github 上的 MySQL　高可用及 Master 服务的发现解决方案，此方案可以让我们可信的执行一个跨数据中心操作，容忍数据中心隔离，同时很短的时间进行故障切换。

# 高可用目标

在想要实现高可用之前，我们需要问自己几个问题，以便我们更好的进行决策。例如：

- 能容忍多久的宕机时间？
- 故障检测的可靠性如何？能否容忍误报（过早的故障迁移）？
- 故障迁移的可靠性如何？何处可能出现失败？
- 解决方案跨数据中心工作起来怎么样？在低延迟和高延迟网络情况下的表现如何？
- 方法能否容忍一个完全的数据中心宕机或网络隔离？
- 用什么算法来避免脑裂（在一个群集内两个服务器都宣告自己是 Master，相互都不知道对方可以接受写操作）？
- 能否接受数据丢失？

# 从基于 VIP 和 DNS 的发现迁移

在之前的实现中，我们使用：

- [orchestrator](https://githubengineering.com/orchestrator-github/) 来检查故障和故障迁移
- 使用 VIP 和 DNS　来发现 Master

client 使用一个域名，如 *mysql-writer-1.github.net* 来发现可写节点。这个域名会解析到一个 VIP（绑定在 Master 主机上）。

这样，在正常情况下， clients 只需要解析域名，连接到解析后的IP就可以了。

下面三复制的拓扑，分布在三个数据中心：

![](https://github.blog/wp-content/uploads/2018/06/sample-3dc-topology.png)

如果 Master 宕机，在复制中我们必须将一个提升为 Master。

*orchestrator* 将会检查故障，提升一个新 Master，接着重新设置域名/VIP。所有的 Client 都不知道 Master已经变更：它们都只拥有一个标示 Master 的域名，这个域名现在解析到了一个新的 Master。然而，我们思考一下：

VIP 是协作性的：其被所有的数据库服务器拥有和宣告。为了获取或者释放一个VIP，一个服务器必须发送一个 ARP 请求。拥有这个 VIP 的服务器必须在新的 Master 需要之前释放这个 VIP。这会有一些影响：

- 一个有序的故障迁移操作会首先联系宕机的 Master 并要求其释放 VIP，然后联系新的 Master 让其使用这个 VIP。但是，如果老的 Master 不可达或者拒绝释放 VIP 怎么办？
    - 我们可能会出现脑裂的情况：两台主机都宣告拥有相同的 VIP。不同的 Client 会连接到不同的主机去，这依赖于其最短网络路径。
    - Master 依赖于两个个独立的服务器来相互协作确定，这是不可靠的。
- 及时老的 Master 确实配合，但是这个工作流程浪费了时间：我们需要在切换到新的 Master 前等待联系老的 Master 进行响应。
- 例如 VIP 改变这样的事件，已经存在的 client 与 master 之间的连接不保证会从老服务器断开，依然可能出现脑裂。

在我们的部分配置中，VIP 受物理位置的约束。他们被一个路由器或者交换机所拥有。因此，我们只能重新将 VIP 配置到位置相知的服务器上。实际上，当我们无法将 VIP 配置到另外一个数据中心提上来的 Master 时，就不得不做 DNS 的变更。

- DNS 变更的传播需要时间。clients 会缓存 DNS 的解析结果一定的时间。那么一个跨数据中心的迁移就会花费更多的时间。

上面这些限制已经足够推动我们去研究新的解决方案了，但我们需要更多的考虑：

- 通过 *pt-heartbeat* 服务，Master 间通过心跳来进行自注入，这是为了进行[滞后测量和节流控制](https://githubengineering.com/mitigating-replication-lag-and-reducing-read-load-with-freno/)。在新的 Master 上，这个服务必须踢出。可能的话，这个服务也会在老的 Master 上关闭。
- 同样 *[Pseudo-GTID](https://github.com/github/orchestrator/blob/master/docs/pseudo-gtid.md)* 注入由 masters 自身管理。在新的 Master 上踢出，当然，完美实现的话还需要在 老的 Master 上关闭。
- 新的 Master 会被设置为可写。可能的话，老的 Master 设置为只读。

这些额外的步骤是导致总停机时间的一个因素，并引入了自己的故障和摩擦。

上面这个解决方案能工作，Github 在这种方案下也成功的工作了，但是我们希望我们的 HA 在以下方面进行提高：

- 数据中心不可知
- 容忍数据中心故障
- 移除不可靠的协同工作流程
- 减少宕机时间
- 进可能少的故障迁移。

# Github HA：orchestrator, Consul, GLB

在新的方案中，解决了上面的大多数问题，且有了一些附带的提升。当前的 HA 配置是：

- [orchestrator](https://github.com/github/orchestrator) 进行检测和故障迁移。我们使用一个跨数据中心的 [orchestrator/raft](https://github.com/github/orchestrator/blob/master/docs/raft.md) 配置。下面会介绍。
- [Consul](https://www.consul.io/) 来进行服务发现。
- [GLB/HAProxy](https://githubengineering.com/glb-director-open-source-load-balancer/) 在 clients 和可写节点间作为一个代理层。 [GLB 是开源的](https://github.com/github/glb-director)。
- *anycast* 进行网络路由

![](https://github.blog/wp-content/uploads/2018/06/mysql-ha-solution-at-github.png)

## 正常流程

正常请下，通过 GLP/HAProxy 连接到可写节点。

应用永远不会意识到 Master 的变化。和之前的方案一样，他们使用一个域名。例如，对于 *cluster1* 的 Master 可能是 *mysql-writer-1.github.net*。在我们当前的配置中，这个域名被解析到一个 *anycast* IP。

使用 *anycast* 技术后，域名会解析到一个IP地址，但是流量却是根据 client 的位置进行路由。实际上，在每个数据中心我们都有 GLB（高可用负载均衡器，部署在多个盒子上）。到 *mysql-writer-1.github.net* 的流量总是会路由到当地数据中心的 GLB 群集上。也就是说，所有 clients 都由 本地代理提供服务。

我们在 *HAProxy* 之上运行 GLB。 HAProxy 有写入池：每个 Mysql Cluster 一个池，每个池只有一个后端服务器： cluster 的 Master 服务器。所有数据中心的 GLB/HAProxy 都有相同的写入池。因此，如果应用想要连接到 *mysql-writer-1.github.net* 的时候，其不用关心是连接到了哪个 GLB。其总是会被路由到 *cluster1* 的 Master 节点。


对于应用而言，在 GLB 处就停止了服务发现，也不需要重新进行发现。把流量路由到正确的地方这是 GLB 的责任。

GLB 是如何知道哪些服务器作为后端的？我们如何在 GLB 间传递修改呢？

## 通过 Consul 进行发现

Consul 因其服务发现解决方法而知名，同时也提供了 DNS 服务。在我们的解决方案中，我们只将其使用为一个高可用键值存储。

我们将每个群集的 Master 都写到这个 Consul 的键值存储中。对于每个群集，会有一系列的 KV 项来表示这个群集 Master 的 *fqdn*，port, ipv4, ipv6。

每个 GLP/HAproxy 都运行 [consul-templates](https://github.com/hashicorp/consul-template)：一个监听 Consul 数据变更的服务。*consul-template* 会产生一个有效的配置文件，并能根据这个文件的变更来重新加载 HAProxy。

这样，Consule 中每个 Master 信息的变更都会被 GLB/HAProxy 观察到，并以此来重新配置，将新的 Master 配置为一个群集中的唯一项，接着重新加载来反应这些变化。

在 Github 中，我们在每个数据中心有一个 Consul 配置，每个配置都是高可用的。然而，这些配置与其他数据中心是相互独立的。他们不会相互复制和共享数据。

那么如果在数据中心之间传播 Consul 的数据呢？

## orchestrator/raft
我们使用一个 *orchestrator/raft* 配置：*orchestrator* 节点通过 raft 共识算法来与其他节点通信。在每个数据中心中我们有一个或两个 *orchestrator* 节点。

*orchestrator* 的作用是进行故障检测，MySQL 故障迁移，已经将 master 的改变通知到 Consul。故障迁移由一个 *orchestrator/raft* 领导节点来操作，但是 master 的变更信息通过 *raft* 算法在 *orchestrator* 节点间进行传播。

一个 *orchestrator* 节点收到 master 变更的通知后，会与其本地的 Consul 进行通信：都会在 Consul KV Store 中写入一个信息。拥有多个 *orchestrator* 节点的数据中心会对 Consul 有多个写入。

## 流程总览

在一个 Master 崩溃的情况下：

1. *orchestrator* 检测到故障
2. *orchestrator/raft* 开始进行恢复。选举出一个新的 Master
3. *orchestrator/raft* 会将 Master 的变更宣告到所有的 *raft* 群集节点
4. 每个 *orchestrator/raft* 成员收到一个领导节点的变更通知。然后他们会以收到的 Master 信息来写入到本地的 Consule 的 KV Store 中。
5. 每个 *GLB/HAProxy* 都有一个 *consul-templating* 在运行，他们会观察  Consul KV Store 的变化，并重新配置和加载 *HAProxy*
6. Client 的流量就会被重定向到新的 Master

这样，每个组件都有其清晰的任务，整个设计进行了解耦并且非常简单。*orchestrator* 不知道 GLB 的存在。 Consul 也不需要知道信息是从哪里来的。HAProxy 只需要关心 Consul。 Client 只需要关心 HAProxy。

然后，我们这里就少了一些其他的需要：

- 不需要 DNS 变更的传播
- 没有 TTL
- 不需要故障的 Master 的配合。

## 更多细节

为了让这个流程更安全，我们还做了下面的事情：

- HAProxy 配置了一个非常低的 *hard-stop-after* 属性。当其在一个 写入池 中重新加载一个后端服务器的时候，其会自动终止所有在老的 Master 上的连接。
    - 配置了 *hard-stop-after* 后，我们就不再需要与客户端进行协作，同时这个减少了出现脑裂的概率。
- 我们不需要 Consul 保持一直可用。实际上，只需要在故障迁移的时候 Consul 可用就行了。如果 Consul 碰巧也不工作了，GLB 会在最后已知的那个服务器上进行操作，而不会有什么激烈的变化。
- GLB 设置来识别新变更的 Master。与我们的 [context-aware MySQL pools](https://githubengineering.com/context-aware-mysql-pools-via-haproxy/) 类似，其会在后端做一个检查来确认其是否需要一个可写节点。如果我们不小心从 Consul 中删除了 Master 的信息，没问题；空的项目会被忽略。如果我们在 Consul 中写入了一个不是 Master 的信息，也没有问题； GBL 会拒绝更新。


## orchestrator/raft 故障检测

*orchestrator* 使用一个整体的方式来检测故障，因此变得非常可靠。我们不会观察到误报：因此不会出现过早的迁移，因此也不会增加额外的宕机时间。

*orchestrator/raft* 进一步解决了整个数据中心网络隔离的问题。一个 DC 网络隔离会引起混乱：在 DC　内部的服务器间可以相互通信。是当前 DC 与其他网络隔离，还是其他的 DC 被网络隔离了？

在 *orchestrator/raft* 中，*raft* 领导节点是唯一进行故障迁移的节点。一个领导节点是获得了组内大多数支持的那个节点。我们 *orchestrator* 节点的部署，不会让任何一个数据中心投票就成为领导节点，而必须剩下的 n -1 个数据中心共同来决定。

在一个 DC 网络隔离的情况下，当前 DC 内的 *orchestrator* 节点将会与其他 DC 间的节点断开连接。那么，当前DC 内的 *orchestrator* 就不会成为 *raft* 群集的领导节点。如果有这样的节点成为了领导节点，那他将会被踢下去。其他的 DC 会重新选取一个领导节点。

那么在隔离的 DC 内需要一个 master 么。*orchestrator* 会将其他正常 DC 内的服务器作为其 Master。我们通过将决策委托给非隔离DC中的领导者来缓解DC隔离。

## 快速宣告

更快的宣告 Master 的变更可以减少宕机时间？那么这如何达到。

当 *orchestrator* 开始故障迁移，其观察将要被提升的那些服务器。其了解复制规则且遵守提示和限制，以此来作出最佳的决定。

其会了解，一个将要提升的服务器也是一理想候选人：

- 没有什么会阻止选举这个服务器（有可能用户已经将此服务器设置为首选）
- 此服务器可以作为其他所有兄弟服务器的副本

当遇到这样的服务器时，即使需要异步开始修复复制树（这个操作通常需要几秒钟。），*orchestrator* 也会将这个服务器设置为可写，将这个进行宣告（在我们的场景下是写到 Consul KV Store 中）。

很可能在我们的GLB服务器完全重新加载时，复制树已经完好无损，但并不是严格要求的。服务器可以很好接收写入！

## 半同步复制

在 [MySQL的半同步复制中](https://dev.mysql.com/doc/refman/en/replication-semisync.html)，如果一个事务进行提交，只有提交所产生的变化已经被至少一个 Slave 库所知晓，Master 才会进行确认。其提供了一个达到无损故障迁移的方式：对于 Master 的变化，要么已经完成，要么等待应用到 slave 库上。

一致性带来成本：可用性风险。如果没有 slave 确认收到了变化，master 就会阻塞，写入就会停止。幸运的是，这里有一个超时设置，在经过了这个设置的时间后，主服务器可以恢复到异步复制模式。

我们把这个超时时间设置为：500ms。这个时间完全足够将变化在 当前 DC内进行传播，当然，到其他 DC 间也是可能够的。

我们在本地DC副本上启用半同步，并且在 Master 死亡的情况下，我们期望（尽管不严格执行）无损故障转移。完全DC故障的无损故障转移成本很高，我们不期望它。

在尝试半同步超时时，我们还观察到了一种对我们有利的行为：在主服务器故障的情况下，我们能够影响理想候选人的身份。通过在指定服务器上启用半同步，并将其标记为候选服务器，我们可以通过影响故障结果来减少总停机时间。
## 心跳注入

我们选择一直运行 *pt-heartbeat* 服务，而不是只是 选举/踢出 Master 的时候进行 启动/停止。这个需要打个补丁来让 *pt-heartbeat* 服务适应服务器在 *read-only* 状态的变化或者完全的崩溃。

当前 *pt-heartbeat* 服务运行在 Master 和 slave 上。在 Master 上，他们产生心跳事件。在 slave 上，他们确认当前的服务器是 *read-only*，和进行常规的状态检查。当一个服务器被选举为 Master 的时候， *pt-heartbeat* 会检查此服务器是否为可写，同时开始产生心跳事件。
## orchestrator 权限委托

同时我们让 *orchestrator* 来做下面的事情：

- 伪GTID注入
- 设置被选举为 Master 服务器为 可写，清楚其 slave 状态
- 可能的话，设置老的 Master 为 read-only

在 新的 Master 上，这会减少摩擦。被选举为 Master 的服务器很明显是需要在线的和可用的，不然我们就不会选举它了。因此，让orchestrator直接将更改应用于提升的 Master 是有道理的。

## 限制与缺点

代理层让应用不清楚 Master 的身份，但同时其也向 Master 屏蔽了 应用的身份。Master 看到的所有连接都是从代理来的，我们丢失了实际的连接源头信息。

随着分布式系统的发展，我们仍然处于未处理的情况。

需要注意的是，在一个数据中心隔离的情况下，我们假设 Master 就在这个数据中心中，在这个数据中心的应用依然可以向这个 Master 写入。当网络恢复的时候，这可能会导致数据的不一致性。我们努力通过在一个隔离的DC中 实现一个可靠的 [STONITH](https://en.wikipedia.org/wiki/STONITH) 来减少脑裂的情况。和之前一样，在我们将 Master 关闭前可能会有短时间的脑裂。避免脑裂的操作成本非常高。

还有些场景：故障迁移时 Consul 停止；局部的 DC 隔离；等等。分布式的系统不可能把所有的问题都解决，所以只能先解决最重要的。
## 结果

我们的 *orchestrator/GLB/Consul* 提供了我们如下功能：

- 可靠的故障检测
- 数据中心不敏感的故障迁移 
- 通常是无损故障转移
- 数据中心隔离的支持
- 减少脑裂
- 不依赖协作
- 多数情况下 10-13秒的宕机
