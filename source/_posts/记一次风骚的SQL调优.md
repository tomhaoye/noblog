---
title: 记一次风骚的SQL调优
date: 2019-05-26 17:06:54
tags: MySQL
categories: 有趣
---

> 这次SQL调优的事情已经过去一段时间了，一直都没有时间彻底搞懂那个慢查的原因，先mark下来再慢慢研究。

具体事件起因经过就忽略吧，这里就简单的介绍一下基本信息以及模拟一下表结构和慢查询的语句。

当时使用的`MySQL`版本是`5.7`，具体的小版本大家就别问了，问我也不记得了。

然后表结构最主要的部分就如下所示：
```
# father表
CREATE TABLE `father` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `son_id` int(10) unsigned NOT NULL,
  `fname` varchar(255) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

```
# son表
CREATE TABLE `son` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

当时的慢`SQL`其实也非常的简单，大概就是这个样子的：

```SQL
select count(0) FROM test_father WHERE son_id in (SELECT id from test_son);
```

当时的数据量并不是很大，`father`表符合条件的数据大概是两到三万，`son`表总数据不到一万，最后查询得到的结果是`6000`左右，即是大概有`6000`条记录满足条件，但是查询时间大概要`6`秒，我在自己电脑上面大概模拟了一下数据量，也出现了慢查的情况，不过由于减少了其他的列或者总数据量有差别的原因，总的查询时间大概是`4`秒多，而我的`MySQL`版本是`5.7.16`。

<center>
{% asset_img slowLog.png %}
</center>

当然，`father`表最大的问题就是`son_id`字段没有建立索引，但是这数据量也并不大啊，`MySQL`应该几十毫秒就能搞定的吧~

什么，你不信？我给大家看看另外一个查询哈：

```SQL
select count(0) FROM test_father WHERE son_id NOT in (SELECT id from test_son);
```

<center>
{% asset_img notIn.png %}
</center>

不是说反向查询对索引的利用并不友好的吗，怎么在没有索引的情况下`NOT IN`比`IN`快这么多？

一开始我尝试使用`explain`查看过第一个语句，发现查询语句`MySQL`优化成了半连接查询，关于连接查询，大家都知道如果能利用索引的话，耗费的时间大概是`A`表满足条件的记录数乘以`B`表索引的搜索时间（基本可以认为是常数时间），而`A`表是哪张表，是由满足条件的记录数决定的，记录数越少的，就会被上拉成为`A`表把数据加载到内存中，然后再拿出每条数据与`B`表关联的列数据去`B`表查询，因为索引的关系，每条记录查询的次数一般不会超过`4`次。

而如果是不能利用索引的情况呢，`A`表还是那张记录数少的表，但是`B`表由于不能利用到索引，所以是需要全表遍历的，而`MySQL`为了减少`IO`的次数，会有一个`join buffer`的数据缓存区缓存`B`表的数据，当然如果你设置的`join buffer`大小太小以至于不能完全装下`B`表所有数据的话，跟每条`A`表数据比对的时候都要轮流替换`join buffer`中`B`表的数据。

<center>
{% asset_img explain1.png %}
</center>

那时候的我就以为这个慢查询的主要原因就在于`MySQL`将它优化成了半连接查询，而我们的`father`表又没有建立好索引，而且当时数据库`join buffer`的参数设置也只是使用了默认值，所以导致这个`SQL`执行非常耗时。

由于各种条件限制，不能立刻给线上数据库加索引或者修改`join buffer`参数大小，于是乎当时的想法就是想办法让`MySQL`不将这个查询优化为连接查询，而最简单又容易转化数据的办法就是反向查询了，也就是图二中展示耗时`0`秒的查询。

我后来也用`explain`查看了`NOT IN`子查询的执行过程，但是没有太大的收获，只能看得出它确实是直接执行了子查询，也就是没有优化成连接查询：

<center>
{% asset_img explain2.png %}
</center>

后面使用了的`optimizer_tracer`观察了它的优化流程，也没能很好的理解这个`NOT IN`之所以“快”的原因。后来在一些学习资料上面阅读到了在一些情况下，`MySQL`会建立不同类型的临时表（依据数据量和参数设置）并自动建立索引这样的机制。

所以后来天真的我得出的结论就是，上面的`NOT IN`语句`MySQL`会给`father`表建立临时表并对`son_id`建立索引，所以使得`IO`次数大概是 ***`son`表记录数乘以常数*** ，大概是连接查询的 ***`son`表记录数乘以`father`表记录数*** 的万分之一左右（两个表满足条件的记录数都是五位数），考虑到`join buffer`的因素，实际`IO`次数比应该是百分之一到千分之一，也就是上面`NOT IN`比`IN`快数百倍的原因。

但是真正的原因真的是这样吗？过了一段时间我又在别的电脑模拟了当时的情景，由于已经过去比较久了，流程记得不太清楚，漏了某些我认为不重要的步骤，结果得出来也十分的意外，相仿的数据量，相似的表结构，简化了的语句，并没有构成慢查询，无论是`IN`还是`NOT IN`，查询时间都是毫秒级别。

这就引起了我强烈的好奇心，不断的复盘，想起了一些自己漏掉的步骤：

 - `son`表并不是一开始就只有一万的数据，而是导入了第三方的百万条数据
 - 由于需求变更，`son`表只保留了最新的一万条左右的数据，其余的数据全被`delete`掉了

整理了漏掉的这些步骤之后，我再重新模拟了多遍当时的场景，终于重现了慢查询，而且发现了`son`表中的一些异常，而我认为这些异常正是导致慢查询的真凶。

大家可以看看关于`son`表的一些统计数据：

<center>
{% asset_img sonSta.png %}
</center>

表里面明明有一万条记录，但是却行数是`0`；`father`表记录和数据都比`son`表多不少，但是`father`表的`Data Length`却只有`1M`多。虽说`innodb`是不准确的统计，但也没有这么这么夸张吧，到底是什么原因导致统计到`0`行数据以及数据长度差异如此大呢，我们可能要深入探索一下`innodb`的数据统计方法~

首先，`innodb`的记录数的计算，是根据算法选取`STATS_SAMPLE_PAGES`个叶子节点页面，然后计算得到选取页面中主键值的平均数，再乘以全部叶子节点数，最后就得出了该表的总记录数。

那么实际上这个计算的方法排除掉设置的那些参数，剩下的变量就只有两个了，其中一个是叶子结点数，另外一个就是选取的页面中的主键平均数。所以导致最后统计的结果为`0`的因素就基本确定了，要么叶子结点数为`0`，要么主键平均数为`0`。叶子节点数为`0`，这个就不需要讨论了，因为没有叶子节点就是一个数据页都没有，根本就没数据，不会跟实际记录产生出入，所以最终的元凶就是选取页面中主键值的平均数了。

相信有的童鞋比较熟悉`innodb`的数据页管理，那么你们肯定知道`innodb`删除数据的时候并不会从数据页中抹除，只是会标记为已删除。所以`son`表的统计记录数为`0`的原因就是因为删除了`99%`的数据，而统计的时候又刚好选取了记录全部都被标记为已删除的数据页，得出的平均数自然也为`0`了。

关于`Data Length`的统计问题我还没搞懂，不过既然已经知道记录数统计误差的来历了，我们也可以先回到一开始的问题。所以这个统计的数据到底是否影响到了查询的效率呢？我可以一脸认真的告诉你，我现在也不知道，不过我想知道，如果你也想知道的话，那咱们再查查资料，或者看看执行过程和代码？
