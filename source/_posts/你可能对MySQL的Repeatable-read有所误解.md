---
title: 你可能对MySQL的Repeatable read有所误解
date: 2019-02-20 21:23:13
tags: MySQL
categories: 探究
---

>有些东西，你深究下去，才会发现原来你的认知一直是错的。

相信大家对`MySQL`事务的隔离级别都不陌生：

 - 读未提交 (`Read uncommitted`)
 - 读已提交 (`Read committed`)
 - 可重复读 (`Repeatable read`)
 - 串行化 (`Serializable`)

`MySQL`默认的事务隔离级别为可重复读 (`Repeatable read`)。

也许大家都见过这个在各种博客或文章里出现频率极高的表格：
<table>
<tbody>
<tr>
<td>事务隔离级别</td>
<td>脏读</td>
<td>不可重复读</td>
<td>幻读</td>
</tr>
<tr>
<td>读未提交</td>
<td>是</td>
<td>是</td>
<td>是</td>
</tr>
<tr>
<td>读已提交</td>
<td>否</td>
<td>是</td>
<td>是</td>
</tr>
<tr>
<td>可重复读</td>
<td>否</td>
<td>否</td>
<td>是</td>
</tr>
<tr>
<td>串行化</td>
<td>否</td>
<td>否</td>
<td>否</td>
</tr>
</tbody>
</table>

也许大家看到的样式不是这样，但是我肯定它们的灵魂都是一样的。当然上面这个表格是对的，它描述的是`SQL`的标准，只是在`MySQL`的`MVCC`机制下，会让人容易出现理解上面的误差。

上面确实都是废话，这篇文章就一个重点，那就是想说明`MySQL`在可重复读这个隔离级别下，让幻读出现所必需要的条件。我们在准确理解了`MySQL`可重复读之后，就可以更好的优化我们的`SQL`代码了。

如果有的童鞋早就知道了，请不要大喊出来，我想你们也不忍心让我知道自己到底愚蠢了多久。:joy:

下面我们实验完之后，如果有童鞋觉得我的观点是错误的话，那大家一起来快活的讨论吧。

由于不可重复读和幻读的概念很容易搞混，所以实验之前我们先回顾一下：

 - 不可重复读： 多次读取一条记录, 发现其中的数据与事务内上一次的读取不一致，主要侧重其它事务的更新影响了本事务的读取内容。
 - 幻读： 多次读取一个范围内的记录, 发现结果不一致，主要针对的是其它事务的插入和删除操作。

我要建的表结构如下：

```
CREATE TABLE `test` (
  `id` int(11) NOT NULL,
  `content` varchar(255) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

首先连接上你的数据库，查看当前的隔离级别：

```sql
SELECT @@tx_isolation;
```

确定是`REPEATABLE-READ`之后，我们再新开一个窗口建立另一个连接。然后在两个窗口分别运行：

```sql
-- 开启事务A
START TRANSACTION;
```

```sql
-- 开启事务B
START TRANSACTION;
```

都开启了事务后，咱们在窗口`A`查询：
```sql
-- 事务A
SELECT * FROM test;
```
结果集当然为空，因为咱们表还没有任何的数据。

接下来我们就要去窗口`B`进行插入数据操作并提交事务：
```sql
-- 事务B
INSERT INTO test(content) VALUES('B插入内容1');
COMMIT;
-- 事务B结束
```

然后再返回窗口`A`重新运行：
```sql
-- 事务A
SELECT * FROM test;
```

依然是没有任何数据，接着我们对`A`的事务进行提交并重新查询：
```sql
-- 事务A
COMMIT;
-- 事务A已结束
SELECT * FROM test;
```

终于在窗口`A`的结果集中出现了在事务`B`中插入的内容：
<table>
<tbody>
<tr>
<td>id</td>
<td>content</td>
</tr>
<tr>
<td>1</td>
<td>B插入内容1</td>
</tr>
</tbody>
</table>

在上面这个栗子中并没有出现幻读，直接的原因就是**可重复读中读取的是`MVCC`机制所提供的事务级的快照**，所以别的事务对数据的修改并不会直接影响到当前事务的读取的数据。

然而经过我的细心研究，发现在可重复读级别下，幻读终究还是会出现的，它只不过是有点害羞罢了。

相信大家还记得当前表中的数据，如果不记得了，我们来查一下：

```sql
-- 开启事务A
START TRANSACTION;
SELECT * FROM test;
```

得到结果集：
<table>
<tbody>
<tr>
<td>id</td>
<td>content</td>
</tr>
<tr>
<td>1</td>
<td>B插入内容1</td>
</tr>
</tbody>
</table>

接下来到新的窗口开启事务`B`：

```sql
-- 开启事务B
START TRANSACTION;
INSERT INTO test(content) VALUES('B插入内容2');
COMMIT;
-- 事务B结束
```

返回到事务`A`我们重新查询：
```sql
-- 事务A
SELECT * FROM test;
```
得到结果集没有改变：
<table>
<tbody>
<tr>
<td>id</td>
<td>content</td>
</tr>
<tr>
<td>1</td>
<td>B插入内容1</td>
</tr>
</tbody>
</table>

接下来我们来一波骚操作：
```sql
-- 事务A
UPDATE test SET content='A更新内容';
```

执行之后，清楚的看到了反馈：
```
[SQL]UPDATE test SET content='A更新内容';
Affected rows: 2
Time: 0.000s
```

我刚刚查到表只有一行数据，怎么我更新了两行？那我们再查查看：

```sql
-- 事务A
SELECT * FROM test;
```

咦？得到的结果集有两条记录：
<table>
<tbody>
<tr>
<td>id</td>
<td>content</td>
</tr>
<tr>
<td>1</td>
<td>A更新内容</td>
</tr>
<td>2</td>
<td>A更新内容</td>
</tr>
</tbody>
</table>

相信各位聪明的童鞋读到这里都知道是什么回事了，毕竟我们上帝视角看着事务`A`和`B`，都看到这里了还装什么傻。没错，使可重复读级别下出现幻读的必要条件，就是两个事务中都有`DML`操作，而且是恰到好处的`DML`操作，否则你们的修改的内容和条件（可以理解为修改的数据所属的范围）没有一点交集的话，幻读也是不会出现的。

总的来说，在事务`A`中也加入`DML`操作的话，会发生明明能读取到，却操作不到的奇怪现象，或者会出现读取的数据中没这条记录，但是最后`DML`操作的时候却把这条记录也一起修改了的情况。我认为`Repeatable read`虽然是能在`DQL`操作中防止了幻读和不可重复读的情况出现，但是却在`DML`操作中会引入其他一些奇怪的现象。

有的童鞋看到这里可能会有一种被标题党骗了的感觉，这不是说了跟没说一样嘛，可重复读级别还是会有幻读出现啊，到最后我们还不是需要用串行化？

但是这篇文章真的没有意义吗？在你的项目里面，为了避免幻读，是直接使用串行化？还是依然使用可重复读，只是需要在那些可能会产生幻读的地方加上`GAP`锁？

如果你的回答是“都可以，怎么方便怎么来”的话，那我觉得，确实没意义。

```sql
-- 事务A
COMMIT;
-- 事务A结束
```

完。