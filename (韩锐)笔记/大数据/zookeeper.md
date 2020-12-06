znode一共有4种类型：持久的（persistent）、临时的 （ephemeral）、持久有序的（persistent_sequential）和临时有序的 （ephemeral_sequential）。

通知（notification）机制：客户端向ZooKeeper注册需要接收通知的 znode，通过对znode设置监视点（watch）来接收通知。监视点是一个单次触发的操作，意即监视点会触发一个通知。为了接收多个通知，客户端必须在每次通知后设置一个新的监视点。通知机制的一个重要保障是，对同一个znode的操作，先向客户端传送通知，然后再对该节点进行变更。

每一个znode都有一个版本号，它随着每次数据变化而自增。两个 API操作可以有条件地执行：setData和delete。这两个调用以版本号作为转入参数，只有当转入参数的版本号与服务器上的版本号一致时调用才会成功。版本机制为了避免多个客户端同时修改znode导致数据丢失，这个和并发里面的ABA问题比较类似。

ZooKeeper服务器端运行于两种模式下：独立模式（standalone）和仲裁模式（quorum）。独立模式几乎与其术语所描述的一样：有一个单独的服务器，ZooKeeper状态无法复制。在仲裁模式下，具有一组 ZooKeeper服务器，我们称为ZooKeeper集合（ZooKeeper ensemble），它们之前可以进行状态的复制，并同时为服务于客户端的请求。

### 1. 命令行操作

```bash
bin/zkCli.sh -server 192.168.15.123:2181

# 创建临时节点
create -e /mynode "temp node"
# 列出所有节点
ls /
# 查看节点统计信息
stat /mynode true
```

### 2. 集群角色

Zookeeper集群中有三种角色：

- Leader 由leader选举机制选取出来的角色，leader为客户端提供读写服务
- Follower 为客户端提供读服务，它会参与leader选举。当有事务处理请求时，会转发给Leader处理
- Observer 它的作用主要是为了提升读取性能，为客户端提供读服务，但不参与选举

### 3. 节点选举

#### 3.1 术语解释

- SID：服务器ID，是一个数字，集群内的机器不能重复，与myid一致
- ZXID：事务ID，用来标识一次服务器状态的变更。ZXID中高32位表示Leader的选举周期，称为epoch；低32位是操作序列号，可以理解成一个计数器。
- Vote：投票，通过投票机制来选取Leader
- Quorum：过半机器数，Leader选举中至少需要过半机器投出相同的票

#### 3.2 选举细节

这里以集群初始化场景为例子，此时集群中还没有leader，各服务器的状态为`looking`。接下来进入leader的选举流程：

- 每台服务器首先尝试将自己选举为Leader，然后向其他服务器发起投票请求
- 每台服务器在收到投票信息后，会按以下规则进行投票：
  1. 对比ZXID，以ZXID较大者为准。如果vote_zxid大于self_zxid，那么认可接收到的投票信息并将投票信息发送出去，否则不做改变
  2. ZXID相等时，以SID较大者为准。如果vote_sid大于self_sid，那么认可接收到的投票信息并将投票信息发送出去，否则不做改变
- 经过多轮投票后，如果一台服务器中收到了超过半数的相同的投票，那么该投票对应的SID服务器将被推选为Leader
- Leader服务器的状态变更为`leading`，Follower服务器状态变更为`following`

### 4. 数据写入流程

> 这段是从网上拷贝的，书中没有讲

1. 客户端发出写入数据请求给任意Follower。
2. Follower把写入数据请求转发给Leader。
3. Leader采用二阶段提交方式，先发送Proposal广播给Follower。
4. Follower接到Proposal消息，写入日志成功后，返回ACK消息给Leader。
5. Leader接到半数以上ACK消息，返回成功给客户端，并且广播Commit请求给Follower

### 参考

1. https://mp.weixin.qq.com/s/gFsWM75R7BJkTolwAInRjw （深度好文，说的是zk的缺点以及微服务场景的缺陷）