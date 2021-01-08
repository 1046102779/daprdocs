本文由三部分构成：
- 前言讲述placement服务集群的用途；
- 分析placement应用raft sdk实现raft集群的构建；
- 分析placement与actor和raft集群的通信；
- placement server的启动过程
- 总结placement

通过本文，你可以做到：
1. 了解和实现自己的raft集群分布式状态存储应用；
2. 了解actor保持强一致性是如何实现的；
3. 了解placement服务的特点和内部实现机制

## 前言

本文主要讲述dapr runtime中的placement服务源码分析，placement部署在k8s dapr-system命名空间下, 如下所示：

```shell
root:~$ kubectl get po -n dapr-system
NAME                                     READY   STATUS    RESTARTS   AGE
dapr-dashboard-78d4b778b5-6z4tq          1/1     Running   0          8d
dapr-operator-5b7d566558-bthdp           1/1     Running   0          8d
dapr-placement-59954bd646-c556x          1/1     Running   0          8d
dapr-sentry-56b6d54755-t9c5k             1/1     Running   0          8d
dapr-sidecar-injector-59ff5c6946-mrxdc   1/1     Running   0          8d
```

**dapr actor相当于mapreduce**。placement服务主要是对actor计算任务进行分布式状态管理，保证每个actor都拥有整个集群所有actor信息，这样从任何一个业务pod访问${actor_type}/${actor_id}路由去访问actor，不管这个要访问的actor是在本地还是在远端，我都可以通过本地actor存储整个集群actor信息，找到远端地址再进行rpc访问。那么这个actor存储的整个集群actor信息的获取，就是通过placement分布式集群来实现，多服务placement由raft算法构建成一个集群，然后由placement集群中的leader与整个集群中的actor进行通信，通信方式是三阶段提交集群的所有actor信息。

placement服务主要是对整个集群下所有注入了daprd Runtime的业务pod中，所有的actor需要进行分布式状态管理，这个actor分布式状态的管理就是由placement集群来完成的。

先给出各个业务pod中daprd runtime构建成的集群所有actor与dapr-system命名空间下placement集群的通信图：

![actor与placement集群的通信](https://wework.qpic.cn/wwpic/524031_8YhJfqqbSFGmMoE_1609903332/0)

## raft在dapr中的实现和应用

dapr placement集群对actor分布式状态的管理，是通过[hashicorp/raft](https://github.com/hashicorp/raft)第三方库实现，大多数golang版本的raft算法实现基本上都是使用的该hashicorp/raft库，包括consul的实现。

读者可以先了解下：[基于hashicorp/raft的分布式一致性实战教学](https://zhuanlan.zhihu.com/p/58048906), 阅读该文，读可以知道raft基本概念，以及如何实现一个基本的分布式缓存应用，最后知道dapr使用hashicorp/raft库需要实现的接口。

### dapr raft库

首先需要知道placcement在raft算法中传输的entry logs具体内容。然后再了解placement在hashicorp/raft库中有限状态机FSM接口的实现，和placement服务与raft相关的业务逻辑

#### placement raft state

placement服务中state主要结构是hashingTableMap存储结构，其他都是围绕这个结构进行CRUD操作。这个结构存储的key表示每一类actor_type类型任务，value表示hash一致性存储，所以可以看出整个集群内该每个placement都存储了分布式集群所有的actor_type任务以及每一类actor_type对应的完整actor服务地址（通过这个地址，我们可以找到业务POD）的hash一致性存储数据。

每个placement进程中raft服务节点，存储的集群actor信息包括：

```golang
   // DaprHostMember 表示Dapr运行时每个actor host成员，每个dapr runtime容器中的actor，包括多个actor_type
  type DaprHostMember struct {
      // Name表示host名称, 比如：127.0.0.1:50002, 所以这个host并不是指主机host或者IP，而是指dapr runtime host IP:PORT. sidecarInternalGRPCPort=50002内部grpc与placement集群通信的端口
      Name string
      // AppID表示Dapr runtime的appid，每个应用服务都有一个唯一的appid。是在yaml文件中配置
      AppID string
      // Entities表示业务pod向dapr runtime申请的dapr actor types。也相当于对任务进行分类
      // 一个业务pod，可以处理多个不同类型的actor_type任务.
      Entities []string

      // UpdatedAt表示actor与placement通信最后一次更新时间.
      UpdatedAt int64
  }

  // DaprHostMemberState表示业务pod含有的dapr runtime集群所有的actor信息
  type DaprHostMemberState struct {
      // Index表示raft log的索引
      Index uint64
      // Members表示集群中所有actor的信息
      Members map[string]*DaprHostMember

      // TableGeneration表示hashingTableMap更新的次数
      TableGeneration uint64

      // hashingTableMap 使用hash一致性算法提供负载均衡， 目前dapr提供的hash一致性算法有两种：1. 基于虚拟节点的负载均衡；2. 基于有界负载的hash一致性算法；目前dapr采用的是第一种方案；
      // 这个hash一致性算法使用场景：
      // 首先可以对actor计算任务进行分类为actor_type，每个actor_type分类任务做一类相同的事情.
      // 每次client请求dapr处理任务时，会首先生成一个全局唯一的actor_id，任务不执行完成，则actor_id不会销毁。然后根据client带的actor_type在一致性hash数据上找到Host，然后再通过grpc/http请求发送给本地的业务pod或者远端的业务pod处理actor_type任务。也就是说一个外部请求或者一个内部请求，需要访问actor_type类型的任务执行，则可以随便找个节点都可以找到目标服务地址。
      // 使用hashing/consistent的目的是为了降低集群各个节点的任务负载，加快任务执行速度。因为有两点原因：
      //   1. 因为如果相同的actor_type业务pod存在多个，为了业务不均衡，导致有的节点资源空闲，有的资源紧张，导致任务排队；
      //   2. actor是独立执行的，就是为了解决并发导致的资源竞争问题，所以业务pod如果存在多个actor任务需要执行，则会一个个排队执行任务。所以负载均衡变得非常重要，防止任务执行花费过得时间等待执行或者处于阻塞状态
      hashingTableMap map[string]*hashing.Consistent
  }

// updateHashingTables 更新DaprHostMemberState对象中的hashTableMap。
func (s *DaprHostMemberState) updateHashingTables(host *DaprHostMember) {
      // 前面说了一个业务POD中可以同时处理不同类型的actor_type任务
      // 那么这里的entities遍历，主要是针对不存在的actor_type哈希一致性数据对象进行创建，每一个actor_type都会有整个集群能够处理该actor_type类型任务的业务POD列表， 那么存储在DaprHostMember中的Name，就是actor服务地址端口。那么dapr runtime就可以通过内部的grpc请求，把actor_type转发给从hash一致性数据中拿到的Name，也就是actor服务处理。最后转发给本地的业务POD执行actor_type任务.
      for _, e := range host.Entities {
          if _, ok := s.hashingTableMap[e]; !ok {
              s.hashingTableMap[e] = hashing.NewConsistentHash()
          }

          // 如果hash一致性数据已存在，则把DaprHostMember数据存储到hash一致性存储中。
          s.hashingTableMap[e].Add(host.Name, host.AppID, 0)
      }
}

// removeHashingTables 从所有的hash一致性存储中删除DaprHostMember.
func (s *DaprHostMemberState) removeHashingTables(host *DaprHostMember) {
      // 同上，遍历这个业务pod上可以处理的所有actor_type类型任务，然后再在每个hash一致性存储中删除节点。
      for _, e := range host.Entities {
          if t, ok := s.hashingTableMap[e]; ok {
              // 从hash一致性存储中删除节点, 该host.Name表示actor服务地址
              t.Remove(host.Name)

              // 如果hash一致性存储的节点为0，则直接删除该结构，防止内存泄漏。
              // 因为这个placement服务是全生命周期，是属于dapr-system命名空间下的dapr-placmemt集群；处理不同actor_type业务POD的销毁和创建，会导致不同actor_type类型hash一致性存储，会随着不断新增业务导致内存泄漏。
              if len(t.Hosts()) == 0 {
                  delete(s.hashingTableMap, e)
              }
          }
      }
  }

// upsertMember 新增或者更新DaprHostMember
func (s *DaprHostMemberState) upsertMember(host *DaprHostMember) bool {
    // 判断actor业务所在pod支持的actor_type列表是否大于0， 如果小于等于0，表示当前业务pod不支持处理actor_type。
    if !s.isActorHost(host) {
        return false
    }

    // 首先判断host是否在hash一致性存储上已存在，这个通过一个map结构直接存储来校验host是否存在。
    if m, ok := s.Members[host.Name]; ok {
        // 如果已存在，其他所有信息都相同，则不需要更新或者新增操作
        if m.AppID == host.AppID && m.Name == host.Name && cmp.Equal(m.Entities, host.Entities) {
            m.UpdatedAt = host.UpdatedAt
            return false
        }

        // 如果存在host，但是DaprHostMember内容发生变化，则需要先删除，然后统一走插入逻辑，再插入到hash一致性存储
        // 这样使得插入和更新操作相同.
        s.removeHashingTables(m)
    }

    // Members做更新操作，host中针对不同的actor_type，在hash一致性存储做插入操作
    s.Members[host.Name] = &DaprHostMember{
        Name:      host.Name,
        AppID:     host.AppID,
        UpdatedAt: host.UpdatedAt,
    }
    s.Members[host.Name].Entities = make([]string, len(host.Entities))
    copy(s.Members[host.Name].Entities, host.Entities)

    // 新增DaprHostMember到各个hash一致性存储中
    s.updateHashingTables(s.Members[host.Name])

    // 更新整个集群中所有hash一致性存储的次数
    s.TableGeneration++

    return true
}

// removeMember 移除DaprHostMemberState存储中与DaprHostMember相关的Members和所有hash一致性存储。
func (s *DaprHostMemberState) removeMember(host *DaprHostMember) bool {
    if m, ok := s.Members[host.Name]; ok {
        s.removeHashingTables(m)
        s.TableGeneration++
        delete(s.Members, host.Name)

        return true
    }

    return false
}
```

#### placement raft FSM

placement raft FSM主要是实现hashicorp/consistent库中FSM接口，它一共三个方法：Apply、Snapshot和Restore, 分别的用途:

1. Apply方法用于接收raft leader发送过来的entry log，保存到raft follower状态中；
2. Snapshot方法按照一定规则(比如：每隔多长时间；传过来多少条entry log)满足条件后，才会做一次快照落地存储。防止服务挂掉重启后，从快照迅速恢复一部分数据;
3. Restore方法用于读取快照数据到内存中;

那么对于placement raft fsm主要就是针对整个集群中所有类型actor_type的hash一致性数据进行存储。

```golang
// FSM 表示raft集群每个节点的有限状态机
type FSM struct {
      stateLock sync.RWMutex
      // state 主要就是用于存储dapr整个集群中所有actor_type的hash一致性数据。
      // 也就是前面placement raft state部分的存储内容.
      state     *DaprHostMemberState
}

// State 返回整个集群的各个actor_type对应的hash一致性数据，也即actors分布式状态信息
func (c *FSM) State() *DaprHostMemberState {
    c.stateLock.RLock()
    defer c.stateLock.RUnlock()
    return c.state
}

// PlacementState 返回整个集群的所有actor状态信息，主要是协议转换为pb。
func (c *FSM) PlacementState() *v1pb.PlacementTables {
    c.stateLock.RLock()
    defer c.stateLock.RUnlock()

    // 下面的操作， DaprHostMemberState协议数据结构上和v1pb.PlacementTables协议结构基本上是一致的
    // 只是后者是pb协议
    // 创建pb协议实例
    newTable := &v1pb.PlacementTables{
        Version: strconv.FormatUint(c.state.TableGeneration, 10),
        Entries: make(map[string]*v1pb.PlacementTable),
    }

    totalHostSize := 0
    totalSortedSet := 0
    totalLoadMap := 0

    // 通过获取每个actor_type对应的hashing.Consistent数据
    entries := c.state.hashingTableMap
    for k, v := range entries {
        // 获取hash一致性数据的各个属性数据
        hosts, sortedSet, loadMap, totalLoad := v.GetInternals()
        table := v1pb.PlacementTable{
            Hosts:     make(map[uint64]string),
            SortedSet: make([]uint64, len(sortedSet)),
            TotalLoad: totalLoad,
            LoadMap:   make(map[string]*v1pb.Host),
        }

        for lk, lv := range hosts {
            table.Hosts[lk] = lv
        }

        copy(table.SortedSet, sortedSet)

        for lk, lv := range loadMap {
            h := v1pb.Host{
                Name: lv.Name,
                Load: lv.Load,
                Port: lv.Port,
                Id:   lv.AppID,
            }
            table.LoadMap[lk] = &h
        }
        newTable.Entries[k] = &table

        totalHostSize += len(table.Hosts)
        totalSortedSet += len(table.SortedSet)
        totalLoadMap += len(table.LoadMap)
    }

    logging.Debugf("PlacementTable Size, Hosts: %d, SortedSet: %d, LoadMap: %d", totalH  ostSize, totalSortedSet, totalLoadMap)

    return newTable
}

// upsertMember 新增或者更新接收到的FSM Apply内部转发过来的处理请求.
// 这个请求最初是来自actor服务发送过来的新增或者删除actor.
func (c *FSM) upsertMember(cmdData []byte) (bool, error) {
    c.stateLock.Lock()
    defer c.stateLock.Unlock()

    var host DaprHostMember
    // 对于传输过来的MessagePack消息, 反序列化为DaprHostMember,
    // 也就是业务pod支持的所有与actor相关的数据
    if err := unmarshalMsgPack(cmdData, &host); err != nil {
        return false, err
    }

    // 做更新或者新增DaprHostMember操作，存储到所有与actor_type相同的hash一致性存储上。
    return c.state.upsertMember(&host), nil
}

// removeMember 删除FSM Apply内部转发过来的处理请求
// 这个请求最初是来自actor服务发送过来的删除自身actor，这个很可能是服务销毁导致自身actor从集群中清除
func (c *FSM) removeMember(cmdData []byte) (bool, error) {
    c.stateLock.Lock()
    defer c.stateLock.Unlock()

    var host DaprHostMember
    // 对于传输过来的MessagePack消息, 反序列化为DaprHostMember,
    // 也就是业务pod支持的所有与actor相关的数据
    if err := unmarshalMsgPack(cmdData, &host); err != nil {
        return false, err
    }

    // 移除actor支持所有actor_type所在的hash一致性存储数据
    return c.state.removeMember(&host), nil
}

// Apply方法，是FSM有限状态机接口的Apply方法的实现
// 它的用途是接收raft leader发送的entry log。并存储到placement的state状态中。
func (c *FSM) Apply(log *raft.Log) interface{} {
    var (
        err     error
        updated bool
    )

    if log.Index < c.state.Index {
        logging.Warnf("old: %d, new index: %d. skip apply", c.state.Index, log.Index)
        return false
    }

    // 首先获取数据流第一个字节，这个字节表示actor想要的操作类型：删除还是Upsert操作
    switch CommandType(log.Data[0]) {
    case MemberUpsert:
        // 把业务pod支持的所有actor_type的hash一致性存储中，新增或者更新数据DaprHostMember数据
        updated, err = c.upsertMember(log.Data[1:])
    case MemberRemove:
        // 把业务所支持的所有actor_type，在所有关联的hash一致性存储中，都删除该DaprHostMember数据
        updated, err = c.removeMember(log.Data[1:])
    default:
        err = errors.New("unimplemented command")
    }

    if err != nil {
        logging.Errorf("fsm apply entry log failed. data: %s, error: %s",
            string(log.Data), err.Error())
        return false
    }

    return updated
}

// Restore 是raft FSM有限状态机接口的Restore方法实现，它用于恢复绝大部分raft节点数据
// Restore接收reader流，这个流是读取的最新快照数据流，并通过MessagePack反序列化到placement state状态中
func (c *FSM) Restore(old io.ReadCloser) error {
    defer old.Close()

    dec := codec.NewDecoder(old, &codec.MsgpackHandle{})
    var members DaprHostMemberState
    if err := dec.Decode(&members); err != nil {
        return err
    }

    c.stateLock.Lock()
    // 数据恢复到placement state内存中
    c.state = &members
    c.state.restoreHashingTables()
    c.stateLock.Unlock()

    return nil
}

// Snapshot 是raft FSM有限状态机接口的Snapshot方法实现，它用于满足快照备份的条件规则后，就会调用该方法
// 同时注意FSMSnapshot也是一个接口类, 它需要实现两个方法：Persist和Release。一般后者不需要做啥。
func (c *FSM) Snapshot() (raft.FSMSnapshot, error) {
     return &snapshot{
         state: c.state.clone(),
     }, nil
 }

// FSMSnapshot 是raft FSM有限状态机实现备份的接口
type FSMSnapshot interface {
    // Persist方法用于把raft分布式状态信息写入到磁盘文件中
    Persist(sink SnapshotSink) error

    // Release方法用于snapshot快照落入磁盘文件成功后的回调
    Release()
}

// snapshot 表示raft FSM有限状态机中进行分布式状态的快照时使用，它是在FSM的Snapshot方法内使用
type snapshot struct {
    // state表示placement state分布式状态信息，它是整个集群的所有actor_type和hash一致性存储的数据信息。
    // hash一致性存储的节点信息是Host={actor服务grpc地址Name， 业务POD的Appid，当前节点的负载情况Load, 端口Port(ps. 这里的端口在Name已经存在，属于荣冗余信息)}
    state *DaprHostMemberState
}

// Persist 该方法用于把placement state分布式状态信息全部存储到快照文件中
func (s *snapshot) Persist(sink raft.SnapshotSink) error {
    // 把placement存储分布式state状态，通过MessagePack序列化为数据流
    b, err := marshalMsgPack(s.state)
    if err != nil {
        sink.Cancel()
        return err
    }
    // 把MessagePack序列化后的数据流写入到快照文件中
    if _, err := sink.Write(b); err != nil {
        sink.Cancel()
        return err
    }

    return sink.Close()
}

// Release 无需操作
func (s *snapshot) Release() {}
```

#### placement raft server

placement raft server主要是创建raft节点服务，然后通过启动多个placement raft server构建成一个完整的raft集群，并进行自选举。主要包含的业务逻辑包括创建raft节点服务，以及raft集群中的leader接收actor通过grpc发送的stream流，然后在raft集群内同步entry log. 这个最后就是raft FSM的Apply方法接收到。那么placement raft server主要两个功能：

1. 创建placement raft server节点服务；——StartRaft方法
2. raft集群的leader接收到dapr集群actor发送的stream流，最后通过ApplyCommand同步到raft集群内部，使得raft集群存储整个集群所有业务pod提供的actor服务。——ApplyCommand方法


```golang
// 创建Raft节点服务, 传入默认配置nil。
// raft.Config表示raft集群内部节点之间的心跳、选举超时、提交超时、快照条件机等集群内部协同的相关配置
func (s *Server) StartRaft(config *raft.Config) error {
    // If we have an unclean exit then attempt to close the Raft store.
    defer func() {
        if s.raft == nil && s.raftStore != nil {
            if err := s.raftStore.Close(); err != nil {
                logging.Errorf("failed to close log storage: %v", err)
            }
        }
    }()

    // 创建raft有限状态机对象，存储分布式状态信息DaprHostMemberState, 这里面就是dapr集群内所有的业务pod的actor_type相关信息
    s.fsm = newFSM()

    // 解析raftBind DNS地址，获取IP:PORT
    addr, err := tryResolveRaftAdvertiseAddr(s.raftBind)
    if err != nil {
        return err
    }

    loggerAdapter := newLoggerAdapter()
    // Transport是raft集群内部节点之间的通信渠道, 这里trans表示raft集群内部通信的服务端口
    // 这个trans用于NewRaft raft节点时传入到集群内部的信息
    trans, err := raft.NewTCPTransportWithLogger(s.raftBind, addr, 3, 10*time.Second, l  oggerAdapter)
    if err != nil {
        return err
    }

    s.raftTransport = trans

    // inMem 为true表示分布式状态快照信息存储在内存中，那么raft节点down掉，再恢复时需要从0开始同步状态
    if s.inMem {
        raftInmem := raft.NewInmemStore()
        s.stableStore = raftInmem
        s.logStore = raftInmem
        s.snapStore = raft.NewInmemSnapshotStore()
    } else {
        // 否则，则把快照存储在磁盘文件中。
        // 首先校验存放快照的raftStorePath存储的路径是否存在，不存在则创建raft-id快照目录。
        if err = ensureDir(s.raftStorePath()); err != nil {
            return errors.Wrap(err, "failed to create log store directory")
        }

        // LogStore、StableStore分别用来存储raft log、节点状态信息
        // 目前raft log和raft节点状态信息都是存在raft-boltdb小型数据库中
        s.raftStore, err = raftboltdb.NewBoltStore(filepath.Join(s.raftStorePath(), "ra  ft.db"))
        if err != nil {
            return err
        }
        s.stableStore = s.raftStore
        s.logStore, err = raft.NewLogCache(raftLogCacheSize, s.raftStore)
        if err != nil {
            return err
        }

        // 创建raft节点上存储分布式状态信息的快照文件存储对象。用于创建快照、获取快照数据、打开快照数据文件
        s.snapStore, err = raft.NewFileSnapshotStoreWithLogger(s.raftStorePath(), snaps  hotsRetained, loggerAdapter)
        if err != nil {
            return err
        }
    }

    // 如果raft.Config值为空，则创建默认raft节点在raft集群内通信的信息。
    if config == nil {
        s.config = &raft.Config{
            ProtocolVersion:    raft.ProtocolVersionMax,
            HeartbeatTimeout:   1000 * time.Millisecond,
            ElectionTimeout:    1000 * time.Millisecond,
            CommitTimeout:      50 * time.Millisecond,
            MaxAppendEntries:   64,
            ShutdownOnRemove:   true,
            TrailingLogs:       10240,
            SnapshotInterval:   120 * time.Second,
            SnapshotThreshold:  8192,
            LeaderLeaseTimeout: 500 * time.Millisecond,
        }
    } else {
        s.config = config
    }

    // 再设置raft节点其他日志输出、本raft节点ID信息
    s.config.Logger = loggerAdapter
    s.config.LocalID = raft.ServerID(s.id)

    // 第一次启动raft节点时，则raft节点没有任何存储信息，则获取初始化该节点所需要的配置信息
    // 这里重点注意s.peers列表信息，它表示raft节点初始化创建时，会告知构建集群的整个raft列表信息。这样才能构建成一个完整的raft集群
    bootstrapConf, err := s.bootstrapConfig(s.peers)
    if err != nil {
        return err
    }

    if bootstrapConf != nil {
        // 首次启动raft节点，通过上面初始化raft节点所需要的所有信息：内部通信trans，raft log和节点信息的raft-boltdb、快照存储对象等配置信息，来初始化raft节点
        if err = raft.BootstrapCluster(
            s.config, s.logStore, s.stableStore,
            s.snapStore, trans, *bootstrapConf); err != nil {
            return err
        }
    }

    // 创建raft节点服务
    s.raft, err = raft.NewRaft(s.config, s.fsm, s.logStore, s.stableStore, s.snapStore,   s.raftTransport)
    if err != nil {
        return err
    }

    logging.Infof("Raft server is starting on %s...", s.raftBind)

    return err
}

// ApplyCommand raft集群的leader节点会接收到actor通过grpc内部通信发送过来的删除、新增和修改actor信息, 然后把actor信息同步给raft集群来管理actor分布式状态信息。
func (s *Server) ApplyCommand(cmdType CommandType, data DaprHostMember) (bool, error) {
    // 首先判断当前接受外部消息的raft节点是不是leader。
    // 因为在raft集群中，只有leader节点才能处理外部的请求
    if !s.IsLeader() {
        return false, errors.New("this is not the leader node")
    }

    // 首先从actor的操作类型和DaprMemberState转换成entry log数据流
    // 在整个placement集群内和raft集群内，都是通过MessagePack协议进行数据流转的
    cmdLog, err := makeRaftLogCommand(cmdType, data)
    if err != nil {
        return false, err
    }

    // 然后传递给各个集群下的所有raft节点，通过Apply方法推送给集群内其他follower节点；
    // 最后每个raft节点通过FSM有限状态机的Apply方法接收entry log，来存储和快照actor分布式状态信息。
    future := s.raft.Apply(cmdLog, commandTimeout)
    if err := future.Error(); err != nil {
        return false, err
    }

    // 判断raft集群所有节点是否同步分布式状态成功。
    resp := future.Response()
    return resp.(bool), nil
}
```

**小结：通过上面对整个placement raft的介绍，我们可以知道placement raft是如果接收actor信息，并通过hashicorp/consistent SDK如何同步分布式状态信息和保存快照的， 这个了解后下面的placement其他内容就很简单了**


#### placement Server

placement server集群是分散的，没有master-slave结构，是平等的。每一个placement进程会有raft节点，各个raft会构建成一个完整的raft集群。而placement server是在k8s的dapr-system命名空间下，我们可以启动多个placement server，然后构建成一个raft集群；

因为各个placement server是平等的，在每个业务pod启动时，其中注入的daprd runtime启动时，会把当前所有placement server地址获取到，然后形成一个placement服务地址列表。


然后我们再看看placement server服务的整个创建、接收dapr runtime actor流式请求，并处理和响应数据。

placement server比较关键的两个方法：
- Run方法，用于启动placment server grpc端口服务， 提供的是ReportDaprStatus方法的流式服务
- ReportDaprStatus方法，用于接收所有业务pod注入的daprd runtime，并与它们建立grpc stream连接；如果该placement server是leader，则表示它可以接收actor的注册和修改、删除，并同步到raft集群分布式状态中。以及把raft集群存储的所有分布式状态信息返回给dapr runtime存储。
- MonitorLeadership方法，用于监测本placement server是否为leader。如果是leader，就会接收ReportDaprStatus方法通过channel传递给自己的actor状态信息，并通过ApplyCommand下发给raft node leader。

```golang
// Run 方法用于创建并启动placment server grpc服务，并注册接口服务能力：ReportDaprStatus
// 端口就是placment server服务端口
func (p *Service) Run(port string, certChain *dapr_credentials.CertChain) {
    var err error
    p.serverListener, err = net.Listen("tcp", fmt.Sprintf(":%s", port))
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    opts, err := dapr_credentials.GetServerOptions(certChain)
    if err != nil {
        log.Fatalf("error creating gRPC options: %s", err)
    }
    p.grpcServer = grpc.NewServer(opts...)
    placementv1pb.RegisterPlacementServer(p.grpcServer, p)

    if err := p.grpcServer.Serve(p.serverListener); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}

// 重点看ReportDaprStatus方法
// ReportDaprStatus 方法用于接收dapr runtime actor上报Host信息
// 上报场景，业务POD启动时，注入到业务POD的dapr runtime容器会上报actor给placement server，raft集群添加actor到分布式状态信息中
// 然后是业务pod销毁时，会把actor转发给placement server，然后在raft集群中删除该信息
func (p *Service) ReportDaprStatus(stream placementv1pb.Placement_ReportDaprStatusServer) error {
  registeredMemberID := ""
  // 首先通过hasLeadership变量，校验placement server内部的raft node是否为leader。
  // 如果不是leader，则直接报错,返回codes.FailedPrecondition，那么grpc主调方接收到这个错误
  // 则dapr runtime actor会通过serverIndex加1操作，建立和其他placment server的连接。
  //
  // 所以如果该placement server不是leader，则直接返回报错。
  // 对于hasLeadership状态的改变，后面会通过membership详细介绍
  for p.hasLeadership {
      // recv方法阻塞读daprd runtime actor发送过来的Host信息
      req, err := stream.Recv()
      switch err {
      case nil:
          // 一个dapr runtime actor与placment server连接初次建立时，registeredMemberID为空

          if registeredMemberID == "" {
              // 把registeredMemberID设置为非空，这样下次再读取Host发送过来的信息时，不需要再做一次添加
              registeredMemberID = req.Name
              // 则会通过把建立好的连接存储到streamConnPool列表中
              p.addStreamConn(stream)
              // 最后再把该dapr runtime actor加入之前整个集群的分布式状态信息actor，同步给当前新来的stream。
              // 这样新添加的业务pod dapr runtime就有来整个集群的所有actor信息
              // 注意：这里是三阶段提交{lock, 分布式状态信息，unlock}
              p.performTablesUpdate([]placementGRPCStream{stream}, p.raftNode.FSM().PlacementState())
          }

          // 心跳信息，如果2s没有收到dapr runtime actor发送过来的心跳信息，则表示actor可能已经销毁了。
          p.lastHeartBeat.Store(req.Name, time.Now().UnixNano())

          //
          members := p.raftNode.FSM().State().Members
          upsertRequired := true
          ......

          // 如果actor确实是第一次或者需要修改，则把信息通过membershipCh传递给下游placement server的membership
          if upsertRequired {
              // cmdType表示操作类型；host表示存储的状态信息
              // Name：表示上游actor内部grpc服务端口信息；AppID表示业务POD注册到dapr的应用id；
              // Entities：表示业务pod支持可以处理的所有actor_type类型任务
              p.membershipCh <- hostMemberChange{
                  cmdType: raft.MemberUpsert,
                  host: raft.DaprHostMember{
                      Name:      req.Name,
                      AppID:     req.Id,
                      Entities:  req.Entities,
                      UpdatedAt: time.Now().UnixNano(),
                  },
            }
            ......
        }
    }

    return status.Error(codes.FailedPrecondition, "only leader can serve the request")
}

// MonitorLeadership 方法用于placement server监测自己是否为leader，改变自己的状态
// 并且为leader时，接收placment server接收dapr runtime actor通过grpc发送过来的Host actor信息，
// 最后作为下游接收channel下发的actor状态信息，通过ApplyCommand转发给raft node leader。
func (p *Service) MonitorLeadership() {
      // 通过raft node节点获取leader事件通知
      leaderCh := p.raftNode.Raft().LeaderCh()

      // placement server服务启动后，直接会监听自己的raft node是否为leader。这样告知上层
      // 改变本placement server服务的状态。自己可以处理daprd runtime actor发送过来的状态信息
      for {
          select {
          // 接收leaderCh发送过来的事件通过
          case isLeader := <-leaderCh:
              // 如果isLeader是true，表示本raft node选举为leader了。
              if isLeader {
                  weAreLeaderCh = make(chan struct{})
                  leaderLoop.Add(1)
                  go func(ch chan struct{}) {
                      defer leaderLoop.Done()
                      //
                      p.leaderLoop(ch)
                  }(weAreLeaderCh)
                  log.Info("cluster leadership acquired")
              } else {
                  ......
              }

          case <-p.shutdownCh:
              return
          }
      }
  }

// leaderLoop 方法用于在placement server的raft node事件通知自己已经被选举为leader后
// 做三个方面的事情：
// 1. 需要改变placement server的状态, hasLeadership为true，可以处理grpc上游dapr runtime actor的请求;
// 2. 作为leader，可以接受上游集群内所有daprd runtime actor的请求，并通过memberCh获取，删除、添加或者修改actor
// 3. 同时当修改或者删除actor信息，隔一段时间，就需要把raft集群的所有分布式状态信息，传播给上游所有注册的dapr runtime actor；这样整个集群注册业务POD的actor，就可以保证拥有分布式状态信息，路由就这样实现了。
// 4. 如果上游存在dapr runtime actor与该placement server长时间每隔1s没有数据流，则placment server认为该服务不在线，需要剔除，
func (p *Service) leaderLoop(stopCh chan struct{}) {
    for !p.hasLeadership {
        ......
        if !p.hasLeadership {
            // 把当前placement server服务状态设置为leader
            p.establishLeadership()
            log.Info("leader is established.")
            // revoke leadership process must be done before leaderLoop() ends.
            defer p.revokeLeadership()
        }
    }

    // 这里执行2、3和4的业务逻辑
    p.membershipChangeWorker(stopCh)
}

// membershipChangeWorker 方法用于处理新增更新或者删除的上游daprd runtime actor请求的数据流；
// 以及只要更新、删除或者新增actor Host操作，就会在2秒后执行传播到整个集群到dapr runtime actor存储整个分布式状态嘻嘻
// 以及1秒接收不到actor的上报保活信息，就要立即执行剔除actor操作，认为能够处理该actor_type选择的该业务pod可能挂掉了。
func (p *Service) membershipChangeWorker(stopCh chan struct{}) {
    // actor上报最长间隔时间500ms, 如果500ms该actor没有上报Host给placement server，则剔除
    faultyHostDetectTimer := time.NewTicker(faultyHostDetectInterval)
    // 在整个集群有dapr runtime actor上报信息的情况，500ms内就会传播同步给整个集群的所有dapr runtime actor信息
    disseminateTimer := time.NewTicker(disseminateTimerInterval)

    // 初始化时，memberUpdateCount默认为0，表示当前还没有任何actor更新、删除或者新增请求
    // 只有当memberUpdateCount不为0时, 才会发送placement server leader通过三阶段提交分布式状态信息给上游所有的dapr runtime actor。使其拥有全局的分布式状态信息。
    p.memberUpdateCount.Store(0)

    //
    go p.processRaftStateCommand(stopCh)

    for {
        select {
        // 如果placement server接收到上游dapr runtime actor的请求后，且需要发生变更，则到了传播时间后
        // 需要同步给全局所有actor
        case t := <-disseminateTimer.C:
          // Earlier stop when leadership is lost.
          if !p.hasLeadership {
              continue
          }

          // 这里检查是否到了传播时间，且当前所有要处理的变更actor信息都处理完成。
          // 且memberUpdateCount不为0, 表示当前一段时间内有actor发生变更，则需要进行全局传播。
          if p.disseminateNextTime <= t.UnixNano() && len(p.membershipCh) == 0 {
              if cnt := p.memberUpdateCount.Load(); cnt > 0 {
                  log.Debugf("Add raft.TableDisseminate to membershipCh. memberUpdate  Count count: %d", cnt)
                  // 最后是通过三阶段提交进行一致性提交，保证信息的一致性
                  p.membershipCh <- hostMemberChange{cmdType: raft.TableDisseminate}
              }
          }

        // 到了检测全局actor是否有死掉的actor，然后进行raft集群剔除该Host信息
      case t := <-faultyHostDetectTimer.C:
          // 如果membershipCh channel长度为0，表示当前没有需要变更的actor了。
          // 说明可以进行actor心跳检测，并剔除超时actor
          if len(p.membershipCh) == 0 {
              // 获取raft集群存储的分布式状态信息，遍历所有的Host。
              m := p.raftNode.FSM().State().Members
              for _, v := range m {
                heartbeat, _ := p.lastHeartBeat.LoadOrStore(v.Name, time.Now().Unix  Nano())

                    elapsed := t.UnixNano() - heartbeat.(int64)
                    if elapsed < int64(p.faultyHostDetectDuration) {
                        continue
                    }
                    log.Debugf("Try to remove outdated host: %s, elapsed: %d ns", v.Nam  e, elapsed)
                    // 剔除actor
                    p.membershipCh <- hostMemberChange{
                        cmdType: raft.MemberRemove,
                        host:    raft.DaprHostMember{Name: v.Name},
                    }
                }
            }
        }
    }
}

// processRaftStateCommand 方法用于处理两个点：
// 1. placement server通过ReportDaprStatus方法接收daprd runtime actor通过grpc下发的actor host信息，并通过membershipCh下发给下游的消费者处理acotr的变更操作，并通过raft node leader同步到raft集群中；
// 2. 通过接收actor变更带来的整体传播事件，把分布式状态信息通过三阶段提交推送给整个集群所有dapr runtime actor
func (p *Service) processRaftStateCommand(stopCh chan struct{}) {
      // 限制goroutine的并发量10个。同时处理超过10个actor变更，则等待阻塞处理
      logApplyConcurrency := make(chan struct{}, raftApplyCommandMaxConcurrency)

      for {
          select {
          case op := <-p.membershipCh:
            switch op.cmdType {
            // 如果actor是变更操作：删除、新增或者修改
            // 则通过ApplyCommand方法最终提交给raft集群。
            case raft.MemberUpsert, raft.MemberRemove:
                // MemberUpsert updates the state of dapr runtime host whenever
                // Dapr runtime sends heartbeats every 1 second.
                // MemberRemove will be queued by faultHostDetectTimer.
                // Even if ApplyCommand is failed, both commands will retry
                // until the state is consistent.
                logApplyConcurrency <- struct{}{}
                go func() {
                    updated, raftErr := p.raftNode.ApplyCommand(op.cmdType, op.host)
                    if raftErr != nil {
                        log.Errorf("fail to apply command: %v", raftErr)
                    } else {
                        // 如果是删除actor操作，则不需要在发生传播和心跳动作
                        if op.cmdType == raft.MemberRemove {
                            p.lastHeartBeat.Delete(op.host.Name)
                        }

                        // ApplyCommand returns true only if the command changes hashin  g table.
                        if updated {
                            // 如果更新成功，则表示某一段时间内有actor变更，加1操作，且下次全局全波时间是2s后
                            p.memberUpdateCount.Inc()
                            p.disseminateNextTime = time.Now().Add(disseminateTimeout).  UnixNano()
                        }
                    }
                    <-logApplyConcurrency
                }()

            case raft.TableDisseminate:
                // 这个表示是全局传播分布式状态信息给所有的daprd runtime actor，首先获取集群所有的actor stream流
                nStreamConnPool := len(p.streamConnPool)
                // 然后获取所有actor的数量
                nTargetConns := len(p.raftNode.FSM().State().Members)
                // 如果memberUpdateCount不为0，表示这段时间内存在actor变更，所以需要进行三阶段提交给所有的actor
                if cnt := p.memberUpdateCount.Load(); cnt > 0 {
                    state := p.raftNode.FSM().PlacementState()
                    p.performTablesUpdate(p.streamConnPool, state)
                    //然后清空最近一段时间发生变更的所有actor数量，重新在一段时间内发生actor变更统计数据。
                    p.memberUpdateCount.Store(0)
                     p.faultyHostDetectDuration = faultyHostDetectDefaultDuration
                 }
             }
         }
     }
 }

// performTablesUpdate 方法执行三阶段提交，lock、update和unlock。
// 这个是用来传播分布式状态信息给上游所有的actors，并保证actor存储全局分布式状态数据的一致性
 func (p *Service) performTablesUpdate(hosts []placementGRPCStream, newTable *v1pb.Place  mentTables) {
      p.disseminateLock.Lock()
      defer p.disseminateLock.Unlock()

      // TODO: error from disseminationOperation needs to be handle properly.
      // Otherwise, each Dapr runtime will have inconsistent hashing table.
      p.disseminateOperation(hosts, "lock", nil)
      p.disseminateOperation(hosts, "update", newTable)
      p.disseminateOperation(hosts, "unlock", nil)
  }

// disseminateOperation 方法是传播操作，遍历所有的actor，通过grpc client stream流Send发送信息给各个actor。
  func (p *Service) disseminateOperation(targets []placementGRPCStream, operation string,   tables *v1pb.PlacementTables) error {
      o := &v1pb.PlacementOrder{
          Operation: operation,
          Tables:    tables,
      }

      var err error
      for _, s := range targets {
          err = s.Send(o)
      }
}
```

通过对placement server的源码分析，我们可以了解到placement server集群内部的leader是如何监听和上报的，以及整个集群的actor通过多个stream流与placement server的ReportDaprtatus方法建立stream流列表，然后在管理和传播分布式状态信息actors，到raft集群内部，以及通过placement server leader通过三阶段提交传播给上游所有的daprd runtime actor存储分布式状态信息。


## placement与actor和raft集群的通信过程


placement server服务主要是接收外面dapr runtime actor的grpc数据流请求ReportDaprStatus方法，因为raft集群对节点数量有容忍度要求，placement集群的数量是有一些要求的。然后每个业务pod中注入的daprd runtime容器中actor因为有全部的placement servers地址列表，那么这个actor如何知道placement server列表地址中哪个placement server的raft节点是leader呢？通过下面两个方法可以看到其实实现机制：

```golang
pkg/actors/internal/placement.go
// establishStreamConn 从placement server列表中当前索引上建立一个placement server的stream
// 这个stream至于连接的是不是raft node leader，是通过上层方法校验的。
func (p *ActorPlacement) establishStreamConn() ... {
  for !p.shutdown {
      // 从placement servers列表中从索引为0的位置开始寻找raft node为leader的placement server服务。
      serverAddr := p.serverAddr[p.serverIndex]
      // placement server会提供一个健康检查接口，服务路由: /healthz
      // 如果健康检查失败，则会反复重试直到placement server起来，这个完全依赖于deployment机制
      if !p.appHealthFn() {
        time.Sleep(placementReconnectInterval)
        continue
      }
      // 业务pod注入的daprd runtime容器中actor会创建与placement server服务建立连接的grpc client数据流
      conn, err := grpc.Dial(serverAddr, opts...)
    NEXT_SERVER:
      if err != nil {
          log.Debugf("error connecting to placement service: %v", err)
          if conn != nil {
              conn.Close()
          }
          p.serverIndex = (p.serverIndex + 1) % len(p.serverAddr)
          time.Sleep(placementReconnectInterval)
          continue
      }

        // grpc client与placement server建立一个ReportDaprStatus方法的数据流stream
        client := v1pb.NewPlacementClient(conn)
        stream, err := client.ReportDaprStatus(context.Background())
        if err != nil {
            goto NEXT_SERVER
        }
}

// Start 方法用于找到一个raft node是leader的placement server，并建立与placement server的grpc client stream
func (p *ActorPlacement) Start() {
  p.serverIndex = 0
  // 初始化时streamConnAlive为true，默认为建立连接的placement server就是raft node leader。
  p.streamConnAlive = true
  p.clientStream, p.clientConn = p.establishStreamConn()
  // 首先业务pod注入的daprd runtime容器启动时，会通过ActorPlacement实例的Start方法
  // 把业务pod支持的所有actor_type和runtime服务端口地址信息actor，
  // 通过与placement server集群建立grpc client stream数据流在ReportDaprStatus方法上建立连接
  //
  // 下面有两个协程：
  // 一个是grpc client stream用于接收placement server集群存储的所有actor_type和actor分布式状态信息
  // 另一个是daprd runtime actor在启动时会把自己的信息通过grpc client stream添加到placement server集群中
  // 也就是最后把Host信息，添加到raft集群中；以及从raft集群读取所有业务pod支持的actor_type信息。
  go func() {
    for !p.shutdown {
        // 接收placement server集群下raft集群上报的所有分布式状态state信息。
        resp, err := p.clientStream.Recv()
        // 如果stream流读取响应信息，发现是err为codes.FailedPrecondition错误
        // 和serverIndex建立连接的placement server的raft node并不是leader。
        // 所以逐步加1操作，与其他的placement server节点服务建立stream连接，然后再通过另一个协程发送要添加的actor_type列表和actor信息，给placement server建立的raft集群。
        if !p.streamConnAlive || err != nil {
          // 如果streamConnAlive
          p.streamConnAlive = false
          s, ok := status.FromError(err)
          if ok && s.Code() == codes.FailedPrecondition {
              p.serverIndex = (p.serverIndex + 1) % len(p.serverAddr)
          } else {
              log.Debugf("disconnected from placement: %v", err)
          }

          // 通过serverIndex加1操作，建立与另一个placement server的grpc client stream数据流
          newStream, newConn := p.establishStreamConn()
          if newStream != nil {
              p.clientConn = newConn
              p.clientStream = newStream
              // 并把streamConnAlive设置为true，默认表示这个建立的placement server的raft node是leader。
              p.streamConnAlive = true
              close(p.streamConnectedCh)
              p.streamConnectedCh = make(chan struct{})
          }
        }
        // 把placement server集群raft集群存储的所有分布式状态信息Host，给daprd runtime actor存储
        // 这样每个业务pod的daprd runtime都有了整个集群所有actor信息
        // 这样任何请求访问任何一个actor，都能找到目标actor地址。
        p.onPlacementOrder(resp)
    }
  }()

  // 发送本daprd runtime actor Host信息给raft集群，也就是placement server集群。
  go func() {
    for !p.shutdown {
      // 初始化默认为streamConnAlive为true
      if !p.streamConnAlive {
          <-p.streamConnectedCh
      }
      // Host代表raft集群中存储的分布式状态信息。只是状态信息存储是通过每个actor_type映射的hash一致性数据存储。
      host := v1pb.Host{
          Name:     p.runtimeHostName,
          Entities: p.actorTypes,
          Id:       p.appID,
          Load:     1, // Not used yet
          // Port is redundant because Name should include port number
      }

      // 通过前面与placement server建立的grpc client stream流，发送本地daprd runtime actor信息给raft leader
      // 并进行存储
      // 如果上面读取stream流的协程，读取数据失败，且为非raft node leader错误，表示建立流式连接的placement server并不是leader。
      // 这样上面Recv()方法时，err就是FailedPrecondition错误，这样就会serverIndex加1操作，且streamConnAlive为false操作，找到建立与+1操作的placement server建立stream连接，以及本协程读取阻塞在
      err := p.clientStream.Send(&host)
      if err != nil {
          diag.DefaultMonitoring.ActorStatusReportFailed("send", "status")
          log.Debugf("failed to report status to placement service : %v", err)
      }
      // 如果建立连接的placement server不是leader，则默认streamConnAlive为true，则会等待一段时间
      // 之后另一个协程会读取报错后，会把streamConnAlive设置为false，这样等待p.streamConnectedCh关闭后在打开，
      // 这样，这个协程又会执行一次把Host发送给已建立的placement server连接。然后对端判断自己是不是leader后；
      // 如果对端是leader，则会把raft集群存储的所有分布式状态返回给本daprd runtime actor存储。
      // 否则，两个goroutine重复+1操作、与placement server建立grpc client stream连接、发送Host添加到raft集群中
      // placement server会通过建立好的stream流，把整个分布式集群的分布式状态信息作为响应信息，返回给本服务。
      if p.streamConnAlive {
        // 每睡眠1秒后，发送一次actor数据给placement server，这样保活。不然会被placment server leader剔除
        time.Sleep(statusReportHeartbeatInterval)
      }
    }
  }()
}
```

通过上面对daprd runtime actor的分析，就知道了整体actor与placement server集群的通信过程，包括：寻找placement server的leader、添加或者删除Host信息同步给placment server集群、获取placement server集群存储分布式状态所有信息。placment server底层就是raft节点服务。


## 总结

上面就是针对pkg/placement包源码整体的阅读，不过这里没有阅读hashing/consistent包的代码。有兴趣或者有疑问的童鞋可以和我交流和沟通。 我们可以看到通过placement服务对raft集群的时间，以及hash一致性表来解决相同actor_type的各个业务pod的负载均衡问题， 并最后通过三阶段提交传播actor分布式状态信息给整个集群的actor，使得业务寻找在整个集群寻找和访问actor，变得非常容易。
通过本文，你可以了解和实现自己的raft集群分布式状态存储应用；

