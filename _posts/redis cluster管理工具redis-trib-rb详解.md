title: redis cluster管理工具redis-trib.rb详解
categories: redis
tags: 
- redis
- redis-trib
- cluster
- 分布式
date: 2016-01-08 00:00:00

---

### 概述

redis-trib.rb是redis官方推出的管理redis集群的工具，集成在redis的源码src目录下，是基于redis提供的集群命令封装成简单、便捷、实用的操作工具。redis-trib.rb是redis作者用ruby完成的。为了看懂redis-trib.rb，我特意花了一个星期学习了ruby，也被ruby的简洁、明了所吸引。ruby是门非常灵活的语言，redis-trib.rb只用了1600行左右的代码，就实现了强大的集群操作。本文对redis-trib.rb的介绍是基于redis 3.0.6版本的源码。阅读本文需要对redis集群功能有一定的了解。关于redis集群功能的介绍，可以参考本人的另一篇文章[《redis3.0 cluster功能介绍》](http://weizijun.cn/2015/12/11/redis3.0%20cluster%E5%8A%9F%E8%83%BD%E4%BB%8B%E7%BB%8D/)。

先从redis-trib.rb的help信息，看下redis-trib.rb提供了哪些功能。

```
  $ruby redis-trib.rb help
  Usage: redis-trib <command> <options> <arguments ...>

    create          host1:port1 ... hostN:portN
                    --replicas <arg>
    check           host:port
    info            host:port
    fix             host:port
                    --timeout <arg>
    reshard         host:port
                    --from <arg>
                    --to <arg>
                    --slots <arg>
                    --yes
                    --timeout <arg>
                    --pipeline <arg>
    rebalance       host:port
                    --weight <arg>
                    --auto-weights
                    --threshold <arg>
                    --use-empty-masters
                    --timeout <arg>
                    --simulate
                    --pipeline <arg>
    add-node        new_host:new_port existing_host:existing_port
                    --slave
                    --master-id <arg>
    del-node        host:port node_id
    set-timeout     host:port milliseconds
    call            host:port command arg arg .. arg
    import          host:port
                    --from <arg>
                    --copy
                    --replace
    help            (show this help)

  For check, fix, reshard, del-node, set-timeout you can specify the host and port of any working node in the cluster.
```

可以看到redis-trib.rb具有以下功能：

*	1、`create`：创建集群
*	2、`check`：检查集群
*	3、`info`：查看集群信息
*	4、`fix`：修复集群
*	5、`reshard`：在线迁移slot
*	6、`rebalance`：平衡集群节点slot数量
*	7、`add-node`：将新节点加入集群
*	8、`del-node`：从集群中删除节点
*	9、`set-timeout`：设置集群节点间心跳连接的超时时间
*	10、`call`：在集群全部节点上执行命令
*	11、`import`：将外部redis数据导入集群

下面从redis-trib.rb使用和源码的角度详细介绍redis-trib.rb的每个功能。

redis-trib.rb主要有两个类：`ClusterNode`和`RedisTrib`。`ClusterNode`保存了每个节点的信息，`RedisTrib`则是redis-trib.rb各个功能的实现。

### ClusterNode对象

先分析`ClusterNode`源码。`ClusterNode`有下面几个成员变量（ruby的类成员变量是以@开头的）：

*	`@r`：执行redis命令的客户端对象。
*	`@info`：保存了该节点的详细信息，包括cluster nodes命令中自己这行的信息和cluster info的信息。
*	`@dirty`：节点信息是否需要更新，如果为true，我们需要把内存的节点更新信息到节点上。
*	`@friends`：保存了集群其他节点的info信息。其信息为通过cluster nodes命令获得的其他节点信息。

`ClusterNode`有下面一些成员方法：

*	`initialize`：ClusterNode的构造方法，需要传入节点的地址信息。
*	`friends`：返回@friends对象。
*	`slots`：返回该节点负责的slots信息。
*	`has_flag?`：判断节点info信息的的flags中是否有给定的flag。
*	`to_s`：类似java的toString方法，返回节点的地址信息。
*	`connect`：连接redis节点。
*	`assert_cluster`：判断节点开启了集群配置。
*	`assert_empty`：确定节点目前没有跟任何其他节点握手，同时自己的db数据为空。
*	`load_info`：通过cluster info和cluster nodes导入节点信息。
*	`add_slots`：给节点增加slot，该操作只是在内存中修改，并把dirty设置成true，等待flush_node_config将内存中的数据同步在节点执行。
*	`set_as_replica`：slave设置复制的master地址。dirty设置成true。
*	`flush_node_config`：将内存的数据修改同步在集群节点中执行。
*	`info_string`：简单的info信息。
*	`get_config_signature`：用来验证集群节点见的cluster nodes信息是否一致。该方法返回节点的签名信息。
*	`info`：返回@info对象，包含详细的info信息。
*	`is_dirty?`：判断@dirty。
*	`r`：返回执行redis命令的客户端对象。

有了ClusterNode对象，在处理集群操作的时候，就获得了集群的信息，可以进行集群相关操作。在此先简单介绍下redis-trib.rb脚本的使用，以create为例：

	  create host1:port1 ... hostN:portN
	         --replicas <arg>

`host1:port1 ... hostN:portN`表示子参数，这个必须在可选参数之后，`--replicas <arg>`是可选参数，带<arg>的表示后面必须填写一个参数，像`--slave`这样，后面就不带参数，掌握了这个基本规则，就能从help命令中获得redis-trib.rb的使用方法。

其他命令大都需要传递`host:port`，这是redis-trib.rb为了连接集群，需要选择集群中的一个节点，然后通过该节点获得整个集群的信息。

下面就一一详细介绍redis-trib.rb的每个功能。

### create创建集群

create命令可选replicas参数，replicas表示需要有几个slave。最简单命令使用如下：

	$ruby redis-trib.rb create 10.180.157.199:6379 10.180.157.200:6379 10.180.157.201:6379

有一个slave的创建命令如下：

	$ruby redis-trib.rb create --replicas 1 10.180.157.199:6379 10.180.157.200:6379 10.180.157.201:6379 10.180.157.202:6379  10.180.157.205:6379  10.180.157.208:6379 

创建流程如下：

*	1、首先为每个节点创建ClusterNode对象，包括连接每个节点。检查每个节点是否为独立且db为空的节点。执行`load_info`方法导入节点信息。
*	2、检查传入的master节点数量是否大于等于3个。只有大于3个节点才能组成集群。
*	3、计算每个master需要分配的slot数量，以及给master分配slave。分配的算法大致如下：
	*	先把节点按照host分类，这样保证master节点能分配到更多的主机中。
	*	不停遍历遍历host列表，从每个host列表中弹出一个节点，放入interleaved数组。直到所有的节点都弹出为止。
	*	master节点列表就是interleaved前面的master数量的节点列表。保存在masters数组。
	*	计算每个master节点负责的slot数量，保存在slots_per_node对象，用slot总数除以master数量取整即可。
	*	遍历masters数组，每个master分配slots_per_node个slot，最后一个master，分配到16384个slot为止。
	*	接下来为master分配slave，分配算法会尽量保证master和slave节点不在同一台主机上。对于分配完指定slave数量的节点，还有多余的节点，也会为这些节点寻找master。分配算法会遍历两次masters数组。
	*	第一次遍历masters数组，在余下的节点列表找到replicas数量个slave。每个slave为第一个和master节点host不一样的节点，如果没有不一样的节点，则直接取出余下列表的第一个节点。
	*	第二次遍历是在对于节点数除以replicas不为整数，则会多余一部分节点。遍历的方式跟第一次一样，只是第一次会一次性给master分配replicas数量个slave，而第二次遍历只分配一个，直到余下的节点被全部分配出去。
*	4、打印出分配信息，并提示用户输入“yes”确认是否按照打印出来的分配方式创建集群。
*	5、输入“yes”后，会执行flush_nodes_config操作，该操作执行前面的分配结果，给master分配slot，让slave复制master，对于还没有握手（cluster meet）的节点，slave复制操作无法完成，不过没关系，flush_nodes_config操作出现异常会很快返回，后续握手后会再次执行flush_nodes_config。
*	6、给每个节点分配epoch，遍历节点，每个节点分配的epoch比之前节点大1。
*	7、节点间开始相互握手，握手的方式为节点列表的其他节点跟第一个节点握手。
*	8、然后每隔1秒检查一次各个节点是否已经消息同步完成，使用ClusterNode的get_config_signature方法，检查的算法为获取每个节点cluster nodes信息，排序每个节点，组装成`node_id1:slots|node_id2:slot2|...`的字符串。如果每个节点获得字符串都相同，即认为握手成功。
*	9、此后会再执行一次flush_nodes_config，这次主要是为了完成slave复制操作。
*	10、最后再执行check_cluster，全面检查一次集群状态。包括和前面握手时检查一样的方式再检查一遍。确认没有迁移的节点。确认所有的slot都被分配出去了。
*	11、至此完成了整个创建流程，返回`[OK] All 16384 slots covered.`。


### check检查集群

检查集群状态的命令，没有其他参数，只需要选择一个集群中的一个节点即可。执行命令以及结果如下：

	$ruby redis-trib.rb check 10.180.157.199:6379
	>>> Performing Cluster Check (using node 10.180.157.199:6379)
	M: b2506515b38e6bbd3034d540599f4cd2a5279ad1 10.180.157.199:6379
	   slots:0-5460 (5461 slots) master
	   1 additional replica(s)
	S: d376aaf80de0e01dde1f8cd4647d5ac3317a8641 10.180.157.205:6379
	   slots: (0 slots) slave
	   replicates e36c46dbe90960f30861af00786d4c2064e63df2
	M: 15126fb33796c2c26ea89e553418946f7443d5a5 10.180.157.201:6379
	   slots:10923-16383 (5461 slots) master
	   1 additional replica(s)
	S: 59fa6ee455f58a5076f6d6f83ddd74161fd7fb55 10.180.157.208:6379
	   slots: (0 slots) slave
	   replicates 15126fb33796c2c26ea89e553418946f7443d5a5
	S: 460b3a11e296aafb2615043291b7dd98274bb351 10.180.157.202:6379
	   slots: (0 slots) slave
	   replicates b2506515b38e6bbd3034d540599f4cd2a5279ad1
	M: e36c46dbe90960f30861af00786d4c2064e63df2 10.180.157.200:6379
	   slots:5461-10922 (5462 slots) master
	   1 additional replica(s)
	[OK] All nodes agree about slots configuration.
	>>> Check for open slots...
	>>> Check slots coverage...
	[OK] All 16384 slots covered.	

检查前会先执行load_cluster_info_from_node方法，把所有节点数据load进来。load的方式为通过自己的cluster nodes发现其他节点，然后连接每个节点，并加入nodes数组。接着生成节点间的复制关系。

load完数据后，开始检查数据，检查的方式也是调用创建时候使用的check_cluster。

### info查看集群信息

info命令用来查看集群的信息。info命令也是先执行load_cluster_info_from_node获取完整的集群信息。然后显示ClusterNode的info_string结果，示例如下：

	$ruby redis-trib.rb info 10.180.157.199:6379
	10.180.157.199:6379 (b2506515...) -> 0 keys | 5461 slots | 1 slaves.
	10.180.157.201:6379 (15126fb3...) -> 0 keys | 5461 slots | 1 slaves.
	10.180.157.200:6379 (e36c46db...) -> 0 keys | 5462 slots | 1 slaves.
	[OK] 0 keys in 3 masters.
	0.00 keys per slot on average.

### fix修复集群


fix命令的流程跟check的流程很像，显示加载集群信息，然后在check_cluster方法内传入fix为
true的变量，会在集群检查出现异常的时候执行修复流程。目前fix命令能修复两种异常，一种是集群有处于迁移中的slot的节点，一种是slot未完全分配的异常。

fix_open_slot方法是修复集群有处于迁移中的slot的节点异常。

*	1、先检查该slot是谁负责的，迁移的源节点如果没完成迁移，owner还是该节点。没有owner的slot无法完成修复功能。
*	2、遍历每个节点，获取哪些节点标记该slot为migrating状态，哪些节点标记该slot为importing状态。对于owner不是该节点，但是通过`cluster countkeysinslot`获取到该节点有数据的情况，也认为该节点为importing状态。
*	3、如果migrating和importing状态的节点均只有1个，这可能是迁移过程中redis-trib.rb被中断所致，直接执行move_slot继续完成迁移任务即可。传递dots和fix为true。
*	4、如果migrating为空，importing状态的节点大于0，那么这种情况执行回滚流程，将importing状态的节点数据通过move_slot方法导给slot的owner节点，传递dots、fix和cold为true。接着对importing的节点执行`cluster stable`命令恢复稳定。
*	5、如果importing状态的节点为空，有一个migrating状态的节点，而且该节点在当前slot没有数据，那么可以直接把这个slot设为stable。
*	6、如果migrating和importing状态不是上述情况，目前redis-trib.rb工具无法修复，上述的三种情况也已经覆盖了通过redis-trib.rb工具迁移出现异常的各个方面，人为的异常情形太多，很难考虑完全。

fix_slots_coverage方法能修复slot未完全分配的异常。未分配的slot有三种状态。

*	1、所有节点的该slot都没有数据。该状态redis-trib.rb工具直接采用随机分配的方式，并没有考虑节点的均衡。本人尝试对没有分配slot的集群通过fix修复集群，结果slot还是能比较平均的分配，但是没有了连续性，打印的slot信息非常离散。
*	2、有一个节点的该slot有数据。该状态下，直接把slot分配给该slot有数据的节点。
*	3、有多个节点的该slot有数据。此种情况目前还处于TODO状态，不过redis作者列出了修复的步骤，对这些节点，除第一个节点，执行cluster migrating命令，然后把这些节点的数据迁移到第一个节点上。清除migrating状态，然后把slot分配给第一个节点。


### reshard在线迁移slot

reshard命令可以在线把集群的一些slot从集群原来slot负责节点迁移到新的节点，利用reshard可以完成集群的在线横向扩容和缩容。

reshard的参数很多，下面来一一解释一番：

    reshard         host:port
                    --from <arg>
                    --to <arg>
                    --slots <arg>
                    --yes
                    --timeout <arg>
                    --pipeline <arg>
                    
*	`host:port`：这个是必传参数，用来从一个节点获取整个集群信息，相当于获取集群信息的入口。
*	`--from <arg>`：需要从哪些源节点上迁移slot，可从多个源节点完成迁移，以逗号隔开，传递的是节点的node id，还可以直接传递`--from all`，这样源节点就是集群的所有节点，不传递该参数的话，则会在迁移过程中提示用户输入。
*	`--to <arg>`：slot需要迁移的目的节点的node id，目的节点只能填写一个，不传递该参数的话，则会在迁移过程中提示用户输入。
*	`--slots <arg>`：需要迁移的slot数量，不传递该参数的话，则会在迁移过程中提示用户输入。
*	`--yes`：设置该参数，可以在打印执行reshard计划的时候，提示用户输入yes确认后再执行reshard。
*	`--timeout <arg>`：设置migrate命令的超时时间。
*	`--pipeline <arg>`：定义`cluster getkeysinslot`命令一次取出的key数量，不传的话使用默认值为10。

迁移的流程如下：

*	1、通过load_cluster_info_from_node方法装载集群信息。
*	2、执行check_cluster方法检查集群是否健康。只有健康的集群才能进行迁移。
*	3、获取需要迁移的slot数量，用户没传递`--slots`参数，则提示用户手动输入。
*	4、获取迁移的目的节点，用户没传递`--to`参数，则提示用户手动输入。此处会检查目的节点必须为master节点。
*	5、获取迁移的源节点，用户没传递`--from`参数，则提示用户手动输入。此处会检查源节点必须为master节点。`--from all`的话，源节点就是除了目的节点外的全部master节点。这里为了保证集群slot分配的平均，建议传递`--from all`。
*	6、执行compute_reshard_table方法，计算需要迁移的slot数量如何分配到源节点列表，采用的算法是按照节点负责slot数量由多到少排序，计算每个节点需要迁移的slot的方法为：迁移slot数量 * (该源节点负责的slot数量 / 源节点列表负责的slot总数)。这样算出的数量可能不为整数，这里代码用了下面的方式处理：

            n = (numslots/source_tot_slots*s.slots.length)
            if i == 0
                n = n.ceil
            else
                n = n.floor
这样的处理方式会带来最终分配的slot与请求迁移的slot数量不一致，这个BUG已经在github上提给作者，<https://github.com/antirez/redis/issues/2990>。
*	7、打印出reshard计划，如果用户没传`--yes`，就提示用户确认计划。
*	8、根据reshard计划，一个个slot的迁移到新节点上，迁移使用move_slot方法，该方法被很多命令使用，具体可以参见下面的迁移流程。move_slot方法传递dots为true和pipeline数量。
*	9、至此，就完成了全部的迁移任务。

下面看下一次reshard的执行结果：

```
  $ruby redis-trib.rb reshard --from all --to 80b661ecca260c89e3d8ea9b98f77edaeef43dcd --slots 11 10.180.157.199:6379
  >>> Performing Cluster Check (using node 10.180.157.199:6379)
  S: b2506515b38e6bbd3034d540599f4cd2a5279ad1 10.180.157.199:6379
     slots: (0 slots) slave
     replicates 460b3a11e296aafb2615043291b7dd98274bb351
  S: d376aaf80de0e01dde1f8cd4647d5ac3317a8641 10.180.157.205:6379
     slots: (0 slots) slave
     replicates e36c46dbe90960f30861af00786d4c2064e63df2
  M: 15126fb33796c2c26ea89e553418946f7443d5a5 10.180.157.201:6379
     slots:10923-16383 (5461 slots) master
     1 additional replica(s)
  S: 59fa6ee455f58a5076f6d6f83ddd74161fd7fb55 10.180.157.208:6379
     slots: (0 slots) slave
     replicates 15126fb33796c2c26ea89e553418946f7443d5a5
  M: 460b3a11e296aafb2615043291b7dd98274bb351 10.180.157.202:6379
     slots:0-5460 (5461 slots) master
     1 additional replica(s)
  M: 80b661ecca260c89e3d8ea9b98f77edaeef43dcd 10.180.157.200:6380
     slots: (0 slots) master
     0 additional replica(s)
  M: e36c46dbe90960f30861af00786d4c2064e63df2 10.180.157.200:6379
     slots:5461-10922 (5462 slots) master
     1 additional replica(s)
  [OK] All nodes agree about slots configuration.
  >>> Check for open slots...
  >>> Check slots coverage...
  [OK] All 16384 slots covered.

  Ready to move 11 slots.
    Source nodes:
      M: 15126fb33796c2c26ea89e553418946f7443d5a5 10.180.157.201:6379
     slots:10923-16383 (5461 slots) master
     1 additional replica(s)
      M: 460b3a11e296aafb2615043291b7dd98274bb351 10.180.157.202:6379
     slots:0-5460 (5461 slots) master
     1 additional replica(s)
      M: e36c46dbe90960f30861af00786d4c2064e63df2 10.180.157.200:6379
     slots:5461-10922 (5462 slots) master
     1 additional replica(s)
    Destination node:
      M: 80b661ecca260c89e3d8ea9b98f77edaeef43dcd 10.180.157.200:6380
     slots: (0 slots) master
     0 additional replica(s)
    Resharding plan:
      Moving slot 5461 from e36c46dbe90960f30861af00786d4c2064e63df2
      Moving slot 5462 from e36c46dbe90960f30861af00786d4c2064e63df2
      Moving slot 5463 from e36c46dbe90960f30861af00786d4c2064e63df2
      Moving slot 5464 from e36c46dbe90960f30861af00786d4c2064e63df2
      Moving slot 0 from 460b3a11e296aafb2615043291b7dd98274bb351
      Moving slot 1 from 460b3a11e296aafb2615043291b7dd98274bb351
      Moving slot 2 from 460b3a11e296aafb2615043291b7dd98274bb351
      Moving slot 10923 from 15126fb33796c2c26ea89e553418946f7443d5a5
      Moving slot 10924 from 15126fb33796c2c26ea89e553418946f7443d5a5
      Moving slot 10925 from 15126fb33796c2c26ea89e553418946f7443d5a5
  Do you want to proceed with the proposed reshard plan (yes/no)? yes
  Moving slot 5461 from 10.180.157.200:6379 to 10.180.157.200:6380:
  Moving slot 5462 from 10.180.157.200:6379 to 10.180.157.200:6380:
  Moving slot 5463 from 10.180.157.200:6379 to 10.180.157.200:6380:
  Moving slot 5464 from 10.180.157.200:6379 to 10.180.157.200:6380:
  Moving slot 0 from 10.180.157.202:6379 to 10.180.157.200:6380:
  Moving slot 1 from 10.180.157.202:6379 to 10.180.157.200:6380:
  Moving slot 2 from 10.180.157.202:6379 to 10.180.157.200:6380:
  Moving slot 10923 from 10.180.157.201:6379 to 10.180.157.200:6380:
  Moving slot 10924 from 10.180.157.201:6379 to 10.180.157.200:6380:
  Moving slot 10925 from 10.180.157.201:6379 to 10.180.157.200:6380:
```

move_slot方法可以在线将一个slot的全部数据从源节点迁移到目的节点，fix、reshard、rebalance都需要调用该方法迁移slot。

move_slot接受下面几个参数，
*	1、`pipeline`：设置一次从slot上获取多少个key。
*	2、`quiet`：迁移会打印相关信息，设置quiet参数，可以不用打印这些信息。
*	3、`cold`：设置cold，会忽略执行importing和migrating。
*	4、`dots`：设置dots，则会在迁移过程打印迁移key数量的进度。
*	5、`update`：设置update，则会更新内存信息，方便以后的操作。

move_slot流程如下：

*	1、如果没有设置cold，则对源节点执行`cluster importing`命令，对目的节点执行`migrating`命令。fix的时候有可能importing和migrating已经执行过来，所以此种场景会设置cold。
*	2、通过`cluster getkeysinslot`命令，一次性获取远节点迁移slot的pipeline个key的数量.
*	3、对这些key执行migrate命令，将数据从源节点迁移到目的节点。
*	4、如果migrate出现异常，在fix模式下，BUSYKEY的异常，会使用migrate的replace模式再执行一次，BUSYKEY表示目的节点已经有该key了，replace模式可以强制替换目的节点的key。不是fix模式就直接返回错误了。
*	5、循环执行`cluster getkeysinslot`命令，直到返回的key数量为0，就退出循环。
*	6、如果没有设置cold，对每个节点执行`cluster setslot`命令，把slot赋给目的节点。
*	7、如果设置update，则修改源节点和目的节点的slot信息。
*	8、至此完成了迁移slot的流程。

### rebalance平衡集群节点slot数量

rebalance命令可以根据用户传入的参数平衡集群节点的slot数量，rebalance功能非常强大，可以传入的参数很多，以下是rebalance的参数列表和命令示例。

    rebalance       host:port
                    --weight <arg>
                    --auto-weights
                    --threshold <arg>
                    --use-empty-masters
                    --timeout <arg>
                    --simulate
                    --pipeline <arg>
                    
	$ruby redis-trib.rb rebalance --threshold 1 --weight b31e3a2e=5 --weight 60b8e3a1=5 --use-empty-masters  --simulate 10.180.157.199:6379

下面也先一一解释下每个参数的用法：

*	`host:port`：这个是必传参数，用来从一个节点获取整个集群信息，相当于获取集群信息的入口。
*	`--weight <arg>`：节点的权重，格式为node_id=weight，如果需要为多个节点分配权重的话，需要添加多个`--weight <arg>`参数，即`--weight b31e3a2e=5 --weight 60b8e3a1=5`，node_id可为节点名称的前缀，只要保证前缀位数能唯一区分该节点即可。没有传递--weight的节点的权重默认为1。
*	`--auto-weights`：这个参数在rebalance流程中并未用到。
*	`--threshold <arg>`：只有节点需要迁移的slot阈值超过threshold，才会执行rebalance操作。具体计算方法可以参考下面的`rebalance命令流程`的第四步。
*	`--use-empty-masters`：rebalance是否考虑没有节点的master，默认没有分配slot节点的master是不参与rebalance的，设置`--use-empty-masters`可以让没有分配slot的节点参与rebalance。
*	`--timeout <arg>`：设置migrate命令的超时时间。
*	`--simulate`：设置该参数，可以模拟rebalance操作，提示用户会迁移哪些slots，而不会真正执行迁移操作。
*	`--pipeline <arg>`：与reshar的pipeline参数一样，定义`cluster getkeysinslot`命令一次取出的key数量，不传的话使用默认值为10。

rebalance命令流程如下：

*	1、load_cluster_info_from_node方法先加载集群信息。
*	2、计算每个master的权重，根据参数`--weight <arg>`，为每个设置的节点分配权重，没有设置的节点，则权重默认为1。
*	3、根据每个master的权重，以及总的权重，计算自己期望被分配多少个slot。计算的方式为：总slot数量 * （自己的权重 / 总权重）。
*	4、计算每个master期望分配的slot是否超过设置的阈值，即`--threshold <arg>`设置的阈值或者默认的阈值。计算的方式为：先计算期望移动节点的阈值，算法为：(100-(100.0*expected/n.slots.length)).abs，如果计算出的阈值没有超出设置阈值，则不需要为该节点移动slot。只要有一个master的移动节点超过阈值，就会触发rebalance操作。
*	5、如果触发了rebalance操作。那么就开始执行rebalance操作，先将每个节点当前分配的slots数量减去期望分配的slot数量获得balance值。将每个节点的balance从小到大进行排序获得sn数组。
*	6、用dst_idx和src_idx游标分别从sn数组的头部和尾部开始遍历。目的是为了把尾部节点的slot分配给头部节点。

	>sn数组保存的balance列表排序后，负数在前面，正数在后面。负数表示需要有slot迁入，所以使用dst_idx游标，正数表示需要有slot迁出，所以使用src_idx游标。理论上sn数组各节点的balance值加起来应该为0，不过由于在计算期望分配的slot的时候只是使用直接取整的方式，所以可能出现balance值之和不为0的情况，balance值之和不为0即为节点不平衡的slot数量，由于slot总数有16384个，不平衡数量相对于总数，基数很小，所以对rebalance流程影响不大。


*	7、获取sn[dst_idx]和sn[src_idx]的balance值较小的那个值，该值即为需要从sn[src_idx]节点迁移到sn[dst_idx]节点的slot数量。
*	8、接着通过compute_reshard_table方法计算源节点的slot如何分配到源节点列表。这个方法在reshard流程中也有调用，具体步骤可以参考reshard流程的第六步。
*	9、如果是simulate模式，则只是打印出迁移列表。
*	10、如果没有设置simulate，则执行move_slot操作，迁移slot，传入的参数为:quiet=>true,:dots=>false,:update=>true。
*	11、迁移完成后更新sn[dst_idx]和sn[src_idx]的balance值。如果balance值为0后，游标向前进1。
*	12、直到dst_idx到达src_idx游标，完成整个rebalance操作。

### add-node将新节点加入集群

add-node命令可以将新节点加入集群，节点可以为master，也可以为某个master节点的slave。

	add-node	new_host:new_port existing_host:existing_port
              --slave
              --master-id <arg>
                    
add-node有两个可选参数：

*	`--slave`：设置该参数，则新节点以slave的角色加入集群
*	`--master-id`：这个参数需要设置了`--slave`才能生效，`--master-id`用来指定新节点的master节点。如果不设置该参数，则会随机为节点选择master节点。

可以看下add-node命令的执行示例：

	$ruby redis-trib.rb add-node --slave --master-id dcb792b3e85726f012e83061bf237072dfc45f99 10.180.157.202:6379 10.180.157.199:6379
	>>> Adding node 10.180.157.202:6379 to cluster 10.180.157.199:6379
	>>> Performing Cluster Check (using node 10.180.157.199:6379)
	M: dcb792b3e85726f012e83061bf237072dfc45f99 10.180.157.199:6379
	   slots:0-5460 (5461 slots) master
	   0 additional replica(s)
	M: 464d740bf48953ebcf826f4113c86f9db3a9baf3 10.180.157.201:6379
	   slots:10923-16383 (5461 slots) master
	   0 additional replica(s)
	M: befa7e17b4e5f239e519bc74bfef3264a40f96ae 10.180.157.200:6379
	   slots:5461-10922 (5462 slots) master
	   0 additional replica(s)
	[OK] All nodes agree about slots configuration.
	>>> Check for open slots...
	>>> Check slots coverage...
	[OK] All 16384 slots covered.
	>>> Send CLUSTER MEET to node 10.180.157.202:6379 to make it join the cluster.
	Waiting for the cluster to join.
	>>> Configure node as replica of 10.180.157.199:6379.
	[OK] New node added correctly.
	
add-node流程如下：

*	1、通过load_cluster_info_from_node方法转载集群信息，check_cluster方法检查集群是否健康。
*	2、如果设置了`--slave`，则需要为该节点寻找master节点。设置了`--master-id`，则以该节点作为新节点的master，如果没有设置`--master-id`，则调用get_master_with_least_replicas方法，寻找slave数量最少的master节点。如果slave数量一致，则选取load_cluster_info_from_node顺序发现的第一个节点。load_cluster_info_from_node顺序的第一个节点是add-node设置的existing_host:existing_port节点，后面的顺序根据在该节点执行`cluster nodes`返回的结果返回的节点顺序。
*	3、连接新的节点并与集群第一个节点握手。
*	4、如果没设置--slave就直接返回ok，设置了--slave，则需要等待确认新节点加入集群，然后执行`cluster replicate`命令复制master节点。
*	5、至此，完成了全部的增加节点的流程。

### del-node从集群中删除节点

del-node可以把某个节点从集群中删除。del-node只能删除没有分配slot的节点。删除命令传递两个参数：

*	`host:port`：从该节点获取集群信息。
*	`node_id`：需要删除的节点id。

del-node执行结果示例如下：

	$ruby redis-trib.rb del-node 10.180.157.199:6379 d5f6d1d17426bd564a6e309f32d0f5b96962fe53
	>>> Removing node d5f6d1d17426bd564a6e309f32d0f5b96962fe53 from cluster 10.180.157.199:6379
	>>> Sending CLUSTER FORGET messages to the cluster...
	>>> SHUTDOWN the node.

del-node流程如下：

*	1、通过load_cluster_info_from_node方法转载集群信息。
*	2、根据传入的node id获取节点，如果节点没找到，则直接提示错误并退出。
*	3、如果节点分配的slot不为空，则直接提示错误并退出。
*	4、遍历集群内的其他节点，执行`cluster forget`命令，从每个节点中去除该节点。如果删除的节点是master，而且它有slave的话，这些slave会去复制其他master，调用的方法是get_master_with_least_replicas，与add-node没设置`--master-id`寻找master的方法一样。
*	5、然后关闭该节点。

### set-timeout设置集群节点间心跳连接的超时时间

set-timeout用来设置集群节点间心跳连接的超时时间，单位是毫秒，不得小于100毫秒，因为100毫秒对于心跳时间来说太短了。该命令修改是节点配置参数`cluster-node-timeout`，默认是15000毫秒。通过该命令，可以给每个节点设置超时时间，设置的方式使用`config set`命令动态设置，然后执行`config rewrite`命令将配置持久化保存到硬盘。以下是示例：

	ruby redis-trib.rb set-timeout 10.180.157.199:6379 30000
	>>> Reconfiguring node timeout in every cluster node...
	*** New timeout set for 10.180.157.199:6379
	*** New timeout set for 10.180.157.205:6379
	*** New timeout set for 10.180.157.201:6379
	*** New timeout set for 10.180.157.200:6379
	*** New timeout set for 10.180.157.208:6379
	>>> New node timeout set. 5 OK, 0 ERR.
	
### call在集群全部节点上执行命令

call命令可以用来在集群的全部节点执行相同的命令。call命令也是需要通过集群的一个节点地址，连上整个集群，然后在集群的每个节点执行该命令。

	$ruby redis-trib.rb call 10.180.157.199:6379 get key
	>>> Calling GET key
	10.180.157.199:6379: MOVED 12539 10.180.157.201:6379
	10.180.157.205:6379: MOVED 12539 10.180.157.201:6379
	10.180.157.201:6379:
	10.180.157.200:6379: MOVED 12539 10.180.157.201:6379
	10.180.157.208:6379: MOVED 12539 10.180.157.201:6379

### import将外部redis数据导入集群

import命令可以把外部的redis节点数据导入集群。导入的流程如下：

*	1、通过load_cluster_info_from_node方法转载集群信息，check_cluster方法检查集群是否健康。
*	2、连接外部redis节点，如果外部节点开启了cluster_enabled，则提示错误。
*	3、通过scan命令遍历外部节点，一次获取1000条数据。
*	4、遍历这些key，计算出key对应的slot。
*	5、执行migrate命令,源节点是外部节点,目的节点是集群slot对应的节点，如果设置了`--copy`参数，则传递copy参数，如果设置了`--replace`，则传递replace参数。
*	6、不停执行scan命令，直到遍历完全部的key。
*	7、至此完成整个迁移流程

这中间如果出现异常，程序就会停止。没使用`--copy`模式，则可以重新执行import命令，使用`--copy`的话，最好清空新的集群再导入一次。

import命令更适合离线的把外部redis数据导入，在线导入的话最好使用更专业的导入工具，以slave的方式连接redis节点去同步节点数据应该是更好的方式。
