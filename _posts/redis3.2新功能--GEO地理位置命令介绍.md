title: redis3.2新功能--GEO地理位置命令介绍
categories: redis
tags: 
- redis
- geohash
date: 2016-03-28 00:00:00

---
## 概述

redis3.2发布rc版本已经有一段时间了，估计RedisConf 2016左右，3.2版本就能release了。3.2版本中增加的最大功能就是对GEO（地理位置）的支持。说起redis的GEO特性，最大的贡献还是咱们中国人。redis作者在对3.2引进新特性的博客中介绍了为什么支持GEO。GEO hashing的api是在Ardb实现的，Ardb是github用户yinqiwen实现的基于redis协议实现的nosql系统，Ardb支持除了redis、还有LevelDB、RocksDB
、LMDB等kv引擎。其中Ardb实现了GEO hashing功能。从Ardb作者的用户名和标识的位置在深圳可以看出Ardb作者应该是咱中国人。Ardb是用c++写的。redis另一个开发者Matt Stancliff从Ardb提取GEO库，用C语言改写，整合进redis的一个自己的分支，并被redis作者接受，合并进了3.2版本。GEO目前提供以下6个命令。

*	1、geoadd：增加某个地理位置的坐标。
*	2、geopos：获取某个地理位置的坐标。
*	3、geodist：获取两个地理位置的距离。
*	4、georadius：根据给定地理位置坐标获取指定范围内的地理位置集合。
*	5、georadiusbymember：根据给定地理位置获取指定范围内的地理位置集合。
*	6、geohash：获取某个地理位置的geohash值。

地理位置的坐标是以WGS84为标准，WGS84，全称World Geodetic System 1984，是为GPS全球定位系统使用而建立的坐标系统。

## GEO命令
下面来看看具体每个命令的用法。

### geoadd

geoadd用来增加地理位置的坐标，可以批量添加地理位置，命令格式为：

	GEOADD key longitude latitude member [longitude latitude member ...]

key标识一个地理位置的集合。`longitude latitude member`标识了一个地理位置的坐标。longitude是地理位置的经度，latitude是地理位置的纬度。member是该地理位置的名称。GEOADD可以批量给集合添加一批地理位置。

### geopos

geopos可以获取地理位置的坐标，可以批量获取多个地理位置的坐标，命令格式为：

	GEOPOS key member [member ...]


### geodist

geodist用来获取两个地理位置的距离，命令格式为：

	GEODIST key member1 member2 [m|km|ft|mi]

单位可以指定为以下四种类型：

*	m：米，距离单位默认为米，不传递该参数则单位为米。
*	km：公里。
*	mi：英里。
*	ft：英尺。

### georadius

georadius可以根据给定地理位置坐标获取指定范围内的地理位置集合。命令格式为：

	GEORADIUS key longitude latitude radius [m|km|ft|mi] [WITHCOORD] [WITHDIST] [ASC|DESC] [WITHHASH] [COUNT count]
	
`longitude latitude`标识了地理位置的坐标，radius表示范围距离，距离单位可以为m|km|ft|mi，还有一些可选参数：

*	WITHCOORD：传入WITHCOORD参数，则返回结果会带上匹配位置的经纬度。
*	WITHDIST：传入WITHDIST参数，则返回结果会带上匹配位置与给定地理位置的距离。
*	ASC|DESC：默认结果是未排序的，传入ASC为从近到远排序，传入DESC为从远到近排序。
*	WITHHASH：传入WITHHASH参数，则返回结果会带上匹配位置的hash值。
*	COUNT count：传入COUNT参数，可以返回指定数量的结果。
	
### georadiusbymember

georadiusbymember可以根据给定地理位置获取指定范围内的地理位置集合。georadius命令传递的是坐标，georadiusbymember传递的是地理位置。georadius更为灵活，可以获取任何坐标点范围内的地理位置。但是大多数时候，只是想获取某个地理位置附近的其他地理位置，使用georadiusbymember则更为方便。georadiusbymember命令格式为（命令可选参数与georadius含义一样）：
	
	GEORADIUSBYMEMBER key member radius [m|km|ft|mi] [WITHCOORD] [WITHDIST] [ASC|DESC] [WITHHASH] [COUNT count]


### geohash

geohash可以获取某个地理位置的geohash值。geohash是将二维的经纬度转换成字符串hash值的算法，后面会具体介绍geohash原理。可以批量获取多个地理位置的geohash值。命令格式为：

	GEOHASH key member [member ...]


## redis GEO实现

redis GEO实现主要包含了以下两项技术：

*	1、使用geohash保存地理位置的坐标。
*	2、使用有序集合（zset）保存地理位置的集合。

### geohash

geohash的思想是将二维的经纬度转换成一维的字符串，geohash有以下三个特点：

*	1、字符串越长，表示的范围越精确。编码长度为8时，精度在19米左右，而当编码长度为9时，精度在2米左右。
*	2、字符串相似的表示距离相近，利用字符串的前缀匹配，可以查询附近的地理位置。这样就实现了快速查询某个坐标附近的地理位置。
*	3、geohash计算的字符串，可以反向解码出原来的经纬度。

这三个特性让geohash特别适合表示二维hash值。这篇文章：[GeoHash核心原理解析](http://www.cnblogs.com/LBSer/p/3310455.html)详细的介绍了geohash的原理，想要了解geohash实现的朋友可以参考这篇文章。

### redis GEO命令实现

知道了redis使用有序集合（zset）保存地理位置数据(想了解redis有序集合的，可以参看这篇文章[《有序集合对象》](http://redisbook.com/preview/object/sorted_set.html))，以及geohash的特性，就很容易理解redis是如何实现redis GEO命令了。细心的读者可能发现，redis没有实现地理位置的删除命令。不过由于GEO数据保存在zset中，可以用zrem来删除某个地理位置。

*	geoadd命令增加地理位置的时候，会先计算地理位置坐标的geohash值，然后地理位置作为有序集合的member，geohash作为该member的score。然后使用zadd命令插入到有序集合。
*	geopos命令则先根据地理位置获取geohash值，然后decode得到地理位置的坐标。
*	geodist命令先根据两个地理位置各自得到坐标，然后计算两个坐标的距离。
*	georadius和georadiusbymember使用相同的实现，georadiusbymember多了一步把地理位置转换成对应的坐标。然后查找该坐标和周围对应8个坐标符合距离要求的地理位置。因为geohash得到的值其实是个格子，并不是点，这样通过计算周围对应8个坐标就能解决边缘问题。由于使用有序集合保存地理位置，在对地列位置基于范围查询，就相当于实现了zrange命令，内部的实现确实与zrange命令一致，只是geo有些特别的处理，比如获得的某个地理位置，还需要计算该地理位置是否符合给定的距离访问。
*	geohash则直接返回了地理位置的geohash值。

redis关于geohash使用了Ardb的geohash库`geohash-int`，redis使用的geohash编码长度为26位。可以精确到0.59m的精度。

## 总结

通过本文，拨开GEO身后的云雾，可以看出redis借助了有序集合（zset）和geohash，加上redis本身实现的命令框架，可以很容易的实现地理位置相关的命令。

参考文献：

*	[geo命令文档](http://redis.io/commands#geo)
*	[Redis GEO 特性简介](http://blog.huangz.me/diary/2015/redis-geo.html)
*	[Redis GEO 源码注释](http://blog.huangz.me/diary/2015/annotated-redis-geo-source.html)
*	[GeoHash核心原理解析](http://www.cnblogs.com/LBSer/p/3310455.html)
*	[深入浅出空间索引：为什么需要空间索引](http://www.cnblogs.com/LBSer/p/3392491.html)
*	[geohash wiki](https://en.wikipedia.org/wiki/Geohash)
*	[WGS84 wiki](https://zh.wikipedia.org/wiki/WGS84)
*	[离我最近之geohash算法（增加周边邻近编号）](http://blog.csdn.net/sunrise_2013/article/details/42395261)
*	[有序集合对象](http://redisbook.com/preview/object/sorted_set.html)