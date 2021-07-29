1 架构


从etcd的架构图中我们可以看到，etcd主要分为四个部分。

HTTP Server： 用于处理用户发送的API请求以及其它etcd节点的同步与心跳信息请求。
Store：用于处理etcd支持的各类功能的事务，包括数据索引、节点状态变更、监控与反馈、事件处理与执行等等，是etcd对用户提供的大多数API功能的具体实现。
Raft：Raft强一致性算法的具体实现，是etcd的核心。
WAL：Write Ahead Log（预写式日志），是etcd的数据存储方式。除了在内存中存有所有数据的状态以及节点的索引以外，etcd就通过WAL进行持久化存储。WAL中，所有的数据提交前都会事先记录日志。Snapshot是为了防止数据过多而进行的状态快照；Entry表示存储的具体日志内容。
 

通常，一个用户的请求发送过来，会经由HTTP Server转发给Store进行具体的事务处理

如果涉及到节点的修改，则交给Raft模块进行状态的变更、日志的记录，然后再同步给别的etcd节点以确认数据提交

最后进行数据的提交，再次同步。

 

 

 

2 新版etcd重要变更列表
获得了IANA认证的端口，2379用于客户端通信，2380用于节点通信，与原先的（4001 peers / 7001 clients）共用。
每个节点可监听多个广播地址。监听的地址由原来的一个扩展到多个，用户可以根据需求实现更加复杂的集群环境，如一个是公网IP，一个是虚拟机（容器）之类的私有IP。
etcd可以代理访问leader节点的请求，所以如果你可以访问任何一个etcd节点，那么你就可以无视网络的拓扑结构对整个集群进行读写操作。
etcd集群和集群中的节点都有了自己独特的ID。这样就防止出现配置混淆，不是本集群的其他etcd节点发来的请求将被屏蔽。
etcd集群启动时的配置信息目前变为完全固定，这样有助于用户正确配置和启动。
运行时节点变化(Runtime Reconfiguration)。用户不需要重启 etcd 服务即可实现对 etcd 集群结构进行变更。启动后可以动态变更集群配置。
重新设计和实现了Raft算法，使得运行速度更快，更容易理解，包含更多测试代码。
Raft日志现在是严格的只能向后追加、预写式日志系统，并且在每条记录中都加入了CRC校验码。
启动时使用的_etcd/* 关键字不再暴露给用户
废弃集群自动调整功能的standby模式，这个功能使得用户维护集群更困难。
新增Proxy模式，不加入到etcd一致性集群中，纯粹进行代理转发。
ETCD_NAME（-name）参数目前是可选的，不再用于唯一标识一个节点。
摒弃通过配置文件配置 etcd 属性的方式，你可以用环境变量的方式代替。
通过自发现方式启动集群必须要提供集群大小，这样有助于用户确定集群实际启动的节点数量。
 

 

3 etcd概念词汇表
Raft：etcd所采用的保证分布式系统强一致性的算法。
Node：一个Raft状态机实例。
Member： 一个etcd实例。它管理着一个Node，并且可以为客户端请求提供服务。
Cluster：由多个Member构成可以协同工作的etcd集群。
Peer：对同一个etcd集群中另外一个Member的称呼。
Client： 向etcd集群发送HTTP请求的客户端。
WAL：预写式日志，etcd用于持久化存储的日志格式。
snapshot：etcd防止WAL文件过多而设置的快照，存储etcd数据状态。
Proxy：etcd的一种模式，为etcd集群提供反向代理服务。
Leader：Raft算法中通过竞选而产生的处理所有数据提交的节点。
Follower：竞选失败的节点作为Raft中的从属节点，为算法提供强一致性保证。
Candidate：当Follower超过一定时间接收不到Leader的心跳时转变为Candidate开始竞选。
Term：某个节点成为Leader到下一次竞选时间，称为一个Term。
Index：数据项编号。Raft中通过Term和Index来定位数据。
 

 

 

4 集群化应用实践
etcd作为一个高可用键值存储系统，天生就是为集群化而设计的。

由于Raft算法在做决策时需要多数节点的投票，所以etcd一般部署集群推荐奇数个节点，推荐的数量为3、5或者7个节点构成一个集群。

 

 

4.1 集群启动
etcd有三种集群化启动的配置方案，分别为静态配置启动、etcd自身服务发现、通过DNS进行服务发现。

通过配置内容的不同，你可以对不同的方式进行选择。值得一提的是，这也是新版etcd区别于旧版的一大特性，它摒弃了使用配置文件进行参数配置的做法，转而使用命令行参数或者环境变量的做法来配置参数。

4.1.1. 静态配置
这种方式比较适用于离线环境，在启动整个集群之前，你就已经预先清楚所要配置的集群大小，以及集群上各节点的地址和端口信息。那么启动时，你就可以通过配置initial-cluster参数进行etcd集群的启动。

在每个etcd机器启动时，配置环境变量或者添加启动参数的方式如下。

1
2
ETCD_INITIAL_CLUSTER="infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380"
ETCD_INITIAL_CLUSTER_STATE=new
参数方法：

1
2
3
-initial-cluster
infra0=http://10.0.1.10:2380,http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
 -initial-cluster-state new
值得注意的是，

　　-initial-cluster参数中配置的url地址必须与各个节点启动时设置的initial-advertise-peer-urls参数相同。

　　（initial-advertise-peer-urls参数表示节点监听其他节点同步信号的地址）

　　如果你所在的网络环境配置了多个etcd集群，为了避免意外发生，最好使用-initial-cluster-token参数为每个集群单独配置一个token认证。这样就可以确保每个集群和集群的成员都拥有独特的ID。

综上所述，如果你要配置包含3个etcd节点的集群，那么你在三个机器上的启动命令分别如下所示。

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
$ etcd -name infra0 -initial-advertise-peer-urls http://10.0.1.10:2380 \
  -listen-peer-urls http://10.0.1.10:2380 \
  -initial-cluster-token etcd-cluster-1 \
  -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  -initial-cluster-state new
 
$ etcd -name infra1 -initial-advertise-peer-urls http://10.0.1.11:2380 \
  -listen-peer-urls http://10.0.1.11:2380 \
  -initial-cluster-token etcd-cluster-1 \
  -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  -initial-cluster-state new
 
$ etcd -name infra2 -initial-advertise-peer-urls http://10.0.1.12:2380 \
  -listen-peer-urls http://10.0.1.12:2380 \
  -initial-cluster-token etcd-cluster-1 \
  -initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  -initial-cluster-state new
在初始化完成后，etcd还提供动态增、删、改etcd集群节点的功能，这个需要用到etcdctl命令进行操作。

4.1.2. etcd自发现模式
通过自发现的方式启动etcd集群需要事先准备一个etcd集群。如果你已经有一个etcd集群，首先你可以执行如下命令设定集群的大小，假设为3.

1
$ curl -X PUT http://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83/_config/size -d value=3
然后你要把这个url地址http://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83作为-discovery参数来启动etcd。

节点会自动使用http://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83目录进行etcd的注册和发现服务。

所以最终你在某个机器上启动etcd的命令如下。

1
2
3
$ etcd -name infra0 -initial-advertise-peer-urls http://10.0.1.10:2380 \
  -listen-peer-urls http://10.0.1.10:2380 \
  -discovery http://myetcd.local/v2/keys/discovery/6c007a14875d53d9bf0ef5a6fc0257c817f0fb83
如果你本地没有可用的etcd集群，etcd官网提供了一个可以公网访问的etcd存储地址。你可以通过如下命令得到etcd服务的目录，并把它作为-discovery参数使用。

1
2
$ curl http://discovery.etcd.io/new?size=3
http://discovery.etcd.io/3e86b59982e49066c5d813af1c2e2579cbf573de
同样的，当你完成了集群的初始化后，这些信息就失去了作用。

当你需要增加节点时，需要使用etcdctl来进行操作。

为了安全，请务必每次启动新etcd集群时，都使用新的discovery token进行注册。

另外，如果你初始化时启动的节点超过了指定的数量，多余的节点会自动转化为Proxy模式的etcd。

 

4.1.3. DNS自发现模式
etcd还支持使用DNS SRV记录进行启动。关于DNS SRV记录如何进行服务发现，可以参阅RFC2782，所以，你要在DNS服务器上进行相应的配置。

(1) 开启DNS服务器上SRV记录查询，并添加相应的域名记录，使得查询到的结果类似如下。

1
2
3
4
$dig +noall +answer SRV _etcd-server._tcp.example.com
_etcd-server._tcp.example.com. 300 IN   SRV 0 0 2380 infra0.example.com.
_etcd-server._tcp.example.com. 300 IN   SRV 0 0 2380 infra1.example.com.
_etcd-server._tcp.example.com. 300 IN   SRV 0 0 2380 infra2.example.com.
(2) 分别为各个域名配置相关的A记录指向etcd核心节点对应的机器IP。使得查询结果类似如下。

1
2
3
4
$dig +noall +answer infra0.example.com infra1.example.com infra2.example.com
infra0.example.com. 300 IN  A   10.0.1.10
infra1.example.com. 300 IN  A   10.0.1.11
infra2.example.com. 300 IN  A   10.0.1.12
(3) 做好了上述两步DNS的配置，就可以使用DNS启动etcd集群了。配置DNS解析的url参数为-discovery-srv，其中某一个节点地启动命令如下。

1
2
3
4
5
6
7
8
$ etcd -name infra0 \
-discovery-srv example.com \
-initial-advertise-peer-urls http://infra0.example.com:2380 \
-initial-cluster-token etcd-cluster-1 \
-initial-cluster-state new \
-advertise-client-urls http://infra0.example.com:2379 \
-listen-client-urls http://infra0.example.com:2379 \
-listen-peer-urls http://infra0.example.com:2380
　　当然，你也可以直接把节点的域名改成IP来启动。

 

 

 

 

4.2 关键部分源码解析
etcd的启动是从主目录下的main.go开始的，然后进入etcdmain/etcd.go，载入配置参数。

　　如果被配置为Proxy模式，则进入startProxy函数，否则进入startEtcd，开启etcd服务模块和http请求处理模块。

在启动http监听时，为了保持与集群其他etcd机器（peers）保持连接，都采用的transport.NewTimeoutListener启动方式，这样在超过指定时间没有获得响应时就会出现超时错误。

　　而在监听client请求时，采用的是transport.NewKeepAliveListener，有助于连接的稳定。

在etcdmain/etcd.go中的setupCluster函数可以看到，根据不同etcd的参数，启动集群的方法略有不同，但是最终需要的就是一个IP与端口构成的字符串。

 

在静态配置的启动方式中，集群的所有信息都已经在给出，所以直接解析用逗号隔开的集群url信息就好了。

 

DNS发现的方式类似，会预先发送一个tcp的SRV请求，先查看etcd-server-ssl._tcp.example.com下是否有集群的域名信息，如果没有找到，则去查看etcd-server._tcp.example.com。

　　根据找到的域名，解析出对应的IP和端口，即集群的url信息。

 

较为复杂是etcd式的自发现启动。

　　首先就用自身单个的url构成一个集群，然后在启动的过程中根据参数进入discovery/discovery.go源码的JoinCluster函数。

　　　　因为我们事先是知道启动时使用的etcd的token地址的，里面包含了集群大小(size)信息。

　　　　在这个过程其实是个不断监测与等待的过程。

　　启动的第一步就是在这个etcd的token目录下注册自身的信息，然后再监测token目录下所有节点的数量，如果数量没有达标，则循环等待。

　　当数量达到要求时，才结束，进入正常的启动过程。

 

配置etcd过程中通常要用到两种url地址容易混淆

　　一种用于etcd集群同步信息并保持连接，通常称为peer-urls；

　　另外一种用于接收用户端发来的HTTP请求，通常称为client-urls。

peer-urls：通常监听的端口为2380（老版本使用的端口为7001），包括所有已经在集群中正常工作的所有节点的地址。
client-urls：通常监听的端口为2379（老版本使用的端口为4001），为适应复杂的网络环境，新版etcd监听客户端请求的url从原来的1个变为现在可配置的多个。这样etcd可以配合多块网卡同时监听不同网络下的请求。
 

 

 

4.3 运行时节点变更
etcd集群启动完毕后，可以在运行的过程中对集群进行重构，包括核心节点的增加、删除、迁移、替换等。

运行时重构使得etcd集群无须重启即可改变集群的配置，这也是新版etcd区别于旧版包含的新特性。

只有当集群中多数节点正常的情况下，你才可以进行运行时的配置管理。

　　因为配置更改的信息也会被etcd当成一个信息存储和同步，如果集群多数节点损坏，集群就失去了写入数据的能力。

　　所以在配置etcd集群数量时，强烈推荐至少配置3个核心节点。

4.3.1. 节点迁移、替换
当你节点所在的机器出现硬件故障，或者节点出现如数据目录损坏等问题，导致节点永久性的不可恢复时，就需要对节点进行迁移或者替换。

当一个节点失效以后，必须尽快修复，因为etcd集群正常运行的必要条件是集群中多数节点都正常工作。

迁移一个节点需要进行四步操作：

暂停正在运行着的节点程序进程
把数据目录从现有机器拷贝到新机器
使用api更新etcd中对应节点指向机器的url记录更新为新机器的ip
使用同样的配置项和数据目录，在新的机器上启动etcd。
4.3.2. 节点增加
增加节点可以让etcd的高可用性更强。

　　举例来说，如果你有3个节点，那么最多允许1个节点失效；当你有5个节点时，就可以允许有2个节点失效。

同时，增加节点还可以让etcd集群具有更好的读性能。

　　因为etcd的节点都是实时同步的，每个节点上都存储了所有的信息，所以增加节点可以从整体上提升读的吞吐量。

 

增加一个节点需要进行两步操作：

在集群中添加这个节点的url记录，同时获得集群的信息。
使用获得的集群信息启动新etcd节点。
4.3.3. 节点移除
有时你不得不在提高etcd的写性能和增加集群高可用性上进行权衡。

　　Leader节点在提交一个写记录时，会把这个消息同步到每个节点上，当得到多数节点的同意反馈后，才会真正写入数据。

　　所以节点越多，写入性能越差。

　　在节点过多时，你可能需要移除一个或多个。

 

移除节点非常简单，只需要一步操作，就是把集群中这个节点的记录删除。然后对应机器上的该节点就会自动停止。

4.3.4. 强制性重启集群
当集群超过半数的节点都失效时，就需要通过手动的方式，强制性让某个节点以自己为Leader，利用原有数据启动一个新集群。

此时你需要进行两步操作。

备份原有数据到新机器。
使用-force-new-cluster加备份的数据重新启动节点
注意：强制性重启是一个迫不得已的选择，它会破坏一致性协议保证的安全性（如果操作时集群中尚有其它节点在正常工作，就会出错），所以在操作前请务必要保存好数据。

 

 

 

5 Proxy模式
Proxy模式也是新版etcd的一个重要变更，etcd作为一个反向代理把客户的请求转发给可用的etcd集群。

　　这样，你就可以在每一台机器都部署一个Proxy模式的etcd作为本地服务，如果这些etcd Proxy都能正常运行，那么你的服务发现必然是稳定可靠的。



所以Proxy并不是直接加入到符合强一致性的etcd集群中，也同样的，Proxy并没有增加集群的可靠性，当然也没有降低集群的写入性能。

 

5.1 Proxy取代Standby模式的原因
那么，为什么要有Proxy模式而不是直接增加etcd核心节点呢？

　　实际上etcd每增加一个核心节点（peer），都会增加Leader节点一定程度的包括网络、CPU和磁盘的负担，因为每次信息的变化都需要进行同步备份。

　　增加etcd的核心节点可以让整个集群具有更高的可靠性，但是当数量达到一定程度以后，增加可靠性带来的好处就变得不那么明显，反倒是降低了集群写入同步的性能。

　　因此，增加一个轻量级的Proxy模式etcd节点是对直接增加etcd核心节点的一个有效代替。

熟悉0.4.6这个旧版本etcd的用户会发现，Proxy模式实际上是取代了原先的Standby模式。

　　Standby模式除了转发代理的功能以外，还会在核心节点因为故障导致数量不足的时候，从Standby模式转为正常节点模式。

　　而当那个故障的节点恢复时，发现etcd的核心节点数量已经达到的预先设置的值，就会转为Standby模式。

但是新版etcd中，只会在最初启动etcd集群时，发现核心节点的数量已经满足要求时，自动启用Proxy模式，反之则并未实现。主要原因如下。

etcd是用来保证高可用的组件，因此它所需要的系统资源（包括内存、硬盘和CPU等）都应该得到充分保障以保证高可用。任由集群的自动变换随意地改变核心节点，无法让机器保证性能。所以etcd官方鼓励大家在大型集群中为运行etcd准备专有机器集群。
因为etcd集群是支持高可用的，部分机器故障并不会导致功能失效。所以机器发生故障时，管理员有充分的时间对机器进行检查和修复。
自动转换使得etcd集群变得复杂，尤其是如今etcd支持多种网络环境的监听和交互。在不同网络间进行转换，更容易发生错误，导致集群不稳定。
基于上述原因，目前Proxy模式有转发代理功能，而不会进行角色转换。

 

5.2 关键部分源码解析
从代码中可以看到，Proxy模式的本质就是起一个HTTP代理服务器，把客户发到这个服务器的请求转发给别的etcd节点。

etcd目前支持读写皆可和只读两种模式。

　　默认情况下是读写皆可，就是把读、写两种请求都进行转发。

　　只读模式只转发读的请求，对所有其他请求返回501错误。

值得注意的是，除了启动过程中因为设置了proxy参数会作为Proxy模式启动。

　　在etcd集群化启动时，节点注册自身的时候监测到集群的实际节点数量已经符合要求，那么就会退化为Proxy模式。

 

 

6 数据存储
etcd的存储分为内存存储和持久化（硬盘）存储两部分

　　内存中的存储除了顺序化的记录下所有用户对节点数据变更的记录外，还会对用户数据进行索引、建堆等方便查询的操作。

　　持久化则使用预写式日志（WAL：Write Ahead Log）进行记录存储。

在WAL的体系中，所有的数据在提交之前都会进行日志记录。

　　在etcd的持久化存储目录中，有两个子目录。

　　　　一个是WAL，存储着所有事务的变化记录；

　　　　另一个则是snapshot，用于存储某一个时刻etcd所有目录的数据。

　　　　通过WAL和snapshot相结合的方式，etcd可以有效的进行数据存储和节点故障恢复等操作。

既然有了WAL实时存储了所有的变更，为什么还需要snapshot呢？

　　随着使用量的增加，WAL存储的数据会暴增，为了防止磁盘很快就爆满，etcd默认每10000条记录做一次snapshot，经过snapshot以后的WAL文件就可以删除。

　　而通过API可以查询的历史etcd操作默认为1000条。

首次启动时，etcd会把启动的配置信息存储到data-dir参数指定的数据目录中。

　　配置信息包括本地节点的ID、集群ID和初始时集群信息。

用户需要避免etcd从一个过期的数据目录中重新启动，因为使用过期的数据目录启动的节点会与集群中的其他节点产生不一致（如：之前已经记录并同意Leader节点存储某个信息，重启后又向Leader节点申请这个信息）。

　　所以，为了最大化集群的安全性，一旦有任何数据损坏或丢失的可能性，你就应该把这个节点从集群中移除，然后加入一个不带数据目录的新节点。

6.1 预写式日志（WAL）
WAL（Write Ahead Log）最大的作用是记录了整个数据变化的全部历程。

　　在etcd中，所有数据的修改在提交前，都要先写入到WAL中。

　　使用WAL进行数据的存储使得etcd拥有两个重要功能。

故障快速恢复： 当你的数据遭到破坏时，就可以通过执行所有WAL中记录的修改操作，快速从最原始的数据恢复到数据损坏前的状态。
数据回滚（undo）/重做（redo）：因为所有的修改操作都被记录在WAL中，需要回滚或重做，只需要方向或正向执行日志中的操作即可。
WAL与snapshot在etcd中的命名规则
在etcd的数据目录中，WAL文件以$seq-$index.wal的格式存储。

　　最初始的WAL文件是0000000000000000-0000000000000000.wal，表示是所有WAL文件中的第0个，初始的Raft状态编号为0。

　　运行一段时间后可能需要进行日志切分，把新的条目放到一个新的WAL文件中。

　　假设，当集群运行到Raft状态为20时，需要进行WAL文件的切分时，下一份WAL文件就会变为0000000000000001-0000000000000021.wal。

　　　　如果在10次操作后又进行了一次日志切分，那么后一次的WAL文件名会变为0000000000000002-0000000000000031.wal。

　　　　可以看到-符号前面的数字是每次切分后自增1，而-符号后面的数字则是根据实际存储的Raft起始状态来定。

snapshot的存储命名则比较容易理解，以$term-$index.wal格式进行命名存储。

　　term和index就表示存储snapshot时数据所在的raft节点状态，当前的任期编号以及数据项位置信息。

6.2 关键部分源码解析
从代码逻辑中可以看到，WAL有两种模式，读模式（read）和数据添加（append）模式，两种模式不能同时成立。

　　一个新创建的WAL文件处于append模式，并且不会进入到read模式。

　　一个本来存在的WAL文件被打开的时候必然是read模式，并且只有在所有记录都被读完的时候，才能进入append模式，进入append模式后也不会再进入read模式。这样做有助于保证数据的完整与准确。

集群在进入到etcdserver/server.go的NewServer函数准备启动一个etcd节点时，会检测是否存在以前的遗留WAL数据。

　　检测的第一步是查看snapshot文件夹下是否有符合规范的文件，若检测到snapshot格式是v0.4的，则调用函数升级到v0.5。

　　从snapshot中获得集群的配置信息，包括token、其他节点的信息等等

　　然后载入WAL目录的内容，从小到大进行排序。根据snapshot中得到的term和index，找到WAL紧接着snapshot下一条的记录，然后向后更新，直到所有WAL包的entry都已经遍历完毕，Entry记录到ents变量中存储在内存里。

　　此时WAL就进入append模式，为数据项添加进行准备。

当WAL文件中数据项内容过大达到设定值（默认为10000）时，会进行WAL的切分，同时进行snapshot操作。

　　这个过程可以在etcdserver/server.go的snapshot函数中看到。

　　所以，实际上数据目录中有用的snapshot和WAL文件各只有一个，默认情况下etcd会各保留5个历史文件。

 

 

7 Raft
新版etcd中，raft包就是对Raft一致性算法的具体实现。

　　关于Raft算法的讲解，网上已经有很多文章，有兴趣的读者可以去阅读一下Raft算法论文非常精彩。

　　本文则不再对Raft算法进行详细描述，而是结合etcd，针对算法中一些关键内容以问答的形式进行讲解。

　　有关Raft算法的术语如果不理解，可以参见概念词汇表一节。

7.1 Raft常见问答一览
Raft中一个Term（任期）是什么意思？
Raft算法中，从时间上，一个任期讲即从一次竞选开始到下一次竞选开始。
从功能上讲，如果Follower接收不到Leader节点的心跳信息，就会结束当前任期，变为Candidate发起竞选，有助于Leader节点故障时集群的恢复。
发起竞选投票时，任期值小的节点不会竞选成功。
如果集群不出现故障，那么一个任期将无限延续下去。
而投票出现冲突也有可能直接进入下一任再次竞选。


Raft状态机是怎样切换的？
Raft刚开始运行时，节点默认进入Follower状态，等待Leader发来心跳信息。
若等待超时，则状态由Follower切换到Candidate进入下一轮term发起竞选，等到收到集群多数节点的投票时，该节点转变为Leader。
Leader节点有可能出现网络等故障，导致别的节点发起投票成为新term的Leader，此时原先的老Leader节点会切换为Follower。
Candidate在等待其它节点投票的过程中如果发现别的节点已经竞选成功成为Leader了，也会切换为Follower节点。


如何保证最短时间内竞选出Leader，防止竞选冲突？
在Raft状态机一图中可以看到，在Candidate状态下， 有一个times out，这里的times out时间是个随机值，也就是说，每个机器成为Candidate以后，超时发起新一轮竞选的时间是各不相同的，这就会出现一个时间差。
在时间差内，如果Candidate1收到的竞选信息比自己发起的竞选信息term值大（即对方为新一轮term），
并且新一轮想要成为Leader的Candidate2包含了所有提交的数据，那么Candidate1就会投票给Candidate2。
这样就保证了只有很小的概率会出现竞选冲突。
如何防止别的Candidate在遗漏部分数据的情况下发起投票成为Leader？
Raft竞选的机制中，使用随机值决定超时时间，第一个超时的节点就会提升term编号发起新一轮投票，一般情况下别的节点收到竞选通知就会投票。
但是，如果发起竞选的节点在上一个term中保存的已提交数据不完整，节点就会拒绝投票给它。
通过这种机制就可以防止遗漏数据的节点成为Leader。
Raft某个节点宕机后会如何？
通常情况下，如果是Follower节点宕机，如果剩余可用节点数量超过半数，集群可以几乎没有影响的正常工作。
如果是Leader节点宕机，那么Follower就收不到心跳而超时，发起竞选获得投票，成为新一轮term的Leader，继续为集群提供服务。
需要注意的是；etcd目前没有任何机制会自动去变化整个集群总共的节点数量，即如果没有人为的调用API，etcd宕机后的节点仍然被计算为总节点数中，任何请求被确认需要获得的投票数都是这个总数的半数以上。


为什么Raft算法在确定可用节点数量时不需要考虑拜占庭将军问题？
拜占庭问题中提出，允许n个节点宕机还能提供正常服务的分布式架构，需要的总节点数量为3n+1，而Raft只需要2n+1就可以了。
其主要原因在于，拜占庭将军问题中存在数据欺骗的现象，而etcd中假设所有的节点都是诚实的。
etcd在竞选前需要告诉别的节点自身的term编号以及前一轮term最终结束时的index值，这些数据都是准确的，其他节点可以根据这些值决定是否投票。
另外，etcd严格限制Leader到Follower这样的数据流向保证数据一致不会出错。
用户从集群中哪个节点读写数据？
Raft为了保证数据的强一致性，所有的数据流向都是一个方向，从Leader流向Follower，也就是所有Follower的数据必须与Leader保持一致，如果不一致会被覆盖。即所有用户更新数据的请求都最先由Leader获得，然后存下来通知其他节点也存下来，等到大多数节点反馈时再把数据提交。
一个已提交的数据项才是Raft真正稳定存储下来的数据项，不再被修改，最后再把提交的数据同步给其他Follower。
因为每个节点都有Raft已提交数据准确的备份（最坏的情况也只是已提交数据还未完全同步），所以读的请求任意一个节点都可以处理。
etcd实现的Raft算法性能如何？
单实例节点支持每秒1000次数据写入。
节点越多，由于数据同步涉及到网络延迟，会根据实际情况越来越慢，而读性能会随之变强，因为每个节点都能处理用户请求。
7.2 关键部分源码解析
在etcd代码中，Node作为Raft状态机的具体实现，是整个算法的关键，也是了解算法的入口。

在etcd中，对Raft算法的调用如下，你可以在etcdserver/raft.go中的startNode找到：

1
2
storage := raft.NewMemoryStorage()
n := raft.StartNode(0x01, []int64{0x02, 0x03}, 3, 1, storage)
通过这段代码可以了解到，Raft在运行过程记录数据和状态都是保存在内存中，而代码中raft.StartNode启动的Node就是Raft状态机Node。

 

启动了一个Node节点后，Raft会做如下事项。

1. 首先，你需要把从集群的其他机器上收到的信息推送到Node节点，你可以在etcdserver/server.go中的Process函数看到。

1
2
3
4
5
6
func (s *EtcdServer) Process(ctx context.Context, m raftpb.Message) error {
    if m.Type == raftpb.MsgApp {
        s.stats.RecvAppendReq(types.ID(m.From).String(), m.Size())
    }
    return s.node.Step(ctx, m)
}
　　在检测发来请求的机器是否是集群中的节点，自身节点是否是Follower，把发来请求的机器作为Leader，具体对Node节点信息的推送和处理则通过node.Step()函数实现。

2. 其次，你需要把日志项存储起来，在你的应用中执行提交的日志项，然后把完成信号发送给集群中的其它节点，再通过node.Ready()监听等待下一次任务执行。

　　有一点非常重要，你必须确保在你发送完成消息给其他节点之前，你的日志项内容已经确切稳定的存储下来了。

3. 最后，你需要保持一个心跳信号Tick()。

　　Raft有两个很重要的地方用到超时机制：心跳保持和Leader竞选。

　　需要用户在其raft的Node节点上周期性的调用Tick()函数，以便为超时机制服务。

 

综上所述，整个raft节点的状态机循环类似如下所示：

1
2
3
4
5
6
7
8
9
10
11
12
13
for {
    select {
    case <-s.Ticker:
        n.Tick()
    case rd := <-s.Node.Ready():
        saveToStorage(rd.State, rd.Entries)
        send(rd.Messages)
        process(rd.CommittedEntries)
        s.Node.Advance()
    case <-s.done:
        return
    }
}
　　而这个状态机真实存在的代码位置为etcdserver/server.go中的run函数。

 

对状态机进行状态变更（如用户数据更新等）则是调用n.Propose(ctx, data)函数，在存储数据时，会先进行序列化操作。

　　获得大多数其他节点的确认后，数据会被提交，存为已提交状态。

 

之前提到etcd集群的启动需要借助别的etcd集群或者DNS，而启动完毕后这些外力就不需要了，etcd会把自身集群的信息作为状态存储起来。

　　所以要变更自身集群节点数量实际上也需要像用户数据变更那样添加数据条目到Raft状态机中。这一切由n.ProposeConfChange(ctx, cc)实现。

　　当集群配置信息变更的请求同样得到大多数节点的确认反馈后，再进行配置变更的正式操作，代码如下。

1
2
3
var cc raftpb.ConfChange
cc.Unmarshal(data)
n.ApplyConfChange(cc)
　   注意：一个ID唯一性的表示了一个集群，所以为了避免不同etcd集群消息混乱，ID需要确保唯一性，不能重复使用旧的token数据作为ID。

 

 

8 Store
Store这个模块顾名思义，就像一个商店把etcd已经准备好的各项底层支持加工起来，为用户提供五花八门的API支持，处理用户的各项请求。

要理解Store，只需要从etcd的API入手即可。打开etcd的API列表，我们可以看到有如下API是对etcd存储的键值进行的操作，亦即Store提供的内容。

API中提到的目录（Directory）和键（Key），上文中也可能称为etcd节点（Node）。

为etcd存储的键赋值
1
2
3
4
5
6
7
8
9
10
curl http://127.0.0.1:2379/v2/keys/message -XPUT -d value="Hello world"
{
    "action":"set",
    "node": {
        "createdIndex": 2,
        "key":"/message",
        "modifiedIndex": 2,
        "value":"Hello world"
    }
}
　　反馈的内容含义如下：

action: 刚刚进行的动作名称。
node.key: 请求的HTTP路径。etcd使用一个类似文件系统的方式来反映键值存储的内容。
node.value: 刚刚请求的键所存储的内容。
node.createdIndex: etcd节点每次有变化时都会自增的一个值，除了用户请求外，etcd内部运行（如启动、集群信息变化等）也会对节点有变动而引起这个值的变化。　　
node.modifiedIndex: 类似node.createdIndex，能引起modifiedIndex变化的操作包括set, delete, update, create, compareAndSwap and compareAndDelete。
 

查询etcd某个键存储的值
1
curl http://127.0.0.1:2379/v2/keys/message
 

修改键值：与创建新值几乎相同，但是反馈时会有一个prevNode值反应了修改前存储的内容。
1
curl http://127.0.0.1:2379/v2/keys/message -XPUT -d value="Hello etcd"
　　

删除一个值
1
curl http://127.0.0.1:2379/v2/keys/message -XDELETE
　　

对一个键进行定时删除：etcd中对键进行定时删除，设定一个TTL值，当这个值到期时键就会被删除。反馈的内容会给出expiration项告知超时时间，ttl项告知设定的时长。
1
curl http://127.0.0.1:2379/v2/keys/foo -XPUT -d value=bar -d ttl=5
　　

取消定时删除任务
1
curl http://127.0.0.1:2379/v2/keys/foo -XPUT -d value=bar -d ttl= -d prevExist=true
　　

对键值修改进行监控：etcd提供的这个API让用户可以监控一个值或者递归式的监控一个目录及其子目录的值，当目录或值发生变化时，etcd会主动通知。
1
curl http://127.0.0.1:2379/v2/keys/foo?wait=true
　　

对过去的键值操作进行查询：类似上面提到的监控，只不过监控时加上了过去某次修改的索引编号，就可以查询历史操作。默认可查询的历史记录为1000条。
1
curl'http://127.0.0.1:2379/v2/keys/foo?wait=true&waitIndex=7'
　　

自动在目录下创建有序键。在对创建的目录使用POST参数，会自动在该目录下创建一个以createdIndex值为键的值，这样就相当于以创建时间先后严格排序了。这个API对分布式队列这类场景非常有用。
1
2
3
4
5
6
7
8
9
10
curl http://127.0.0.1:2379/v2/keys/queue -XPOST -d value=Job1
{
    "action":"create",
    "node": {
        "createdIndex": 6,
        "key":"/queue/6",
        "modifiedIndex": 6,
        "value":"Job1"
    }
}
　　

按顺序列出所有创建的有序键。
1
curl -s'http://127.0.0.1:2379/v2/keys/queue?recursive=true&sorted=true'
　　

创建定时删除的目录：就跟定时删除某个键类似。如果目录因为超时被删除了，其下的所有内容也自动超时删除。
1
curl http://127.0.0.1:2379/v2/keys/dir -XPUT -d ttl=30 -ddir=true
　　

刷新超时时间
1
curl http://127.0.0.1:2379/v2/keys/dir -XPUT -d ttl=30 -ddir=true -d prevExist=true
　　

自动化CAS（Compare-and-Swap）操作：etcd强一致性最直观的表现就是这个API，通过设定条件，阻止节点二次创建或修改。即用户的指令被执行当且仅当CAS的条件成立。条件有以下几个。
prevValue 先前节点的值，如果值与提供的值相同才允许操作。
prevIndex 先前节点的编号，编号与提供的校验编号相同才允许操作。
prevExist 先前节点是否存在。如果存在则不允许操作。这个常常被用于分布式锁的唯一获取。
假设先进行了如下操作：设定了foo的值。
1
curl http://127.0.0.1:2379/v2/keys/foo -XPUT -d value=one
　　

然后再进行操作：
1
curl http://127.0.0.1:2379/v2/keys/foo?prevExist=false -XPUT -d value=three
　　

就会返回创建失败的错误。
条件删除（Compare-and-Delete）：与CAS类似，条件成立后才能删除。
1
curl http://127.0.0.1:2379/v2/keys/dir -XPUT -ddir=true
　　

创建目录
1
curl http://127.0.0.1:2379/v2/keys/
　　

列出目录下所有的节点信息，最后以/结尾。还可以通过recursive参数递归列出所有子目录信息。
1
curl http://127.0.0.1:2379/v2/keys/
　　

删除目录：默认情况下只允许删除空目录，如果要删除有内容的目录需要加上recursive=true参数。
1
curl'http://127.0.0.1:2379/v2/keys/foo_dir?dir=true' -XDELETE
　　

创建一个隐藏节点：命名时名字以下划线_开头默认就是隐藏键。
1
curl http://127.0.0.1:2379/v2/keys/_message -XPUT -d value="Hello hidden world"
　　

相信看完这么多API，读者已经对Store的工作内容基本了解了。

　　它对etcd下存储的数据进行加工，创建出如文件系统般的树状结构供用户快速查询。

　　它有一个Watcher用于节点变更的实时反馈，还需要维护一个WatcherHub对所有Watcher订阅者进行通知的推送。

　　同时，它还维护了一个由定时键构成的小顶堆，快速返回下一个要超时的键。

最后，所有这些API的请求都以事件的形式存储在事件队列中等待处理。

 

9 总结
通过从应用场景到源码分析的一系列回顾，我们了解到etcd并不是一个简单的分布式键值存储系统。

它解决了分布式场景中最为常见的一致性问题，为服务发现提供了一个稳定高可用的消息注册仓库，为以微服务协同工作的架构提供了无限的可能。

相信在不久的将来，通过etcd构建起来的大型系统会越来越多。
