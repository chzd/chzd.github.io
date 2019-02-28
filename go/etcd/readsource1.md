## RaftExample
我们先通过RaftExample来初步了解raft的工作流
* raft goroutine And application goroutine

### contrib/raftexample
raft provides a commit stream for the proposals from the http api.

The rafteexample project consists of three componets: a raft-backed **key-value store**, *a REST API server*, and a *raft consensus server* base on etcd's raft implementation.

下面简要说明raftexample的设计实现。
底层存储是一个kvstore的对象，该对象消费通道commitC过来的kv数据，正常服务时，通过commitC传递来的数据来自于用户通过restAPI提交的数据kvdata. restAPI 模块接到用户的请求时，调用kvstore.Propose(k,v),kvstore.Propose并不会将数据直接写入自身的map中，而是通过通过ProposeC传递给raftnode. raftnode调用rc.node.Propose().`raft state machine` 接受了该次数据提交，之后,会通过rc.node.Ready()返回的chan返回。返回的信息不仅仅是本次提交的kv信息，还包括了hardstate信息，以及存储了kv的entrry，：
**rd.HardState**: `[{"term":10,"vote":1,"commit":24}]`, **rd.Entries**:`[[{"Term":10,"Index":24,"Type":0,"Data":"IP+BAwEBAmt2Af+CAAECAQNLZXkBDAABA1ZhbAEMAAAAEP+CAQYva2V5MTUBA2ZvbwA="}]]`。所有的这些信息会被追加到rc.raftStorage中，raftStorage中存储了上次快照保存之后所有commits。`rc.node.Ready()`返回数据之后。会通过rc.transport.Send(rd.Message).然后`rc.publishEntries(rc.entriesToApply(rd.CommittedEntries)`,把本次要提交的数据通过rc.publishEntries传给通道commitC，由kvstore存储到其map中供查询。

集群管理使用的元数据采用了类似于redis集群的元数据。
**Term**: 
初始状态为1，进行一次选举之后变为2。
**Index**:
用于标识用户的一次提交。如果提交正常。

##### 启动的步骤

* `rc.startRaft()`, 判断是否有wal，如果有的话，调用·`rc.wal = rc.replayWAL()`, 如果有，说明该节点是在进行重启，所以用创建的config对象重启该raftnode`raft.RestartNode(c)`.如果不是，则启动节点`raft.StartNode`。
    * RestartNode is similar to StartNode but does not take a list of peers. The current membership of the cluster will be restored from the Storage.
* `go rc.serveRaft()`, 解析该节点对应的peer中的url 绑定端口提供服务，2379 端口用于raft之间的协议通信。
* `go rc.serveChannels()`: 
    * 循环1，处理restAPI提交的propose；处理配置变更
    * 循环2，每100ms调用rc.node.Tick(); store raft entries to wal, then publish over commit channel,`rc.publishEntries(ents)`, 该函数在publish结束时，会发送nil数据to signal replay has finished.

* `newKVStore` 函数直接调用kvstore的readCommits(commitC, errorC),然后才是go kvstore的readCommits，最终返回kvstore.
* `readCommits` 中 执行`for data := range commitC {...}` 读取`commitC`通道传递过来的数据。直到收到`nil`并且`kvstore.snapshotter.Load()`返回`snap.ErrNoSnapshot`时，当前函数才会退出。
    * 哪些情况下会收到nil
    1. 启动时，raftnode 调用replayWAL 时，如果if len(ents)==0 {rc.commitC <- nil}, 如果节点时心启动，没有历史数据，应该会发送nil，而且不存在快照，所以kvstore能顺利返回。
    2. rc.publishSnapshot() 时会发送nil 通知kvstore to load snapshot。
    3. go rc.serveChannels() -> rc.publishEntries(ents) 如果`ents[i].Index == rc.lastIndex`, rc.commitC <- nil。

#### Snapshot
是快照，但是需要弄清楚。
1. main-> newRaftNode 返回一个`snapshotter *snap.Snapshotter`,
2.startRaft 时，创建了一个 snapshotter
```
rc.snapshotter = snap.New(zap.NewExample(), rc.snapdir)
rc.snapshotterReady <- rc.snapshotter
```
3. defination 很简单。
```
type Snapshotter struct {
	lg  *zap.Logger
	dir string
}
```

#### Storage
**log entries** 存储上次快照之后，集群数据的操作日志。
Storage 在etcd中的存在意义还没有搞清楚。**TODO**
```
// Storage is an interface that may be implemented by the application
// to retrieve log entries from storage.
//
// If any Storage method returns an error, the raft instance will
// become inoperable and refuse to participate in elections; the
// application is responsible for cleanup and recovery in this case.
```
* dummy entry 就是slice中的第一个元素，只保存了term和index信息。
* entry.Index。

##### MemoryStorage
存储commit 日志

```
// NewMemoryStorage creates an empty MemoryStorage.
func NewMemoryStorage() *MemoryStorage {
	return &MemoryStorage{
		// When starting from scratch populate the list with a dummy entry at term zero.
		ents: make([]pb.Entry, 1),
	}
}
```
* `ApplySnapshot`: 重新初始化了 ents: `ms.ents = []pb.Entry{{Term: snap.Metadata.Term, Index: snap.Metadata.Index}}`

* `Append` 操作的index以及offset的操作需要好好的计算一下，后面找时间看看。**TODO**

#### Ready
Ready encapsulates the entries and messages that are read to read, be saved to stable storage, `committed` or to other peers. All fields in Ready are read-only.

raft.node 通过ReadyC 返回本节点的commit log，写入本地wal，然后写`rc.raftStorage.ApplySnapshot(rd.Snapshot)`(如果有的话),然后，`rc.raftStorage.Append(rd.Entries)`, 然后写入transport, 然后publishEntries(). 
`publishEntries()` 将`raftpb.EntryNormal`的 ent放入rc.commitC, 供kvstore进行消费。


#### Put Key-Value
1. restAPI `../key XPUT -d boo`, `httpKVAPI.store.Propose(key, value)`, `kvstore.proposeC <- buf.String()`, 
2. `go rc.serveChannels()`, 循环读取proposeC的数据。`rc.node.Propose(context.TODO(), []byte(prop))`
3. 上面的rc.node: 
```
func newNode() node {
	return node{
		propc:      make(chan msgWithResult),
		recvc:      make(chan pb.Message),
		confc:      make(chan pb.ConfChange),
		confstatec: make(chan pb.ConfState),
		readyc:     make(chan Ready),
		advancec:   make(chan struct{}),
		// make tickc a buffered chan, so raft node can buffer some ticks when the node
		// is busy processing raft messages. Raft node will resume process buffered
		// ticks when it becomes idle.
		tickc:  make(chan struct{}, 128),
		done:   make(chan struct{}),
		stop:   make(chan struct{}),
		status: make(chan chan Status),
	}
}
```
node.run(raft)
4. node.Propose()


### show me the code
*  The store updates its map once raft reports the updates are committed.
*  When raft reaches a consensus, the server publishes all committed updates over a commit channel
*  

### problems encounterd
* Not clean raftexample-* files after running single node raftexample. Run `goreman start` to start the raftexample local cluster. I found that raftexample1 dose not exchange data with other nodes.
    * 通过日志跟踪发现 首先启动的example1 的term为2，杀掉后通过goreman启动三个节点的集群， 包括example1, 此时 example1的term变成了3， 当其他的node发送消息过来时，就会忽略了。
```
16:29:05 raftexample1 | raft2019/02/27 16:29:05 INFO: 1 [term: 3] ignored a MsgHeartbeat message with lower term from 2 [term: 2]
```
