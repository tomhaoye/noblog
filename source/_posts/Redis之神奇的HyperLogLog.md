---
title: Redis之神奇的HyperLogLog
date: 2018-11-18 14:47:57
tags: Redis
categories: 笔记
---

> 这篇笔记想码很久了，于是今天码完上一篇二维码的文章后立马先占个坑，免得后面又被其他题材冲掉了。

在某一天开完了需求会议之后，我新的工作任务中包含了一项统计浏览量的任务。关于这项任务的详细需求大致如下：

 - 每个用户（user_id）每天浏览过该数据(`1~n`次)算一个单位浏览量
 - 能够统计每条数据一直以来总的浏览量
 - 能够得出15天内的按浏览量排序总排行榜和地区排行榜
 - 能够得出15天内的实际的`UV`总排行榜和地区排行榜
 - 允许存在数分钟的数据更新延迟
 - 允许出现小程度误差

> 由于怕概念有点混淆，所以我这篇文章里将以一天为单位的`UV`称为浏览量，将以15天为单位的`UV`称为`UV`

相信大家看到上面列出的要点后也进行了一轮思考，不知道大家是否理解了`浏览量`和`UV`的概念。或许大家能够想到很多种方案，不管是轻捷的、复杂的、常规的还是骚气的，都希望大家可以分享一下，而我在此也分享一下自己的思路以及实现过程。

从标题中就可以看到，本文将使用到`Redis`，以及`Redis`新支持的一种数据类型`HyperLogLog`。实际上一开始我并不清楚`HyperLogLog`的使用方法，但是从需求出发，有序集合`SortedSet`是需要的，而剩下的步骤，就看自己对`Redis`的理解能不能骚出点东西来了。

由于`Redis`的特性，我认为它能够成为一个很好的中转站，但不应该是一个仓库，所以总的浏览量我最终会选择存储到`Mysql`等关系型数据库中，而`Redis`缓存下来的，则是`15`天内的数据。假设我们的`SortedSet`中存储的是汇总了`15`天内的浏览量数据，那么第一个问题，就是我们如何计算得出总浏览量。如我们定时将缓存数据存入`Mysql`，而`Mysql`中我们只用一个字段存储这个总浏览量，那么那么我们在第`16`天的时候，总浏览量就会丢失掉第一天的数据。所以我在这里使用了三个字段来协助总浏览量的存储，一个是`当天浏览量`，第二个是`今天前的总浏览量`，剩下一个是`缓存更新时间`，为了方便，下面我们会将：
 - `总浏览量`称为`total_view`
 - `当天浏览量`称为`day_view`
 - `今天前的总浏览量`称为`total_view_bt`
 - `缓存更新时间`称为`cache_update_time`

所以最终我们的`total_view = day_view + total_view_bt`，当定时更新数据库数据的时候，如果当前时间跟`cache_update_time`是同一天，则只更新`day_view`值，如果不是同一天，则会先将`total_view_bt`更新为`total_view_bt + day_view`，然后再将`day_view`替换为`Redis`中的当天浏览量数据。

这样我们`Mysql`的总浏览量存储是搞定了，那么接下来我们就需要用`Redis`存储好这`15`天的浏览量了，那这每天的数据我们应该用什么结构去存储呢？我们先来回顾一下需求，首先我们的需求只需要统计浏览量，而不需要知道这个数据具体被哪些用户浏览过，那么就是说明我们不需要知道我存储的实际内容，而另一方面，我们需要的是最近`15`天的数据，而`15`天前的数据已被存储到`Mysql`中，所以第`1`天的数据应该在第`16`天凌晨就过期了。

终上所诉，在大家都熟悉的`Redis`五种数据类型中，能满足需求的，我个人认为只有`SortedSet`是比较符合的（注意这里的`SortedSet`跟上面汇总的`SortedSet`不是同一个）。如果使用`SortedSet`，我想到两种方案，一种是以`data_id + user_id + day`作为`SortedSet`里面元素的键，以过期时间作为`score`通过定时任务剔除过期数据；另外一种是使用`data_id + day`作为键，即一个数据存储`15`个`SortedSet`，然后再单独设置过期时间。但是上述两种方案都存在一些比较麻烦的问题，例如：
 - 方案一需要代码维护`SortedSet`；方案二需要复杂的去重统计`UV`
 - 方案一和方案二都需要一次性获取全部数据进行统计浏览量以及排序
 - `SortedSet`的插入需要`O(N)`的时间复杂度，影响效率
 - 我们完全不需要知道我存储的实际内容，使用`SortedSet`比较浪费空间

认真的思考过后发现存在较多的问题，那到底还有没有其他比较好的方案可以使用呢？或许我们可以开阔一下视野，不要局限于我们常用的数据类型上。后来我是想起了之前比较热门的一篇文章：`如何判断一个数是否在40亿个整数中？`，这篇文章讲到了`bitmap`的使用，而`bitmap`也是`Redis`最新支持的数据类型之一，于是我就去阅读关于`Redis`新支持的两种数据类型的文章了。看完之后豁然开朗，发现`bitmap`和`HyperLogLog`都比较满足我们对效率、空间和简单编码的要求，最后就采用了`HyperLogLog`的数据类型去实现我们的需求。

先简单的介绍一下`HyperLogLog`的特性和操作，首先`HyperLogLog`是用来做基数统计的，基数统计指的是指统计一个集合中不同的元素的个数，下面摘录一段源于`《HyperLogLog the analysis of a near-optimal cardinality estimation algorithm》`的简述：

>在理想状态下，将一堆数据hash至[0,1]，每两点距离相等，1/间距 即可得出这堆数据的基数。然而实际情况往往不能如愿，只能通过一些修正不断的逼近这个实际的基数。实际采用的方式一是分桶，二是取kmax。分桶将数据分为m组，每组取第k个位置的值，所有组中得到最大的kmax，(k-1)/kmax得到估计的基数。
>HyperLogLog算法的另一个主观上的理解可以用抛硬币的方式来理解。以当硬币抛出反面为一次过程，当你抛n次硬币全为正面的概率为1/2^n。当你经历过k(k很大时)次这样的过程，硬币不出现反面的概率基本为0。假设反面为1，正面为0，每抛一次记录1或者0，当记录上显示为0000000...001时，这种可以归结为小概率事件，基本不会发生。转换到基数的想法就是，可以通过第一个1出现前0的个数n来统计基数，基数大致为2^(n+1)时。硬币当中可以统计为(1/2*1+1/4*2+1/8*3...)，大致可以这么去想。

有兴趣深入了解的童鞋可以`Google`一下上面的论文，下面我们回归到如何在`Redis`中操作`HyperLogLog`上来。`Redis`中一共有三种操作`HyperLogLog`的方法：
 - PFADD: 将任意数量的元素添加到指定的`HyperLogLog`里面。作为这个命令的副作用，`HyperLogLog`内部可能会被更新， 以便反映一个不同的唯一元素估计数量（也即是集合的基数）。如果`HyperLogLog`估计的近似基数在命令执行之后出现了变化， 那么命令返回`1`， 否则返回`0`。 如果命令执行时给定的键不存在， 那么程序将先创建一个空的`HyperLogLog`结构， 然后再执行命令。
 - PFCOUNT：当`PFCOUNT`命令作用于单个键时， 返回储存在给定键的`HyperLogLog`的近似基数，如果键不存在，那么返回`0`。当`PFCOUNT`命令作用于多个键时，返回所有给定`HyperLogLog`的并集的近似基数，这个近似基数是通过将所有给定`HyperLogLog`合并至一个临时`HyperLogLog`来计算得出的。通过`HyperLogLog`数据结构，用户可以使用少量固定大小的内存，来储存集合中的唯一元素（每个`HyperLogLog`只需使用`12k`字节内存，以及几个字节的内存来储存键本身）。命令返回的可见集合（observed set）基数并不是精确值，而是一个带有`0.81%`标准错误（standard error）的近似值。
 - PFMERGE：将多个`HyperLogLog`合并（merge）为一个`HyperLogLog`，合并后的`HyperLogLog`的基数接近于所有输入`HyperLogLog`的可见集合（observed set）的并集。合并得出的`HyperLogLog`会被储存在`destkey`键里面，如果该键并不存在，那么命令在执行之前，会先为该键创建一个空的`HyperLogLog`。

知道了如何操作`HyperLogLog`后我们就开始用它来实现我们的需求吧。

首先我使用`data_id + day`作为`SortedSet`的键记录该资源数据当天的`UV`，其中`day`是由当前时间戳除以一天的秒数然后再模`15`得出的一个`0~14`的数，即`day = (1543127511 // 86400) % 15`，然后用户浏览的时候将用户的`user_id`存进去以及设置好过期时间，最后将`data_id_day1`到`data_id_day15`的数据循环用`PFCOUNT`取出来后相加的总和是每天的`UV`总和（即理解为15天内的浏览量），简称之为`15UDV`，而将`data_id_day1`到`data_id_day15`的数据一次性使用`PFCOUNT`得出来的结果是15天的`UV`结果，简称`UV`。那么`15UDV`和`UV`是两种不同排序依据，所以我们这里将此资源数据的`UV`值存入到最后用于排序的`SortedSet`里面。由于有总排行榜和城市排行榜，所以我们需要定义两种`SortedSet`：
 - rank：15天`UV`总排行榜
 - rank_[city]：`city`替换为对应资源所在城市的15天`UV`城市排行榜

至于`15UDV`嘛，其实跟上面`UV`也是一样的操作，就不多说了。所以到这里所有步骤已经完成了，下面就展示一下的伪代码方便大家再理顺一下：

```
# 存储过程
data_id = 1
city = 'guangzhou'
user_id = 'ove1d291d2912ed1ad91e2'
day = (now.timestamp // 86400) % 15
15day_after = now.date.timestamp + 15*86400
key = str(data_id) + '_' + str(day)
this.redis.multi()
this.redis.pfadd(key, user_id)
this.redis.expireat(key, 15day_after)
this.redis.exec()

# 获取UV
for i in range(15):
	keys = [str(data_id) + '_' + str(i)]
UV = this.redis.pfcount(keys)
# 获取15UDV
15UDV = 0
for i in range(15):
	udv_key = str(data_id) + '_' + str(i)
	15UDV += this.redis.pfcount(udv_key)
# 存储到排行榜中
rank_key = 'rank'
city_rank_key = rank_key + '_' + city
udv_rank_key = 'udv_rank'
udv_city_rank_key = udv_rank_key + '_' + city
this.redis.multi()
this.redis.zadd(rank_key, UV, data_id)
this.redis.zadd(city_rank_key, UV, data_id)
this.redis.zadd(udv_rank_key, 15UDV, data_id)
this.redis.zadd(udv_city_rank_key, 15UDV, data_id)
this.redis.exec()

# Mysql统计总浏览量
resource = orm.table('resource').get(data_id)
if date(resource.cache_update_time) != date(now.timestamp):
	resource.total_view_bt += resource.day_view
resource.day_view = this.redis.pfcount(key)
resource.cache_update_time = now.timestamp
resource.total_view = resource.total_view_bt + resource.day_view
resource.save()
```

完。

