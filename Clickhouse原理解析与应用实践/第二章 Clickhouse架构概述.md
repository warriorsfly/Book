### 第二章 Clickhouse架构概述

​        随着业务的迅猛增长， Yandex.Metrica目前已经成为世界第三大Web流量分析平台， 每天处理超过200亿个跟踪事件。 能够拥有如此惊人的体量， 在他背后提供支撑的Clickhouse功不可没。Clickhouse已经为Yandex.Metrica存储了超过20万亿行数据， 90%的自定义查询能够在1秒内返回， 其集群规模也超过了400台服务器。 虽然Clickhouse起初只是为了Yandex.Metrica而研发的，但由于它的出众性能，目前也被广泛应用于Yandex内部其他数十个产品上。

​     初识ClickHouse的时候， 我曾产生这样的感觉： 它仿佛违背了物理定律， 没有任何缺点， 是一个不真实的存在。 一款高性能、高可用OLAP数据库的一切诉求， ClickHouse似乎都能满足， 这股神秘的气息引起了我极大的好奇。 但是刚从Hadoop生态转向ClickHouse的时候， 我曾有诸多的不适应， 因为他和我们往常使用的技术“性格：迥然不同。 如果把数据库比作骑车， 那么ClickHouse俨然就是一辆手动档的赛车。 它在很多方面不像其他系统那样高度自动化。 ClickHouse的一些概念也与我们通常理解的有所不同，特别是在分片和副本方面， 有些时候数据的分析甚至需要手动完成。 在进一步深入使用ClickHouse之后， 我渐渐的理解了这些设计的目的。 有些看似不够自动化的设计， 反过来却在试哟给你周哥你带来了极大的灵活性。 与Hadoop生态的其他数据库相比， ClickHouse更像一款“传统”MPP架构的数据库， 它没有采用Hadoop生态中常用的主从架构， 而是使用了多主对等网络结构， 同时它也是基于关系模型的ROLAP方案。 这一章就让我们抽丝剥茧， 看看ClickHouse都有哪些核心特性。



#### 2.1 ClickHouse的核心特性

​        ClickHouse是一款MPP架构的列式存储数据库， 但MPP和列式存储并不是“稀罕：的设计。 拥有这类架构的其他数据库产品也有很多， 但是为什么偏偏只有ClickHouse的性能如此出众呢？ 通过上一章的介绍， 我们知道了ClickHouse发展至今的演进过程。 它一共经历了四个阶段， 每一次阶段演进， 相比之前都进一步取其精华去其糟粕。 可以说ClickHouse机去了个假技术的精髓， 将每一个细节都做到了极致。 接下来将介绍ClickHouse的一些核心特性， 正是这些特性形成的合力使得ClickHouse如此优秀。

##### 2.1.1 完备的DBMS功能

​		ClickHouse拥有完备的管理功能， 所以它称得上是一个DBMS（Database Management System， 数据库管理系统）， 而不仅是一个数据库。 作为一个DBMS， 它具备了一些基本功能， 如下所示。

- DDL（数据定义语言）： 可以动态地创建、修改或者删除数据库、 表和视图， 而无需重启服务。
- DML（数据操作语言）： 可以动态查询、 插入、 修改或者删除数据。
- 权限控制： 可以按照用户力度设置数据库或者表的操作权限， 保障数据的安全性。
- 数据备份与恢复： 提供了数据备份到出与导入恢复机制， 满足生产环境的要求。
- 分布式管理： 提供集群模式， 能够自动管理多个数据库节点。

这里只列举了一些最具代表性的功能， 但已然足以表明为什么ClickHouse称的上是DBMS了。

##### 2.1.2 列式存储与数据压缩

​		列式存储和数据压缩， 对于一款高性能数据库来说是必不可少的特性。 一个非常流行的观点认为， 如果你想让查询变得更快， 最简单且有效的方法是减少数据扫描范围和数据传输时的大小， 而列式存储和数据压缩就可以帮助我们实现上述两点。 列式存储和数据压缩通常是伴生的， 因为一般来说列式存储室数据压缩的前提。

​		按列存储与氨航存储相比，前者可以有效减少查询时所需扫描的数据量，这一点可以用一个示例说明。 假设一张数据表拥有50个字段A1～A50，医技100行数据。 现在需要查询前5个字段并进行数据分析， 则可以用如下SQL实现：

```sql
SELECT A1, A2, A3, A4, A5 FROM A
```

​		如果数据按行存储，数据库首先会逐行扫描， 并获取每行数据的所有50个字段，再从每一行数据中返回A1～A5这5个字段。不难发现，尽管只需要前面的5个字段，但由于数据是按照行进行组织的，实际上还是扫描了所有的字段。如果数据按照列存储，就不会发生这样的问题。由于数据按列组织，数据库可以直接获取A1～A5这5列的数据，从而避免了多余的数据扫描。

​		按列存储相比按行存储的另一个优势是对数据压缩的友好性。同样可以用一个示例简单说明压缩的本质是什么。假设有两个字符串abcdefghi和bcdefghi，现在对它们进行压缩，如下所示：

```
压缩前:abcdefghi_bcdefghi
压缩后:abcdefghi_(9,8)
```

可以看到，压缩的本质是按照一定步长对数据进行匹配扫描，当发现重复部分的时候就进行编码转换。例如上述示例中的（9，8），表示如果从下划线开始向前移动9个字节，会匹配到8个字节长度的重复项，即这里的bcdefghi。

​		真实的压缩算法自然比这个示例更为复杂，但压缩的实质就是如此。数据中的重复项越多，则压缩率越高；压缩率越高，则数据题量越小；而数据题量越小，则数据在网络中的传输越快，对网络带宽和磁盘IO的压力就越小。既然如此，那怎样的数据最有可能具备重复的特性呢？答案是属于同一个列字段的数据，因为他们拥有相同数据类型和现实语义

