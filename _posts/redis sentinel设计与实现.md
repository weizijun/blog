title:      redis sentinel设计与实现
categories: redis
tags: 
- redis
- sentinel
date: 2015-04-30 00:00:00
---

Sentinel是Redis官方自带的工具，中文意思是哨兵，顾名思义，就是守卫Redis的好帮手。NCR（网易云redis）准备开发高可用的Redis集群，有计划使用Sentinel。本文接下来介绍下Sentinel的设计与实现。本文介绍的Sentinel是用的2.8.19版本（最新的3.0.0版本Sentinel的功能只比2.8.19多了一个client命令）。

先引用官方的说法介绍下Sentinel的作用。Redis Sentinel是一个帮助管理Redis实例的系统，它提供以下功能：

* 监控（Monitoring）：Sentinel会不断的检查你的主节点和从节点是否正常工作。
* 通知（Notification）：被监控的Redis实例如果出现问题，Sentinel可以通过API（pub）通知系统管理员或者其他程序。
* 自动故障转移（Automatic failover）：如果一个master离线，Sentinel会开始进行故障转移，master下的一个slave会被选为新的master，其他的slave会开始复制新的master。应用可以通过Redis服务的通知机制更新新的master地址。
* 配置提供者（Configuration provider）：客户端可以把Sentinel作为权威的配置发布者来获得最新的master地址。如果发生了故障转移，Sentinel集群会通知客户端新的master地址。
	
Sentinel是如何运行的？又是如何提供对Redis的监控和故障转移呢？

![image](http://nos.netease.com/knowledge/f66a5bfe-51e3-4c75-b694-2c6e1d1b2e4b)
	
### 特殊状态的Redis
一个Sentinel就是一个运行在特殊状态下的Redis，以至于可以用启动Redis Server的方式启动Sentinel。不过Sentinel有自己的命令列表，它支持的命令不多。

	struct redisCommand sentinelcmds[] = {
	    {"ping",pingCommand,1,"",0,NULL,0,0,0,0,0},
	    {"sentinel",sentinelCommand,-2,"",0,NULL,0,0,0,0,0},
	    {"subscribe",subscribeCommand,-2,"",0,NULL,0,0,0,0,0},
	    {"unsubscribe",unsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
	    {"psubscribe",psubscribeCommand,-2,"",0,NULL,0,0,0,0,0},
	    {"punsubscribe",punsubscribeCommand,-1,"",0,NULL,0,0,0,0,0},
	    {"publish",sentinelPublishCommand,3,"",0,NULL,0,0,0,0,0},
	    {"info",sentinelInfoCommand,-1,"",0,NULL,0,0,0,0,0},
	    {"role",sentinelRoleCommand,1,"l",0,NULL,0,0,0,0,0},
	    {"shutdown",shutdownCommand,-1,"",0,NULL,0,0,0,0,0}
	};
	
上面就是Sentinel支持的所有命令了。后面会对这些命令进行详细的说明。

### 配置Redis主节点，自动发现从节点
Sentinel运行起来后，会根据配置文件的`"sentinel monitor <master-name> <ip> <redis-port> <quorum>"`去连接master，这里我们把master-name下的所有节点看成一个group，一个group由一个master，若干个slave组成。Sentinel只需配置主节点的ip、port即可。Sentinel会通过向master发送INFO命令来获取master下的slave，然后把slave加入group。

### 自动发现监控同一个group的其他Sentinel
Sentinel连接Redis节点，会创建两条对节点的连接，一条用来向节点发送命令，另一条用来订阅`“__sentinel__:hello”`频道（hello频道）发来的消息，这个频道用做Gossip协议发现其他监控此Redis节点的Sentinel。每个Sentinel会定时向频道发送消息，然后也会接收到其他Sentinel从hello频道发来的消息，消息的格式如下：`sentinel_ip,sentinel_port,sentinel_runid,current_epoch,master_name,master_ip,master_port,master_config_epoch`（127.0.0.1,26381,99ce8dc79e55ce9de040b0cd13d152900db9a7e1,24,mymaster2,127.0.0.1,8000,0）。

这样监控同一个group的的Sentinel就组成了一个集群，每个Sentinel会创建一条连向其他Sentinel的连接，这是在做自动故障转移的时候可以像其他节点发送命令。

### 推送消息
Sentinel在监控Redis和其他Sentinel的时候，发现的异常以及完成的操作都会通过Publish的方式推送出去，客户端想了解Sentinel的处理结果，只要订阅相应的消息类型即可。Sentinel使用的推送方式用的是Redis现有的Pub/Sub方式。Sentinel基本上会把它整个处理流程都推送出来，然而客户端一般只要关注主从切换的消息即可。

### 故障检查
Sentinel会跟group的每个节点以及监控同一个group的其他Sentinel保持心跳，Sentinel会定时向这些节点发送Ping命令，然后等待Pong命令回复。如果一段时间没有收到Pong命令。Sentinel就会主观的认为该节点离线。对于其他Sentinel和group内的slave挂了，Sentinel检测到他们离线也不需要做什么事情，只是简单的推送一条`+sdown`的消息。如果检测到master离线，Sentinel就要确定是否需要进行主从切换。此时Sentinel会向其他Sentinel发送`is-master-down-by-addr`命令（命令格式：`SENTINEL IS-master-DOWN-BY-ADDR <ip> <port> <current-epoch> <runid>`），这个命令有2个功能，这时候的用法是用来向其他节点获取master是否离线的信息。还有一个用法是用来选举leader的，该命令会让其他Sentinel给自己投票，已经投过票的Sentinel会返回投票的结果。Sentinel在监控group的时候会配置一个quorum，Sentinel接收到超过quorum个Sentinel认为master挂了（quorum包含自己），Sentinel就会认为该master是客观下线了。接着Sentinel就进入了自动故障转移状态。

### Sentinel间投票选leader
Sentinel认为master客观下线了，就开始故障转移流程，故障转移的第一步就是竞选leaderSentinel采用了Raft协议实现了Sentinel间选举Leader的算法，不过也不完全跟论文描述的步骤一致。Sentinel集群运行过程中故障转移完成，所有Sentinel又会恢复平等。Leader仅仅是故障转移操作出现的角色。

#### 选举流程
Sentinel采用了Raft协议实现了Sentinel间选举Leader的算法，不过也不完全跟论文描述的步骤一致。

* 1、某个Sentinel认定master客观下线的节点后，该Sentinel会先看看自己有没有投过票，如果自己已经投过票给其他Sentinel了，在2倍故障转移的超时时间自己就不会成为Leader。相当于它是一个Follower。
* 2、如果该Sentinel还没投过票，那么它就成为Candidate。
* 3、和Raft协议描述的一样，成为Candidate，Sentinel需要完成几件事情
	* 1）更新故障转移状态为start
	* 2）当前epoch加1，相当于进入一个新term，在Sentinel中epoch就是Raft协议中的term。
	* 3）更新自己的超时时间为当前时间随机加上一段时间，随机时间为1s内的随机毫秒数。
	* 4）向其他节点发送`is-master-down-by-addr`命令请求投票。命令会带上自己的epoch。
	* 5）给自己投一票，在Sentinel中，投票的方式是把自己master结构体里的leader和leader_epoch改成投给的Sentinel和它的epoch。
* 4、其他Sentinel会收到Candidate的`is-master-down-by-addr`命令。如果Sentinel当前epoch和Candidate传给他的epoch一样，说明他已经把自己master结构体里的leader和leader_epoch改成其他Candidate，相当于把票投给了其他Candidate。投过票给别的Sentinel后，在当前epoch内自己就只能成为Follower。
* 5、Candidate会不断的统计自己的票数，直到他发现认同他成为Leader的票数超过一半而且超过它配置的quorum（quorum可以参考《redis sentinel(哨兵) 设计与实现》）。Sentinel比Raft协议增加了quorum，这样一个Sentinel能否当选Leader还取决于它配置的quorum。
* 6、如果在一个选举时间内，Candidate没有获得超过一半且超过它配置的quorum的票数，自己的这次选举就失败了。
* 7、如果在一个epoch内，没有一个Candidate获得更多的票数。那么等待超过2倍故障转移的超时时间后，Candidate增加epoch重新投票。
* 8、如果某个Candidate获得超过一半且超过它配置的quorum的票数，那么它就成为了Leader。
* 9、与Raft协议不同，Leader并不会把自己成为Leader的消息发给其他Sentinel。其他Sentinel等待Leader从slave选出master后，检测到新的master正常工作后，就会去掉客观下线的标识，从而不需要进入故障转移流程。

#### 关于Sentinel超时时间的说明
Sentinel超时机制并不像Raft协议描述的那样只使用了一个随机超时机制。它有几个超时概念。

* failover_start_time 下一选举启动的时间。默认是当前时间加上1s内的随机毫秒数
* failover_state_change_time 故障转移中状态变更的时间。
* failover_timeout 故障转移超时时间。默认是3分钟。
* election_timeout 选举超时时间，是默认选举超时时间和failover_timeout的最小值。默认是10s。

Follower成为Candidate后，会更新failover_start_time为当前时间加上1s内的随机毫秒数。更新failover_state_change_time为当前时间。

Candidate的当前时间减去failover_start_time大于election_timeout，说明Candidate还没获得足够的选票，此次epoch的选举已经超时，那么转变成Follower。需要等到`mstime() - failover_start_time < failover_timeout*2`的时候才开始下一次获得成为Candidate的机会。

如果一个Follower把某个Candidate设为自己认为的Leader，那么它的failover_start_time会设置为当前时间加上1s内的随机毫秒数。这样它就进入了上面说的需要等到`mstime() - failover_start_time < failover_timeout*2`的时候才开始下一次获得成为Candidate的机会。

因为每个Sentinel判断节点客观下线的时间不是同时开始的，一般都有先后，这样先开始的Sentinel就更有机会赢得更多选票，另外failover_state_change_time为1s内的随机毫秒数，这样也把各个节点的超时时间分散开来。本人尝试过很多次，Sentinel间的Leader选举过程基本上一个epoch内就完成了。

### 故障转移
Sentinel一旦确定自己是leader后，就开始从slave中选出一个节点来作为master。以下是选择slave的流程：

*	1、过滤掉slave列表中主观、客观下线和离线的slave。
*	2、过滤掉slave列表中5s没响应ping的slave。
*	3、过滤掉slave列表中优先级为0的slave。（slave的优先级是redis的配置参数slave-priority，默认是100）
*	3、过滤掉slave列表中一段时间没有回复INFO的slave。
*	4、过滤掉slave列表中很久没跟master连接的slave。
*	5、比较筛选后的slave，优先级小的slave被选为新master。
*	6、如果优先级相同，比较slave对原master的复制偏移量，偏移量大的slave被选为新master。
*	7、如果复制偏移量相同，那就直接比较slave的运行id，字符串小的slave被选为新master。

如果依据选择流程没有选出可用的slave，leader Sentinel会终止本次故障转移。

如果选择除了可用的slave，那么leader Sentinel会给该slave发送`slaveof no one`命令，表示该slave不再复制其他节点，成为了master。然后leader Sentinel就会一直等待选出slave的INFO信息里面确认了自己的master身份。如果等待超时了，leader Sentinel只得终止本次故障转移。

如果选出的slave确认了自己的master身份，leader Sentinel会让其他slave复制新的master，由于初次复制会带来很大的IO开销，Sentinel有个`parallel_syncs`参数，用来确定一次让多少个slave复制新master。一个slave复制master如果超过10s，leader Sentinel会重新发送复制命令。如果在指定的故障转移时间内还没有完成全部的复制工作，leader Sentinel就会忽略那些没复制的slave。leader Sentinel只是向这些slave发送一次复制命令，不等待他们复制成功就直接完成了全部故障转移工作。

全部故障转移工作完成后，leader Sentinel就会推送`+switch-master`消息，同时重置master，重置操作会释放掉原来master全部的slave对象和监听该master的其他Sentinel对象，然后创建出新的slave对象。

### TILT保护模式
TILT模式是Sentinel发现进程出现异常时候的一种保护模式。Sentinel定时器默认每100ms执行一次，Sentinel每次启动定时任务的时候会检查下上次定时任务执行的时间，如果超过2s或者小于0了，Sentinel就认为操作系统出现异常，导致一次任务的执行时间过长，就会进入TILT模式。TILT模式下，Sentinel不再执行任何操作。其他Sentinel发送的`SENTINEL is-master-down-by-addr`命令会直接返回负值。

如果TILT模式下的Sentinel正常运行超过30s，Sentinel就会解除TILT模式。

### 客户端处理流程
客户端可以通过Sentinel获得group的信息。官方给出了客户端操作的推荐方式。看了下Jedis的实现，基本就是按照官方的操作流程进行的。首先客户端配置监听该group的全部Sentinel。连接第一个Sentinel，如果连不上就重新连接下一个，直到连上一个Sentinel。

连上Sentinel后，发送`SENTINEL get-master-addr-by-name master-name`命令可以得到该group的master，如果该Sentinel返回了null，那就重复上面的流程，重新连接下一个Sentinel。

得到group的master后，连上master，发送ROLE命令，确认该master自身确实是作为master在运行。这个确认是必须的，如果Sentinel和group网络分区了，那么该Sentinel认为的master就不会变化了，而group如果出现主从切换，此时Sentinel就拿不到真实的master了。如果ROLE得到的不再是master了，客户端需要重复最前面的流程，重新连接下一个Sentinel。

确认好master的ROLE也是master后，客户端可以从每个Sentinel上订阅消息。一般客户端只要关心`+switch-master`即可，这个消息会告诉客户端发生了主从切换，并把新老master的ip、port都推送在消息里。客户端根据新的master，发送ROLE命令确认后，就可以和新的master通信了。

有些客户端希望把读流量分给slave，那么可以通过`SENTINEL slaves master-name`命令来获得该group下的slave列表。

如果客户端需要重连master，那么建议按照初始化连接的方式重新从Sentinel获取master。

如果采用连接池的方式，官方建议在每次有连接断开需要重连的时候所有的连接都关闭，从而重建连接池。这么做也是为了防止新的连接获取的master跟原来不一致了。

客户端还可以通过`SENTINEL sentinels <master-name>`命令更新自己的Sentinel列表，从而获得最新存活的Sentinel。

### 状态持久化
Sentinel的状态会持久化到配置文件。例如每一次通过set命令设置新的配置，或者加入新节点的监控，修改的配置会持久化到配置文件，同时持久化配置的epoch，这意味着停止和重启Sentinel是安全的。

### 分区问题
Sentinel集群会有分区问题，这个在官方文档上有说明。

![](http://nos.netease.com/knowledge/281654ee-ae73-456e-a4eb-6c88b4b97721)

在该图中，Redis 3原来是master，网络分区后，Sentinel1和Sentinel2会把Redis 1选举为master。此时问题出现了，由于Sentinel3和Redis3处于另外分区，所以Sentinel3依然认为Redis3是master，此时处于分区内的Client B从Sentinel3获得的master就是Redis3，而Redis3也认为自己是Redis3，客户端依然能够操作group，此时Client A和Client B操作的master已经不一样了，同一个group出现了不一致现象。Redis官方给出了两个建议。

等分区恢复后，Client B在Redis3上的写数据会丢失，如果你把Redis当做缓存，能够接受这种现象，那么可以忽略这个问题。

如果不想忽略这个问题，那么对于redis可以这样配置

	min-slaves-to-write 1
	min-slaves-max-lag 10

`min-slaves-to-write`设置为1，保证了master认为至少有一个slave连接正常，master才能正常工作，上面Redis3因为已经与原来的两个slave无法连接，所以Redis3此时已经无效了。（`min-slaves-max-lag`是主从ack延迟的最大时间。单位是秒）

### 连接爆炸

Sentinel还有个连接爆炸的问题。Sentinel的所有操作都是基于group进行的，不同group之间流程完全不干扰，Sentinel会去发现监控相同group的其他Sentinel，即使一个在其他group的Sentinel已经和本Sentinel建立了连接，在这个group内，也仍然会继续建立连接，同时从这个连接发送ping命令确定其他Sentinel的存活，这个带来的好处是流程简洁，代码清晰。但缺点就是带来了连接数的消耗和大量的重复消息。下面我们从代码的层面看下出现连接爆炸的原因。整个Sentinel主要就使用了两个结构体。一个是sentinelState，用来记录Sentinel的全局状态。另一个是sentinelRedisInstance，用来记录Sentinel监控的每一个节点的信息。sentinelState有一个记录所有group的hash表，`dict *masters`。masters记录了所有主节点的信息，hash表的键是主节点的名称，值是一个sentinelRedisInstance结构。

	struct sentinelState {
		...
	    dict *masters;      /* Dictionary of master sentinelRedisInstances.
	                           Key is the instance name, value is the
	                           sentinelRedisInstance structure pointer. */
	    ...
	} sentinel;
	
sentinelRedisInstance结构里面记录了整个group的详细信息，其中包括slave的hash表和sentinel的hash表。

	typedef struct sentinelRedisInstance {
		...
	    dict *sentinels;    /* Other sentinels monitoring the same master. */
	    dict *slaves;       /* slaves for this master instance. */
	    ...
	} sentinelRedisInstance;	

`dict *sentinels`记录了监控该group除自己外的其他Sentinel。问题就出在这里。前面介绍了Sentinel发现其他Sentinel是通过订阅hello频道的消息。Sentinel会订阅每个group的消息，然后当在一个group的hello频道发现一个新的Sentinel后，Sentinel会为这个Sentinel生成一个新的sentinelRedisInstance结构，加入该group的sentinels的hash表里面。每次生成新的sentinelRedisInstance结构，都是从内存重新分配数据，重新和Sentinel建立连接。这样如果另外也有一个Sentinel和自己监听了相同group列表的话，他们会针对每个group彼此都建立一条连接。随着Sentinel监听的group针对，连接将成倍数的增加！这里还没结束。Sentinel有个轮询线程会监听每个节点的状态。这样，对另一个Sentinel，本Sentinel会对每个group上建立的连接向另一个Sentinel发送Ping命令，从而产生了大量的重复消息。

### Sentinel命令列表
以下对Sentinel的每个命令做个说明。

	* ping 用来探测节点的存活，正常会返回pong。
	* sentinel 下面也很多子命令
	       masters(SENTINEL masterS) 获得Sentinel监控的所有master信息
	       master  (SENTINEL master <name>) 获得某个master信息
	       slaves  (SENTINEL slaveS <master-name>) 获得某个master信息
	       sentinels  (SENTINEL SENTINELS <master-name>) 获得监控同一个group的其他sentinel信息
	       is-master-down-by-addr  (SENTINEL IS-master-DOWN-BY-ADDR <ip> <port> <current-epoch> <runid>*/ 获得该Sentinel对于某个master存活状态，同时让该Sentinel为自己投票
	       reset  (SENTINEL RESET <pattern>) 重置指定pattern的group信息
	       get-master-addr-by-name  (SENTINEL GET-master-ADDR-BY-NAME <master-name>)	获得某个group下master信息
	       failover  (SENTINEL FAILOVER <master-name>) 主动触发一次故障转移
	       pending-scripts  (SENTINEL PENDING-SCRIPTS) 执行脚本
	       monitor  (SENTINEL MONITOR <name> <ip> <port> <quorum>) 开始监控某个group
	       remove  (SENTINEL REMOVE <name>) 移除对该group的监控
	       set       (SENTINEL SET <mastername> [<option> <value> ...]) sentinel动态设置Sentinel配置命令
	            down-after-milliseconds  (down-after-millisecodns <milliseconds>) 设置该<mastername> ping多长时间未响应才判定为离线，默认是30s
	            failover-timeout            (failover-timeout <milliseconds>) 故障转移的超时时间，默认是180s
	            parallel-syncs            (parallel-syncs <milliseconds>) 同时让多少个slave复制新的master，默认是1
	            notification-script       (notification-script <path>) 用于通知管理员的脚本的地址
	            client-reconfig-script       (client-reconfig-script <path>) 需要执行的脚本的地址
	            auth-pass                 (auth-pass <password>) group的master密码
	            quorum                      (quorum <count>) 用于配置多少个Sentinel认为节点下线才认为客观下线的数量。
	* subscribe 订阅某个频道
	* unsubscribe 退订某个频道
	* psubscribe 订阅某个模式，可以批量订阅一些频道
	* punsubscribe 退订某个模式
	* publish 发布消息，用作测试。只能发布hello频道的消息
	* info 查看Sentinel信息。
	* role	查看Sentinel和负责监控的master列表。
	* shutdown 关闭Sentinel



### 推送的消息内容
以下是Sentinel推送的所有消息。从Sentinel的推送消息，基本上可以看到Sentinel运行的整个流程。

*	+monitor <master> quorum <quorum num> 有新的master被监控。
*	+reset-master <master> master器已被重置。
*	+slave <sentinelRedisInstance> 一个新的slave已经被Sentinel检测到并关联到对应的master。
*	-pubsub-link <pub-link> <error> 订阅连接断线。
*	-cmd-link <cmd-link> <error> 命令连接断开。
*	+pubsub-link <pub-link> 推送订阅连接连上的消息。
*	+cmd-link <cmd-link> 推送命令连接连上的消息。
*	-cmd-link-reconnection <cmd-link> <error> 命令连接重连出错。
*	-pubsub-link-reconnection <pub-link> <error> 订阅连接重连出错。
*	+reboot <sentinelRedisInstance> 节点重启，更换了新的runid。
*	+role-change <sentinelRedisInstance> new reported role is <new role> 通过解析info信息发现role可能有变化，role发生变化。
*	-role-change  <sentinelRedisInstance> new reported role is <new role> 通过解析info信息发现role可能有变化，role没有发生变化。
*	+promoted-slave <sentinelRedisInstance> 故障转移过测中，新的master自己确认了master的角色。
*	+failover-state-reconf-slaves <sentinelRedisInstance> <old master> 该消息紧接着+promoted-slave消息，此时故障转移已经进入了reconf-slaves状态。
*	+convert-to-slave <new master> 新的master一段时间没有确认自己的master角色，而且它原来的master已经运行正常了，则重新复制原来的master，自己重新做回slave。
*	+fix-slave-config <sentinelRedisInstance> slave现在的master地址和Sentinel保存的master不一致，则让slave重新复制Sentinel认为的master。
*	+slave-reconf-inprog <sentinelRedisInstance> slave正在复制master。
*	+slave-reconf-done <sentinelRedisInstance> slave复制master完成。	*	-dup-sentinel <master> #duplicate of <ip>:<port> or <dup runid> 通过hello频道得到的Sentinel信息与已经保存的Sentinel信息产生冲突，需要被移除 —— 当 Sentinel 实例重启的时候，就会出现这种情况。
*	+sentinel <new sentinel> 一个新的Sentinel已经被Sentinel检测到并关联到对应的master。
*	+new-epoch <current-epoch> 当前epoch被更新。
*	+config-update-from <sentinelRedisInstance> leader完成故障转移后，其他Sentinel通过hello频道获得新的配置信息。
*	`+switch-master <master-name> <old master ip> <old master port> <new master ip> <new master port>` 配置变更，maseter的IP和port已经改变。这是绝大多数外部用户都关心的信息。
*	-monitor <master> 去掉了对该master的监控。
*	+set <master> <option> <value> 设置命令生效。
*	+sdown  <sentinelRedisInstance> 给定的实例现在处于主观下线状态。
*	-sdown  <sentinelRedisInstance> 给定的实例已经不再处于主观下线状态。
*	+odown  <sentinelRedisInstance> #quorum <get quorum>/<config quorum> 给定的实例现在处于客观下线状态。
*	-odown  <sentinelRedisInstance> 给定的实例已经不再处于客观下线状态。
*	+vote-for-leader  <leader> <leader_epoch> 把某个Sentinel设置为leader。
*	+try-failover  <sentinelRedisInstance> 尝试故障迁移操作，等待被大多数Sentinel选中。
*	-failover-abort-not-elected <sentinelRedisInstance> Sentinel 的当选时间已过，取消故障转移计划。
*	+elected-leader  <sentinelRedisInstance> 赢得指定epoch的选举，可以进行故障迁移操作。
*	+failover-state-select-slave  <sentinelRedisInstance> 故障转移步骤中开始选择slave
*	-failover-abort-no-good-slave  <sentinelRedisInstance> Sentinel没有找到合适的slave提升为master，一段时间后将重试，但是也可能在重试的时候出现相同的找不到合适slave的情况。
*	+selected-slave  <slave> 故障转移中，选出了slave作为新的master。
*	+failover-state-send-slaveof-noone  <slave> 故障转移中，选出了slave作为新的master后准备把slave提升为master。
*	-failover-abort-slave-timeout <slave> slave提升为master超时。
*	+failover-state-wait-promotion <new master> 新master执行slaveof no one成功。
*	+failover-end-for-timeout <new master> 故障转移超过时间，还有slave没有复制完新的master。
*	+failover-end  <new master> 故障转移顺利结束。
*	+slave-reconf-sent-be  <slave> 故障转移超过时间，还有slave没有复制完新的master。这些slave将直接发送复制新master命令后就完成了整个故障转移操作。
*	-slave-reconf-sent-timeout  <slave> 超时的slave也正常完成复制工作。
*	+slave-reconf-sent  <slave> leader Sentinel 向slave发送了复制命令，为slave设置新的master。
*	-tilt  #tilt mode exited 退出 tilt 模式。
*	+tilt  #tilt mode entered 进入 tilt 模式。


参考资料：

[Redis 2.8.19 source code](https://github.com/antirez/redis/tree/2.8.19)

[http://redis.io/topics/sentinel](http://redis.io/topics/sentinel)

[http://redis.io/topics/sentinel-clients](http://redis.io/topics/sentinel-clients)

[http://redisdoc.com/topic/sentinel.html](http://redisdoc.com/topic/sentinel.html)

[http://www.wzxue.com/redis核心解读-集群管理工具redis-sentinel](http://www.wzxue.com/redis核心解读-集群管理工具redis-sentinel)

《In Search of an Understandable Consensus Algorithm》 Diego Ongaro and John Ousterhout Stanford University

《Redis设计与实现》黄健宏 机械工业出版社


