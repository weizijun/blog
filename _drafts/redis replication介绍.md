title:      redis replication介绍
categories: redis
tags: 
- redis
- replication
date: 2015-08-08 00:00:00
---
redis自带了复制功能，可以实现主从间的数据同步。redis发展到现在，复制功能已经比较稳定，大量使用在生产环境。本文源码使用的redis最新的3.0.3版本。先翻译下官网Replication的介绍，了解下Replication的功能。

* redis使用异步复制，从2.8版本开始，slave可以定期的从复制流中获取数据。
* 一个master可以支持多个slave。
* slave也能有自己的slave，除了连上同样master的slave，其他redis都能成为该slave的slave，可以构成一个图状结构。
* 复制是非阻塞的，这意味着即使有一个或多个从服务器正在进行初次同步，master也会接着处理请求。
* 复制在slave端也是非阻塞的。但slave在初次连接master的时候，只要你在redis.conf配置，它也能请求旧的数据。在初始化复制完成后，slave上的旧数据会被删除，slave开始装载新的数据，此时会阻塞一小段时间。
* 复制可以用来保证扩展性和简单的数据冗余。扩展性表现在可以提供多个slave做只读（例如，笨重的sort命令可以在slave上完成）。
* 复制可以避免在master执行持久化，在master上去掉save配置，让slave完成持久化操作。然而这样要避免master自动重启（重启后master数据丢失，slave同步后也丢弃持久化数据）。

### replication流程

* 步骤一：`客户端`向`slave`发送"slaveof masterIp masterPort""命令，或者在slave上配置"slaveof masterIp masterPort"
* 步骤二：`slave`定时任务检测到masterIp、masterPort，向`master`发送同步命令，2.6版本是"SYNC"命令，2.8版本多了"PSYNC <MASTER_RUN_ID> <OFFSET>"命令，可以告知自己当前的复制偏移量，有可能能完成增量同步。对于初次复制，2.8版本使用的命令是"PSYNC ? -1"。
* 步骤三：`master`收到"SYNC"或者"PSYNC""命令，"PSYNC"会有两种处理，具体细节后面再说，这里两种处理分别为全量同步和增量同步。"SYNC"只有全量同步的方式。当master执行全量同步，首先会执行bgsave，生成最新的rdb文件。同时增量数据保存在slave结构体的发送缓冲区里。
* 步骤四：`master`将bgsave的rdb文件传给slave。
* 步骤五：`slave`接收rdb文件，并保存在临时文件里，当rdb文件全部接收后，改名为配置文件定义的rdb文件名称。
* 步骤六：`slave`开始装载rdb文件，此时会阻塞读写请求。
* 步骤七：`master`将rdb文件发给slave后，开始将发送缓存的数据发送给slave。此处仅是打开io的可写标识，`master`开始把发送缓冲区的数据发给slave。
* 步骤八：`slave`接收到`master`的命令后，在本地执行命令，写入内存。
* 步骤九：此后`master`有写入数据，都会向`slave`发送一份，在`slave`处执行。
* 步骤十：`slave`每秒向master发送"REPLCONF ACK <OFFSET>"告知自己的偏移量。
* 步骤十一：`master`每隔10s向`slave`发送"PING"命令，探知slave的存活情况。
* 步骤十二：在2.8版本后，如果slave与master连接断开后，slave仍然有机会重连上master，而不用执行这么笨重的初始化方法。slave发送"PSYNC <MASTER_RUN_ID> <OFFSET>"，把缓存的"MASTER_RUN_ID"和当前的"OFFSET"告知master。master有块内存缓冲区会缓存最近的写操作（关于内存缓冲区，"replication内存缓冲区"章节会详细介绍），如果"OFFSET"在master的内存缓冲区里，则直接把master从"OFFSET"到当前缓冲区位置的数据复制给slave，这样slave数据就直接和master保持同步了。

### replication实现解析

#### replication状态说明

redis replication流程的实现是基于状态处理的。replication的状态分为master的视角和slave的视角。

master有如下几个状态

	#define REDIS_REPL_WAIT_BGSAVE_START 6 /* We need to produce a new RDB file. */
	#define REDIS_REPL_WAIT_BGSAVE_END 7 /* Waiting RDB file creation to finish. */
	#define REDIS_REPL_SEND_BULK 8 /* Sending RDB file to slave. */
	#define REDIS_REPL_ONLINE 9 /* RDB file transmitted, sending just updates. */


slave有如下几个状态

	#define REDIS_REPL_NONE 0 /* No active replication */
	#define REDIS_REPL_CONNECT 1 /* Must connect to master */
	#define REDIS_REPL_CONNECTING 2 /* Connecting to master */
	#define REDIS_REPL_RECEIVE_PONG 3 /* Wait for PING reply */
	#define REDIS_REPL_TRANSFER 4 /* Receiving .rdb from master */
	#define REDIS_REPL_CONNECTED 5 /* Connected to master */

接下来master和slave的实现可以看到redis如何根据这些状态完成复制功能。

#### master的实现

##### sync命令

master的replication流程是从接收到sync命令开始的。syncCommand包含了sync命令和psync命令。对于增量同步，前面replication流程步骤十二已经简要说明，在`replication内存缓冲区`章节会详细介绍。增量同步完成，master、slave就进入到正常的复制阶段。

如果是全量同步，master会返回给slave`+FULLRESYNC`。然后开始同步操作。master首先会执行bgsave生成最新的rdb文件。对于多个slave同时初始化复制，redis这里做了优化，如果在slave发送sync命令的时候，master正在执行bgsave，则master会把第一个触发它执行bgsave的slave增量数据拷贝给执行sync命令的slave。实现的方式是，master遍历其他slave，如果有slave状态是REDIS_REPL_WAIT_BGSAVE_END，则说明有slave在等待传输rdb文件。master就会把该slave的增量数据拷贝给发送命令的slave，同时把slave状态改成REDIS_REPL_WAIT_BGSAVE_END。如果没有slave的状态是REDIS_REPL_WAIT_BGSAVE_END，这个分两种情况，一种是master没有bgsave任务，该slave是第一个sync的slave，所以立即执行bgsave，同时该slave状态变成REDIS_REPL_WAIT_BGSAVE_END。

但是还有另一种情况，第一个sync的slave到来的时候，该master已经在执行bgsave了（aof rewrite或者主动执行bgsave命令等），那么master会把该slave状态设置为REDIS_REPL_WAIT_BGSAVE_START。等到前一个bgsave执行完成，又重新开始执行slave的bgsave流程。

最新的redis执行另一种全量同步的方式，master执行bgsave的时候不再生成rdb文件，而是直接使用socket，将bgsave的rdb数据发送给slave。通过配置`repl-diskless-sync yes`就能实现，不过该功能还属于实验阶段，配置文件里给出了醒目的提示：`WARNING: DISKLESS REPLICATION IS EXPERIMENTAL CURRENTLY`

##### 传输rdb数据
bgsave执行完成，会回调replication的updateSlavesWaitingBgsave函数，updateSlavesWaitingBgsave函数检查每个slave，如果有状态为REDIS_REPL_WAIT_BGSAVE_START的slave，就如前面说的那样，会重新执行bgsave，如果有状态为REDIS_REPL_WAIT_BGSAVE_END的slave，那么就会进入下一个步骤，slave的状态变成REDIS_REPL_SEND_BULK。然后开始传输rdb文件。

传输之前先删除slave之前的写事件，然后把写事件注册到传输函数`sendBulkToSlave`，传输的时候从文件不断读取rdb文件，一次最多读取16KB数据，然后往slave的socket写入。全部传输完成，就会把slave的状态设置为REDIS_REPL_ONLINE。同时注册写事件到`sendReplyToClient`，把发送缓冲区的数据写入slave。

##### 命令传播
master把数据复制到slave使用的是命令传播的方式，即把在master的执行的写操作异步发送给slave，让slave也重新执行一次。master在执行一个命令的时候，执行完成会判断该命令是否会把数据库写脏，即用了一个全局的`server.dirty`变量记录数据库是否发生变化，如果发生变化就会执行propagate函数，propagate函数会做两件事情，一件是写aof文件，一件就是复制操作，调用replicationFeedSlaves函数。

replicationFeedSlaves函数会先判断master是否有slave，没有的话就自己返回了。有slave的话，就先把命令写入内存缓冲区。然后遍历所以的slave，把数据写入各个slave的发送缓冲区，事件处理方法会把数据发给异步发给slave。

master在执行写命令的时候，可以通过配置少要多少个健康slave的数量来决定是否能执行写操作。健康的slave统计是在定时任务中执行的，在定时任务中可以看到它的实现。

##### replication内存缓冲区
replication内存缓冲区可用来保存最近的写操作，这样如果slave断线重连的时候，如果能给出slave的当前offset，并且该offset在缓冲区内，master就能直接把缓冲区内的增量数据拷贝给slave，两边的数据就保持同步了。redis以前必须全量复制，即使网络抖动，也得重新执行全量同步，资源消耗明显。redis在2.8版本引入内存缓冲区，可以支持部分同步，大大提高了复制的稳定性。

replication内存缓冲区是块环形数组，master会把写命令发给slave的时候，也写入内存缓冲区，缓冲区的实现使用了下面几个变量：
repl_backlog（char *）：保存了缓冲区的数据。
repl_backlog_size（long long）：缓冲区的长度。
repl_backlog_idx（long long）：缓冲区当前位置。
repl_backlog_off(long long):backlog 中可以被还原的第一个字节的偏移量。
repl_backlog_histlen（long long）：缓冲区数据的真实长度，因为缓冲区是环形，为了确认缓冲区当前位置之后的数据是否还没写入，所以需要该字段保存缓冲区实际数据的长度。如果缓冲区还没写满，保存的就是没写满的长度，如果写满了，开始覆盖写，大小就和repl_backlog_size一样。
master_repl_offset（long long）：全局复制偏移量。
repl_backlog_time_limit（time_t）：缓冲区过期时间，当一段时间master没有slave，则释放缓冲区。
 
##### 定时任务
复制定时任务实现的函数是replicationCron，每秒执行一次，里面集成了master和slave的实现。这里先理出master的内容。

master默认每10秒向稳定状态的slave发送ping命令。这样即使tcp连接未断开，slave也能显式的判断master是否在线。

对于还在初始化连接的slave，master在执行bgsave的时候，为了防止master与slave之间太长未通信，导致socket连接关闭，master会在每次调用replicationCron的时候给`REDIS_REPL_WAIT_BGSAVE_START`或者落盘传输rdb文件的`REDIS_REPL_WAIT_BGSAVE_END`状态的slave发送"\n"。

master还会检查slave最新的响应时间是否超过了响应超时时间，超时的slave，master会关掉连接，释放资源。默认超时时间是60s。

在replication内存缓冲区说到repl_backlog_time_limit是用来当一段时间master没有slave，则释放缓冲区。这个就是在replicationCron处理的。repl_backlog_time_limit默认是1小时。实现方法就是：在最后一个slave释放的时候会记录下当时的时间repl_no_slaves_since，然后比较repl_backlog_time_limit和当前时间与repl_no_slaves_since的差值。

对于使用socket方式复制的，如果当前没有执行传输rdb流程，且有slave在等待传输rdb数据超过repl_diskless_sync_delay，默认是5秒。就会重新开始rdb传输流程。

最后更新健康的slave数量，master用repl_good_slaves_count记录健康的slave数量。检查slave最新的响应时间是否长于repl_min_slaves_max_lag，小于repl_min_slaves_max_lag的就属于健康的slave。master还有个repl_min_slaves_to_write配置，在每次执行写命令的时候会比较repl_good_slaves_count和repl_min_slaves_to_write，只有repl_good_slaves_count大于等于repl_min_slaves_to_write，才能执行写操作。repl_min_slaves_max_lag和repl_min_slaves_to_write默认是不配置的，只有用户对数据可靠性要求高的场景下才需要开启该配置。

#### slave的实现

##### slaveof命令
slave复制master可以通过配置文件设置`slaveof masterIp masterPort`，另一种方式就是对slave发送`slaveof masterIp masterPort`命令。slaveof命令做的事情跟配置`slaveof masterIp masterPort`的事情差不多，配置的方式会修改masterhost和masterport，同时状态改成REDIS_REPL_CONNECT。slaveof命令除了更新这些变量，还会有一些清理操作，slave会断开它自己的slave，断开所有组塞住的客户端。清除replication内存缓冲区，如果有其他slave在初始化连接该slave，则取消复制操作。真正的连接操作会在slave的定时任务中完成。

##### 复制初始化
初始化的时候，slave的复制状态是REDIS_REPL_CONNECT，定时任务检测到该状态，开始连接master。连接成功，会更新状态为REDIS_REPL_CONNECTING，然后注册读写事件到syncWithMaster函数。

这样很快可写状态就会到来，syncWithMaster函数首先会检查socket是否正常，如果出现异常，就会goto error，goto error的操作就是，清除连接master的fd，slave状态变回REDIS_REPL_CONNECT，让定时任务重新执行初始化操作。接着如果是状态是REDIS_REPL_CONNECTING，首先删除可写状态，然后变更状态为REDIS_REPL_RECEIVE_PONG，并同步给master发送ping命令。

收到master的pong响应后，删除可读状态，同步读出master的响应消息。如果消息不对就会goto error。如果消息格式对了，但是提示消息如果不是‘+’且不是`-NOAUTH`且不是`-ERR operation not permitted`，就会goto error。`-NOAUTH`说明master需要密码验证，接下来slave就会进行密码验证。`-ERR operation not permitted`是兼容老版本redis使用的。如果配置了master的密码，slave就会像master同步发送auth命令验证密码，验证失败就会goto error。接着slave会向master同步发送`REPLCONF listening-port <port>`命令告知slave的端口号。

##### 向master发送复制命令
接着slave会向master同步发送`PSYNC <MASTER_RUN_ID> <OFFSET>`命令，slave如果当前缓存了断线的master，就传入该master的run id和offset，如果没缓存master的话，就传入"?"和"-1"。slave接收到master的回复，如果是`+CONTINUE`，则表明slave能直接进行增量同步。增量同步，slave会把缓存的master作为当前master，复制状态变为`REDIS_REPL_CONNECTED`，同时把master最为client加入client列表，注册读事件，开始接收master的消息。如果对master的写缓冲区有数据或者有要回复master的消息，也会注册写事件。这样slave就直接开始正常的复制流程。

如果master回复的是`+FULLRESYNC`，slave记录下master的runid和复制偏移量，释放缓存的master。如果master回复的是`-ERR`，则说明master不支持psync命令，slave会重新同步发送`SYNC`命令，然后slave就创建临时文件开始接收master发送的rdb文件，同时注册可读事件到readSyncBulkPayload函数，然后变更复制状态为REDIS_REPL_TRANSFER。

##### 接收master rdb文件
readSyncBulkPayload会不停的接收rdb数据，然后写到临时文件中。当整个rdb数据接收完成后，把临时文件重命名为rdb文件。然后清空db，删除可读事件。接着就开始导入rdb文件。导入rdb文件完成后，slave开始初始化master对象，在slave中，master的连接是以client的方式交互的，所以为master创建client对象，flag为加上master的标示。复制状态就变更为最终的稳定状态REDIS_REPL_CONNECTED。如果slave开启了aof，此时重置下aof，进行一次aof rewrite。

创建client对象的时候会注册上可读事件，slave就能接收在初始化连接过程中，master产生的增量数据，以及开始复制master的写命令。

##### 接收master传播过来的命令
slave变成稳定状态REDIS_REPL_CONNECTED后，打开了可读事件，就可以开始接收master发来的命令了，由于在slave看来，master是一个client结构体，所以就像client给slave发送命令一样执行命令。

##### 关于replicationSendNewlineToMaster
与master在slave连接的特殊状态会定时发送"\n"给slave一样，slave也会在一些长时间不与master交互的时候给master发送"\n"，防止master断开slave的连接。slave在导入rdb文件和清空数据库的时候地方会调用replicationSendNewlineToMaster。他们都是以回调的方式执行的。

导入rdb文件的时候，在读文件的时候会触发rdb文件回调方法rdbLoadProgressCallback，rdbLoadProgressCallback回调方法执行了replicationSendNewlineToMaster函数。

清空db的时候，支持注册回调函数，回调函数会在删除hash表数据的时候，没删除65535条记录的时候执行下回调函数，前面说道接收master传来的rdb数据完后，清空db的时候就注册了replicationEmptyDbCallback回调函数，replicationEmptyDbCallback函数就执行了replicationSendNewlineToMaster函数。

##### 关于slave缓存master功能
redis的redisServer结构体内有一个cached_master的变量，cached_master用来缓存断开的master，这样一旦因为网络异常造成的连接断开，slave有机会增量同步master。

在slave复制状态已经是稳定状态REDIS_REPL_CONNECTED的时候，如果master和slave的连接异常，或者连接心跳超时，会触发freeClient的操作，freeClient会调用replicationCacheMaster函数，replicationCacheMaster函数就会把server.master赋值给server.cached_master。

在psync的时候，如果cached_maseter不为NULL，就会把cached_maseter的runid和offset传给master。

如果master回复`+CONTINUE`，则cached_master就能直接恢复成master，replicationResurrectCachedMaster函数实现了恢复逻辑，即根据server.cached_master恢复server.master的状态。

如果slave发现cached_master不能恢复成server.master，就会丢弃它。丢弃的场景还有master不支持psync命令，slave开始赋值另外的master，slave成为master。

##### 定时任务
slave的定时任务与master一样，复实现的函数是replicationCron，每秒执行一次。

slave会检查在REDIS_REPL_CONNECTING或者REDIS_REPL_RECEIVE_PONG的状态下，如果最后一次与master通信的时间超过复制超时时间repl_timeout，默认是60秒，slave会断开和master的连接，然后状态变为REDIS_REPL_CONNECT，开始重新连接master。

接着检查slave在REDIS_REPL_TRANSFER状态下，如果最后一次与master通信的时间超过复制超时时间repl_timeout，slave会断开和master的连接，同时关闭并删除临时文件，然后状态变为REDIS_REPL_CONNECT，开始重新连接master。

接着检查slave在REDIS_REPL_CONNECTED下，如果最后一次与master通信的时间超过复制超时时间repl_timeout，slave会释放master连接，释放是以freeClient的方式释放，不过此时slave会缓存master，这样一旦slave重新连上master的时候能够执行增量同步。然后slave状态变为REDIS_REPL_CONNECT。这时候slave还会把复制自己的其他slave连接断开，这样促使其他slave也重新同步自己。

定时任务接着执行，slave会给master发送`REPLCONF ACK <OFFSET>`命令，告知slave的offset。


### 后话
整个replication.c的代码包括注释2104行，