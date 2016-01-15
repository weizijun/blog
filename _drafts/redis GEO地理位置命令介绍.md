title: redis GEO地理位置命令介绍
tags:
---
## 概述
GEO目前提供以下6个命令。

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


### geopos

geopos可以获取地理位置的坐标，可以批量获取多个地理位置的坐标，命令格式为：

	GEOPOS key member [member ...]


### geodist

geodist用来获取两个地理位置的距离，命令格式为：

	GEODIST key member1 member2 [unit]

单位可以指定为以下四种类型：

*	m：米，距离单位默认为米，不传递该参数则单位为米。
*	km：公里。
*	mi：英里。
*	ft：英尺。

### georadius

georadius可以根据给定地理位置坐标获取指定范围内的地理位置集合。命令格式为：

	GEORADIUS key longitude latitude radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count]
	
	
### georadiusbymember

georadiusbymember可以根据给定地理位置获取指定范围内的地理位置集合。georadius命令传递的是坐标，georadiusbymember传递的是地理位置。georadius更为灵活，可以获取任何坐标点范围内的地理位置。但是大多数时候，只是想获取某个地理位置附近的其他地理位置，使用georadiusbymember则更为方便。georadiusbymember命令格式为：
	
	GEORADIUSBYMEMBER key member radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count]


### geohash

geohash可以获取某个地理位置的geohash值。geohash是将二维的经纬度转换成字符串hash值的算法，后面会具体介绍geohash原理。可以批量获取多个地理位置的geohash值。命令格式为：

	GEOHASH key member [member ...]


## redis GEO实现

redis GEO实现主要包含了以下两项技术：

*	1、使用geohash保存地理位置的坐标。
*	2、使用有序集合保存地理位置的集合。

### geohash

geohash的思想是将二维的经纬度转换成一维的字符串，geohash有以下三个特点：

*	1、字符串越长，表示的范围越精确。编码长度为8时，精度在19米左右，而当编码长度为9时，精度在2米左右。
*	2、字符串相似的表示距离相近，利用字符串的前缀匹配，可以查询附近的地理位置。这样就实现了快速查询某个坐标附近的地理位置。
*	3、geohash计算的字符串，可以反向解码出原来的经纬度。

这三个特性让geohash特别适合表示二维hash值。这篇文章：[GeoHash核心原理解析](http://www.cnblogs.com/LBSer/p/3310455.html)详细的介绍了geohash的原理，想要了解geohash实现的朋友可以参考这篇文章。

### redis GEO命令实现

知道了redis使用有序集合保存地理位置数据，以及geohash的特性，就很容易理解redis是如何实现redis GEO命令了。

geoadd命令增加地理位置的时候，会先计算地理位置坐标的geohash值，然后地理位置作为有序集合的member，geohash作为该member的score。

geopos命令则先根据地理位置获取geohash值，然后decode得到地理位置的坐标。

geodist命令先根据两个地理位置各自得到坐标，然后计算两个坐标的距离。

georadius和georadiusbymember使用相同的实现，georadiusbymember多了一步把地理位置转换成对应的坐标。然后查找该坐标和周围对应8个坐标符合距离要求的地理位置。因为geohash得到的值其实是个格子，并不是点，这样通过计算周围对应8个坐标就能解决边缘问题。由于使用有序集合保存地理位置，在对地列位置基于范围查询，就相当于实现了zrange命令，内部的实现确实与zrange命令一致，只是geo有些特别的处理，比如获得的某个地理位置，还需要计算该地理位置是否符合给定的距离访问。

geohash则直接返回了地理位置的geohash值。

参考文献：

*	[geo命令文档](http://redis.io/commands#geo)
*	[Redis GEO 特性简介](http://blog.huangz.me/diary/2015/redis-geo.html)
*	[Redis GEO 源码注释](http://blog.huangz.me/diary/2015/annotated-redis-geo-source.html)
*	[GeoHash核心原理解析](http://www.cnblogs.com/LBSer/p/3310455.html)
*	[深入浅出空间索引：为什么需要空间索引](http://www.cnblogs.com/LBSer/p/3392491.html)
*	[geohash wiki](https://en.wikipedia.org/wiki/Geohash)
*	[WGS84 wiki](https://zh.wikipedia.org/wiki/WGS84)
*	[离我最近之geohash算法（增加周边邻近编号）](http://blog.csdn.net/sunrise_2013/article/details/42395261)