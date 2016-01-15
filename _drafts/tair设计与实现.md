NKV项目全称Netease Key Value，是网易高性能，分布式，可扩展，高可靠的Key/Value结构存储系统，是基于淘宝开源系统tair的二次开发。tair本身已经比较完善，经历过淘宝双十一的考验，我们在此基础上增加了一些额外的接口，增加了Namespace映射功能，完善、修复了tair双副本等众多功能、BUG。最近有很多项目组想使用NKV服务，对于程序员，都不希望使用一个黑盒服务，他们会咨询很多关于NKV设计的问题，在此我总结了NKV的主要特性，对这些特性的设计和实现做些说明。

以下是NKV的架构图：

 ![Alt pic](http://nos.netease.com/knowledge/e587f83e-9c11-4d13-a900-d51fdbc05f08) 

NKV服务端由configserver和dataserver组成，使用c++编写的，网络库和系统库使用了淘宝开源的[tbnet和tbsys](http://code.taobao.org/svn/tb-common-utils/trunk/)。configserver是管理服务，管理则数据集群，能根据集群节点的存活状态动态的增加和删除节点，实现了NKV系统的自动运维，configserver有Master和Slave保证了高可用。dataserver集群是数据存储服务，包含了mdb（memcached），rdb（redis）等存储引擎，dataserver如果有多副本的话，每个副本保存在不同的dataserver上来保证数据高可靠。NKV客户端SDK目前有java版本的可供使用，用Netty作为网络通信框架。

下面先来看下NKV一次get操作的全过程。

 ![Alt pic](http://nos.netease.com/knowledge/0043f92e-5f3f-4ae5-835e-680c4c0c1e35) 

*	1、启动configserver和dataserver，configserver会生成一张数据分布表，分布表bucket（桶）大小一般为1024，然后这1024个bucket会平均分配给所有的dataserver（假设这里有10个dataserver，那么每个dataserver大概被分配了100个bucket。bucket分配算法，NKV有2种分配bucket的策略，负载均衡策略和位置安全策略。我们线上用的是负载均衡策略策略）。
*	2、客户端首先连上configserver获取数据分布表。
*	3、计算key的hash所在的bucket。
*	4、根据所在bucket从数据分布表上获得key所在的dataserver。
*	5、连上该dataserver获取key对应的value。


<br>
这里面有几点需要注意。

*	1、数据分布表带版本，客户端SDK在第一次从configserver拿到数据分布表后就会缓存在内存中，以后每次访问dataserver就会直接从缓存的数据分布表计算bucket所在的dataserver。数据分布表会有个client_version，客户端会一并缓存，每次操作从dataserver获得数据的时候，dataserver都会带上最新的数据分布表client_version。如果客户端发现缓存的client_version小于服务端最新的数据分布表client_version，就会去configserver更新数据分布表。更新是异步完成的，在更新过程中数据访问到老节点写操作会抛出WRITE_NOT_ON_MASTER的异常，所以出现这种异常不妨重试一下，等数据分布表更新完成就能正常写数据了。
*	2、流控机制，NKV有一套针对每个Namespace的流量控制机制，当一个Namespace请求qps过大或者流量过大的时候，就会触发服务端的流控阀值，是否触发阀值是由每台dataserver自己控制（目前NKV dataserver设置的阀值为：当本节点qps超过3w且所有节点总qps超过5w，或者本节点流量超过30MB/s且所有节点总流量超过75MB/s。当本节点qps低于2w或者所有节点总qps低于4w，或者本节点流量低于15MB/s或者所有节点总流量低于65MB/s，流控取消。总的qps和流量很容易达到上限，但以我们目前mdb集群的节点数30个计算，假设dataserver流量均等考虑的话，一个节点超过流量3w需要总流量超过90w才会触发阀值）。阀值被触发后服务端就会发送阀值UP包给客户端。客户端阀值处理的设计非常巧妙，它并不是一下就把所有流量挡住，而是有一套流控的算法。阀值有个threshold，客户端定义最大的threshold，MAX_THRESHOLD为1000。每次请求是否被流控会判断：
flowRandom.nextInt(MAX_THRESHOLD) < threshold，也就是说会有threshold/MAX_THRESHOLD的请求被流控。检查代码如下：


          public boolean isOverflow() {
               if (threshold == 0)
                    return false;
               else if (threshold >= MAX_THRESHOLD) {
                    log.warn("Threshold " + threshold + " larger than max " + MAX_THRESHOLD);
                    return true;
               } 
               else
               {
                    //这么做的好处是一旦被判定为over flow，不会限制所有的该namespace的request
                    //而是以一定的概率限制request
                    return flowRandom.nextInt(MAX_THRESHOLD) < threshold;
               }
                
          }

	threshold的增长与减少也是一个逐步的过程。客户端每次请求都有一个checkLevelDown的过程，checkLevelDown判断上次check时间，超过DOWN_CHECKTIME（5s）就去服务器请求流控状态。代码如下：
     
          public boolean limitLevelCheck(NkvRpcContext ctx, NkvRpcPacketFactory factory, NkvChannel channel, short ns) throws NkvRpcError {
               if (threshold == 0) {
                    return false;
               }

               long now = System.currentTimeMillis();
               if (now - lastTime < DOWN_CHECKTIME || now - checkTime < DOWN_CHECKTIME) {
                    return false;
               }
               synchronized (this) {
                    //刚被别的线程更新lastTime或者刚发出check请求
                    if (now - lastTime < DOWN_CHECKTIME || now - checkTime < DOWN_CHECKTIME) {
                         return true;
                    }
                    checkTime = now;
               }
               log.warn("flow limit check ns " + namespace + " curt " + threshold);
               TrafficCheckRequest request = TrafficCheckRequest.build(ns);
               ctx.callAsync(channel, request, TRAFFIC_CHECK_REQUEST_TIMEOUT, factory);
               return true;
          }

     阀值被触发后，dataserver每次响应客户端数据请求会多发一个阀值UP包，上面limitLevelCheck流程中，客户端也会在limitLevelCheck自己去请求阀值状态，如果为UP，离上次更新threshold超过UP_CHECKTIME（10s），就会更新threshold为threshold + UP_FACTOR * (MAX_THRESHOLD - threshold)，UP_FACTOR默认为0.3。代码如下：
     
          public boolean limitLevelUp() {
               long now = System.currentTimeMillis();
               if (now - lastTime < UP_CHECKTIME) {
                    return false;
               }
               synchronized (this) {
                    if (now - lastTime < UP_CHECKTIME) {
                         return true;
                    }
                    //逐步提高被判断为over flow的概率
                    if (threshold < MAX_THRESHOLD - 10)
                         threshold = (int)(threshold + UP_FACTOR * (MAX_THRESHOLD - threshold));
                    lastTime = now;
                    log.warn("flow limit up ns " + namespace + " curt " + threshold);
               }
               return true;
          }

	如果为DOWN，离上次更新threshold超过DOWN_CHECKTIME（5s），就会更新threshold为threshold - DOWN_FACTOR * threshold，DOWN_FACTOR默认为0.5。代码如下：

          public boolean limitLevelDown() {
               long now = System.currentTimeMillis();
               if (now - lastTime < DOWN_CHECKTIME) {
                    return false;
               }
               synchronized (this) {
                    //在上一个if之后，lasttime被别的线程所更新
                    if (now - lastTime < DOWN_CHECKTIME) {
                         return true;
                    }
                    //这样做的好处是一旦前期被限流，不会因为一个down而导致全被解流，它是一个逐步解流的过程
                    //一旦threshold阀值低于50，则全解流
                    threshold = (int)(threshold - DOWN_FACTOR * threshold);
                    if (threshold < 50)
                         threshold = 0;
                    lastTime = now;
                    log.warn("flow limit down ns " + namespace + " curt " + threshold);
               }
               return true;
          }

*	3、并发控制，memcached和redis如果多线程或者多个进程对同一个key操作会出现并发问题，例如get出该key不存在后再set一个value的话，中间可能有其他进程对该key也set了一个value2，这样value就把value2覆盖了。NKV引入了version的概念，每个key都有一个version，每次对一个key的操作都会返回该key的version，如果两个进程都用version+1的version去set该key的话，后面一个set操作就会抛出VERSION_ERROR的异常。进而可以更新最新的version再用version+1去set。如果不想使用version也很简单，version参数传0就能忽略version强制更新。
*	4、Namespace映射，NKV开始对每个应用分配空间的时候，使用Namespace作为标识，Namespace使用的是一个1024内的short类型，不同应用之间很容易出现Namespace干扰，比如A应用分配的Namespace为1，B应用分配的Namespace为2，A在SDK调用NKV服务的时候，将Namespace写成2的话，就使用了B应用的空间，出现了应用间空间干扰的问题。NKV为解决这个问题原先计划开发权限检查的功能，后来经过内部讨论，设计了一套Namespace映射的策略，Namespace被干扰源于连续数字容易被误写，如果Namespace为String，每个应用申请自己标识的String，那么不同应用就不会误操作其他应用标识符了。Namespace映射在客户端实现，客户端在获取数据分布表的时候也会获取一份Namespace映射表，然后缓存在内存里，每次传入String的标识符会根据Namespace映射表转换成对应的整型Namespace，性能几乎毫无损失。用老版本SDK访问最新的Namespace映射版本的NKV服务（不解析Namespace映射表）和用最新的Namespace映射版本的SDK访问老版本的NKV服务（判断没有Namespace映射数据就不解析，但只能使用short的Namespace）均能兼容。


configserver和dataserver的设计和实现
----

configserver
---

configserver在启动的时候会开启一个server_conf_thread线程，该线程首先加载集群的配置信息（load_group_file），configserver可以管理多个dataserver集群（group），读取配置信息会判断每个集群是否已经有持久化配置信息，该持久化配置信息保存了数据分布表等信息。不存在就会创建持久化配置，存在则更新持久化配置。所以重启configserver是不会重建数据分布表的。这里需要注意的是，load_group_file的过程如果发现有对客户端数据分布表产生影响的配置发生变更就会增加client_version，如果有对dataserver产生影响的配置发生变更就会增加server_version（server_version是configserver发给dataserver的数据分布表，后面会讲到它与client_version不同的原因）。

然后server_conf_thread线程就进入了while(!_stop) 的循环运行状态，该循环每次执行以下操作就sleep 1s。首先，检查集群的配置文件是否发生更新，如果更新则重新加载集群的配置信息（load_group_file），接着会给另一个configserver发生心跳包，然后会检查另一个configserver和所有dataserver的存活状态是否发生变更，最后会有一个rotateLog的操作，每过一天备份老的日志文件，重新生成一个日志文件以供使用。

检查dataserver存活状态的方式是这样的，每个dataserver会定期给configserver发送心跳，首先configserver会更新该dataserver的心跳时间last_time，接着如果发现原先dataserver的状态是down，这里会有2种策略处理，一种是自动检测策略GROUP_AUTO_ACCEPT_STRATEGY，此策略会直接把dataserver状态改成alive并设置需要更新数据分布表标识need_rebuild_hash_table为true。另一种是需要手动处理的策略GROUP_DEFAULT_ACCEPT_STRATEGY，此策略更新dataserver状态延迟到集群配置文件修改时间发生改变才会由load_group_file更新成alive。（另外再说下dataserver的心跳包会带上dataserver的统计信息以供configserver汇总，心跳包还有server_version、migrate_version、plugins_version、area_capacity_version参数，server_version是dataserver保存的数据分布表版本，如果configserver当前的server_version大于心跳包的server_version，说明数据分布表已经更新，在响应包就会带上最新的数据分布表。migrate_version后面讲迁移的时候再说。plugins_version是dataserver插件版本，如果更新了，响应包就会带上最新的插件名称，插件是dataserver的功能，这个后面说。area_capacity_version是namespace空间配额的信息的版本，如果发生变化，响应包会带上最新的配额信息。）前面说了server_conf_thread会循环检查每个dataserver的存活，检查的方式是这样的，首先检查每个集群的need_rebuild_hash_table是否为true，true的话就会引发该集群数据分布表重分布操作，接着遍历所有活着的和强制关闭的dataserver（down的dataserver无需检查，down的dataserver如果活过来会给configserver发心跳，那里会处理down的dataserver），如果发现有dataserver的心跳超时并且connect过去也失败了，或者是强制关闭状态的话，就会把dataserver的状态更新为down，同时引发该集群数据分布表重分布操作。数据分布表重分布操作单独有个table_builder_thread线程在定时遍历group_need_build vector，对group_need_build vector里的每个集群进行重分布操作。前面引发数据分布表重分布操作就是往group_need_build vector里面push集群。

configserver启动的时候会开启监听端口，接收另一个configserver、dataserver和客户端发送的命令。configserver接收了如下命令：
     
-	REQ_GET_GROUP_PACKET：客户端请求数据分布表。
-	REQ_HEARTBEAT_PACKET：接收dataserver心跳包。
-	REQ_QUERY_INFO_PACKET：客户端请求查询一些info信息，SDK接口对应的是getStat。
-	REQ_CONFHB_PACKET：接收另一个configserver心跳包。
-	REQ_SETMASTER_PACKET：给另一个configserver发送此命令，以决定谁是master。
-	REQ_GET_SVRTAB_PACKET：用于slave configserver向master configserver请求集群配置信息。
-	RESP_GET_SVRTAB_PACKET：用于在master configserver的集群配置发送改变时主动发送集群配置信息给slave。
-	REQ_MIG_FINISH_PACKET：这个是发送数据迁移时，dataserver一个bucket迁移完成后发送的迁移完成命令给configserver，关于迁移后面回详细介绍整个迁移流程。
-	REQ_GROUP_NAMES_PACKET：客户端请求查询dataserver集群（group）列表。
-	REQ_DATASERVER_CTRL_PACKET：客户端控制dataserver的请求包，该请求可以强制关闭一个dataserver节点和由强制关闭状态恢复正常。这个请求SDK接口中没有，需要通过NKV管理工具才能操作。
-	REQ_GET_MIGRATE_MACHINE_PACKET：这个请求SDK接口中也没有，这是用来查看真正迁移的dataserver的请求。
-	REQ_OP_CMD_PACKET：这个请求SDK接口中没有，这个请求貌似是不想再为每个请求都创建一个新的请求包，而是做成一个通用的cmd请求包，可以执行指定的命令。

     
     
前面一直没说到configserver的主从处理方式。configserver的slave是一个热备，它除了不能做决策之外，会提供一切查询功能。也会和每个dataserver保持心跳连接。这样在master宕机后，slave能马上提供configserver服务。一个configserver启动后会给另一个configserver发送REQ_SETMASTER_PACKET请求，另一个configserver用来确定自己是否要变更master/slave角色。然后把结果告知本configserver，来确定本configserver的master/slave角色。然后在运行过程中会不断去check对方是否存活，如果slave检查master宕了，则自己切换到master状态。这里会出现分布式系统常见的脑裂问题。就是如果出现网络分区等网络问题，导致master、slave间网络不通，而他们跟dataserver的连接正常的话，那么slave会误以为master宕机而充当master角色，导致存在双主问题。configserver目前无法避免该问题，所以我们把两台configserver部署在同一机架的2台物理机上来避免此类问题。以后我们也可能会考虑使用其他主从检测方式。
     
dataserver
---     
     
dataserver包含了以下几个模块：
     
*	1、数据请求模块request_processor：dataserver的请求转发模块，该模块会先对请求的数据做一些简单校验，比如key和data的size是否为0。然后在请求的前后会有do_request_plugins和do_response_plugins，它可以在请求前后对数据进行一些封装解析等插件化的操作。request_processor的请求会交给一个manager的类处理。这里会对数据进行详细的校验，包括key、data、namespace的size是否合法等，对于写操作，会判断key是否应该写入该节点，如果是多副本还需要调用复制模块发送复制数据，对于正在迁移的dataserver还需要记录迁移日志，然后还会记录一些监控信息，最后才是调用底层引擎处理请求。这里看似request_processor和manager应该是可以合并处理的，我个人猜测，淘宝当初用了2层，可能是后面为了开发插件功能，在不影响原来manager模块的情况下，直接增加了一层request_processor模块。
*	2、复制模块duplicate_sender_manager（异步复制）、dup_sync_sender_manager（同步复制）：在每次写操作的时候，主副本写成功后会把写操作复制给其他从副本，dup_sync_sender_manager会直接发送复制数据，duplicate_sender_manager只是把复制请求加入发送队列，由异步线程从队列中取出复制数据发送给从副本。由于同步、异步复制发送完数据并没有同步等待返回结果，所以不属于强一致性，只能属于最终一致性。不过由于客户端只能操作主副本，所以正常情况下是不会出现不一致现象的。
*	3、心跳模块heartbeat_thread：heartbeat_thread每秒给configserver发送心跳包，然后异步处理接收的心跳响应包，响应包会根据server_version、migrate_version、plugins_version、area_capacity_version做响应的处理。
*	4、迁移模块migrate_manager。
*	5、监控请求模块stat_processor：用来处理流控的相关请求。

数据迁移设计
----

NKV支持数据的在线横向扩容、缩容，mdb双副本保证增加、减少一个节点数据是不会丢失的，而且迁移过程几乎实时可用。在扩容、缩容的时候会发生数据迁移，下面就详细说说数据迁移的整个设计。

configserver的数据分布表实际上有三份，hashtable、m_hashtable、d_hashtable。hashtable就是发给客户端的hashtable，hashtable和d_hashtable是发给dataserver的数据分布表，因为发的数据分布表不一样，所以这里用了不同的version。在正常情况下这三份表内容是一样的，只有在数据迁移的时候会使用到m_hashtable、d_hashtable，m_hashtable可以理解为数据迁移的过渡状态，d_hashtable是数据迁移完成最终达到的状态，所以整个迁移的过程就是把hashtable中和d_hashtable不一样的bucket替换成d_hashtable中对应的bucket。m_hashtable原先设计是先把希望保存逐步把hashtable中和d_hashtable不一样的bucket替换成d_hashtable中对应的bucket的中间状态，等到m_hashtable和d_hashtable达到一致的时候再一次性替换hashtable，后面策略已经改成了每次替换m_hashtable一个bucket的时候也同时把hashtable中的该bucket替换掉。这样m_hashtable和hashtable已经重复了。
     
在configserver检测到有dataserver宕机或者有新的dataserver加入的时候就会触发数据迁移流程。configserver会先进行rebuild过程，即生成临时数据表和最终数据表。首先，configserver会进行根据m_hashtable构建一张临时数据表。对于有dataserver宕机的情况，临时数据表把从副本提为主副本供客户端访问，对于新加入dataserver的情况，临时数据表还是原来的m_hashtable，然后把临时数据表赋给hashtable，给客户端访问。接着会进行重建数据分布表，根据原来的m_hashtable和目前dataserver的数量，重建一份最终数据迁移完成的数据分布表，即d_hashtable。最后增加client_version和server_version来让客户端和dataserver更新数据分布表。

dataserver在接收到configserver心跳响应包发现server_version不一致的时候，就会读取最新的数据分布表，然后启动一个server_table_updater线程，进行数据分布表的更新操作。首先把表更新操作锁住，防止又一次数据更新进入。在发生数据重分布和重分布完成会进入该更新流程。数据分布表的更新流程如下：
     
*	1、清除旧的存活dataserver，然后遍历hashtable的每个bucket，生成一个新的dataserver集合available_server_ids。
*	2、遍历hashtable每个bucket的同时会判断该bucket是不是该dataserver负责的，是的话就加入temp_holding_buckets集合。dataserver会保存一个holding_buckets集合，保存的是上次更新的temp_holding_buckets集合。比较holding_buckets和temp_holding_buckets，在holding_buckets而不在temp_holding_buckets的bucket就会加入release_buckets（release_buckets会在后面有释放逻辑）集合。更新流程在开始数据迁移的时候temp_holding_buckets和holding_buckets是一样的，发生在数据重分布完成，temp_holding_buckets集合即存储的是最新该dataserver负责的bucket，而holding_buckets存储的是迁移之前dataserver负责的bucket。所以holding_buckets有的而已经不在temp_holding_buckets中的节点需要被释放，即加入release_buckets集合。

*	3、NKV的数据迁移都是从hashtable的主副本迁移数据的，遍历hashtable每个该dataserver负责的主副本bucket，然后比较该bucket在hashtable和d_hashtable上的每一个副本。如果d_hashtable有不在hashtable中的副本，该副本就需要将hashtable的主副本数据迁移过来。需要迁移的节点放在迁移map（migrates）中。计算迁移的代码如下：
   
            void table_manager::calculate_migrates(uint64_t *table, int index)
            {
               uint64_t* psource_table = table;
               uint64_t* pdest_table = table + get_hash_table_size();
               //计算该bucket_no对应的数据是否需要迁移，
               //即计算m_hash_table和d_hash_table中负责该bucket的所有data_server_id是否一致(顺序可以不同)
               for (size_t i = 0; i < this->copy_count; ++i) {

                  bool need_migrate = true;
                  uint64_t dest_dataserver = pdest_table[index + this->bucket_count * i];

                  for (size_t j =0; j < this->copy_count; ++j) 
                  {
                     if (dest_dataserver == psource_table[index + this->bucket_count * j]) {
                        need_migrate = false;
                        break;
                     }
                  }

                  if (need_migrate) {
                     log_debug("add migrate item: bucket[%d] => %s", index, tbsys::CNetUtil::addrToString(dest_dataserver).c_str());
                     migrates[index].push_back(dest_dataserver);
                  }
               }
            }

	这里我画几个图说明一下。

 ![Alt pic](http://nos.netease.com/knowledge/3f1fd989-d002-4556-a55a-b80a4e2edaa7) 

图1，对于bucket 0，d_hashtable中ds0在hashtable中而ds2不在，所以hashtable的主节点ds0需要把数据迁移到ds2.

 ![Alt pic](http://nos.netease.com/knowledge/8d9eed39-80c1-453e-bfa6-f8672332fa69) 

图2，对已bucket 1，d_hashtable中ds2、ds3都不在hashtable中，所以hashtable的主节点ds0需要把数据迁移到ds2、ds3.

*	4、遍历d_hashtable的每个bucket，把该dataserver负责的bucket加入available_server_ids。
*	5、遍历d_hashtable每个bucket的同时会判断该bucket是不是该dataserver负责的，是的话就加入padding_buckets集合。


数据分布表更新完成后会接下来完成一些流程：

*	1、关闭release_buckets集合（该dataserver已经不在负责的bucket）上的bucket。关闭逻辑由每个存储引擎自己处理，mdb会锁住内存，遍历全部数据，把关闭的bucket集合上的数据删除，遍历时也会把mdb中过期的数据也清一遍。
*	2、分别遍历holding_buckets和padding_buckets集合，调用引擎的init_buckets方法由引擎自己完成初始化操作。holding_buckets和padding_buckets感觉应该是有重复之嫌，不过mdb这里直接返回了true，也没做任何处理。
*	3、中断当前的迁移任务。这个是防止在迁移过程中又发生扩容、缩容做的处理。
*	4、把前面计算的迁移map（migrates）加入迁移模块，由迁移模块完成迁移任务。
*	5、如果集群使用的是异步复制，锁住复制队列，对于复制队列里要复制的数据如果复制去的dataserver不在available_server_ids（存活的dataserver集合）中，就把复制的数据清除。

接下来看看迁移模块是如何工作的。

迁移模块是在migrate_manager线程中运行的，migrate_manager线程有个while (true)的run方法。该方法会先等待条件变量（condition.wait）触发迁移流程，前面步骤四把迁移map加入migrate_manager的时候会激活条件变量（condition.signal）。然后run方法会开始处理迁移任务。这样处理的好处就是只有在有迁移任务的时候才开始处理，其他时候都sleep着。迁移任务会先把迁移map拷贝出一份temp_servers，然后遍历temp_servers。对于每个bucket，迁移工作如下：

*	1、获得迁移日志的起始坐标，这样在迁移过程中的写请求记录的迁移日志，接下来都能同步到迁移目标。
*	2、记录当前迁移的对象是该bucket。
*	3、遍历存储引擎的所有在该bucket上的数据，组装好数据发送给所有的迁移对象。这里数据不是一条条发送的，而是数据量每次超过MAX_MUPDATE_PACKET_SIZE(16 * 1024)才发送一个包。
*	4、遍历迁移日志，组装好数据发送给所有的迁移对象，当日志遍历到小于MIGRATE_LOCK_LOG_LEN(1 * 1024 * 1024)，会锁住迁移日志（在锁定期间产生的写请求会提示迁移的错误，反应在客户端SDK会抛出MIGRATE_BUSY的异常），然后迁移最后的日志。迁移完成再等待最后的复制任务完成。接着把该bucket迁移完成的请求包发送给configserver，并同步等待接受到configserver接收到正常的响应包，如果响应包异常则会不停的重试直到正常。接着会把完成的bucket加入migrated_buckets集合，这个目前没啥用，接着会标识下该bucket已经迁移完成，这样再有请求过来，会从d_hashtable上判断该节点是否是主节点，如果不是的话就会返回WRITE_NOT_ON_MASTER的异常。接着才释放锁。

前面说到了数据迁移开始configserver会先进行rebuild过程，接下来看看configserver还做了哪些工作。

configserver在收到dataserver发送的bucket迁移完成请求后，如果是slave configserver收到，则直接返回ok。master configserver收到首先比较dataserver发来的server_version和当前的server_version是否一致，如果不一致则返回false。不一致说明dataserver的数据分布表版本低了，它得重新用最新的数据分布表版本来进行迁移工作。一致则往下走，把d_hash_table该bucket的每个副本bucket对应的dataserver都拷贝给m_hash_table和hash_table，接着更新client_version和migrate_version，然后发送给slave configserver。
     
configserver在server_conf_thread的循环逻辑里面还会迁移是否完成（hashtable、m_hashtable、d_hashtable数据一致），如果完成了就会增加client_version、server_version，然后发送给slave configserver。 至此完成了所有的迁移任务。
     
迁移过程中还有个migrate_version。configserver只会在重建数据分布表前后会更新server_version，这样在重建过程中一个dataserver无法知道其他dataserver迁移的状态，虽然configserver和dataserver的数据分布表server_version一致，但是configserver的hashtable就跟dataserver的hashtable却不一致了。migrate_version就是迁移过程中dataserver的hashtable跟configserver的hashtable同步的版本。在每次有dataserver上报configserver迁移完成的时候，configserver的migrate_version就会更新，然后在dataserver发送心跳请求包的时候，configserver发现请求包的migrate_version跟最新的migrate_version不一致，就会在响应包发送数据分布表。dataserver在接收心跳响应包的时候，发现响应包的migrate_version跟自己保存的migrate_version不一致就引发try_update_table操作。
     
try_update_table操作也不是直接拷贝最新的hashtable，而是把主副本bucket原先不属于该dataserver负责，而现在变为该dataserver负责的bucket更新成最新的hashtable值，即以下代码：

          for (int i = 0; i < (int)bucket_count; i++) {
               //原先该master bucket不属于本机负责，现在变为本机负责
               if ((tmp_table[i] == local_server_ip::ip) && (server_table[i] != local_server_ip::ip)) {
                    tair::common::CScopedRwLock __scoped_lock(&m_mutex, true);
                    log_error("change table of bucket %d", i);
                    for (int j = 0; j < (int)copy_count; j++) {
                         server_table[i + j * bucket_count] = tmp_table[i + j * bucket_count];
                    }
               }
          }
    
存储引擎
---    

最后再说下NKV的存储引擎storage的实现方式，它其实就是所有引擎都要实现storage_manager类里的方法，然后上层dataserver只要调用storage_manager里的公共方法即可。tair目前支持kdb([Kyoto Cabinet](http://fallabs.com/kyotocabinet/))、ldb（[LevelDB](http://leveldb.org/)），mdb（[memcached](http://memcached.org/)）、rdb（[redis](http://redis.io/)）。目前kdb、ldb几乎没有使用。mdb在淘宝内部广泛使用，mdb并没有直接在memcached上做适配，而是基于memcached的思想自己实现了内存key/value存储，它最大的创新在于可用配置使用共享内存的方式。memcached数据全保存在内存里，如果重启会造成数据丢失，这样对升级来说非常困难。而mdb以共享内存的方式保存数据，只要主机不宕机，数据都不会丢失，升级只是重启的过程短暂不可用。可能是当初淘宝开发这套系统在升级、服务宕掉上吃过苦头，所以想出这样的创新，的确厉害！rdb在淘宝内部也有使用，它实现的引擎比较简单。redis源码中，对redis的数据结构操作都是通过redisClient结构体完成的，rdb也是把请求封装成redisClient结构体去请求数据。redis功能强大，俨然达到了数据库级别，像NKV这样的共享模式无法适应每个应用的需求，所有我们也在开发NCR（网易云redis），每个应用能申请独占的redis集群，这个不久就能上线。



