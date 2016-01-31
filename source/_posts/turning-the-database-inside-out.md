title: 数据库给流计算带来的启发 - 《Turning the database inside-out》
date: 2016-01-17 16:55:37
tags:
	- architecture
	- 流计算
	- 笔记
---
原文链接：http://www.confluent.io/blog/turning-the-database-inside-out-with-apache-samza/

# 前言
传统数据库发展了几十年，着实有不少思路和技术能让我们细细品味。本文是[ Martin Kleppmann](http://martin.kleppmann.com/)在[Strange Loop 2014](https://thestrangeloop.com/archive/2014)上的一个精彩发言，从几个基本的数据库技术出发，逐步延伸至目前火热的流计算思想，令人深省。

# 什么是数据库
不管是什么品牌什么类型的数据库，不管是oracle，MySql亦或是现在流行的NoSql，基本都可以使用下面这句话来概括：
```
Databases are global, shared, mutable state
```
为什么说它使一个全局共享的可变的状态呢？我们可以来看一下一个典型的数据库应用的场景：
![](images/turning-the-database-inside-out-1.png)
一个数据库应用主要由客户端，后台以及DB组成。客户端发起请求，由后台接受请求并加以一些自定义逻辑，最终将数据持久化至DB或者从DB获取需要的数据。一般而言我们会把后台系统设计成为“无状态”系统，这样做有很多好处，比如你可以更容易的扩展后台，提高并发，使负载更均衡等。对于客户端而言，使用无状态的HTTP协议也是一种很流行的做法，毕竟把状态存储在客户端八成也不是什么特别好的主意。这样一来，系统所需的一些“状态信息“，就只能存在DB里了。这样做了以后，数据库就成了一个海量的全局共享的状态存储，随之而来还有并发，锁竞争，死锁等各种各样的问题。
那么我们为什么一定要这样去设计我们的系统呢？在回答这个问题之前，我们先来看四个看起来与这个问题并没有什么关系的东西。
<!-- more -->
# 1. Replication
对于数据库来说，最典型技术的应该是主从同步。由master来接受所有的写请求，通过一定的机制来把这些写请求转发到所有的follower。至于具体怎么实现，有不少可选的方案。一种是把原始的写请求原封不动的转发给follower，又或者是把leader的write-ahead log转发给follower。第三种选择是，我们把所有的写请求转化成一种逻辑上的事件，把这些事件转发给所有的follower。比如这样：在T这个时间点，ID为A的这条记录的B字段的值从C变成了D。这样一来，所有的写请求都被我们转化成了一条条不可变的事件，而且这个事件一定是成立的。就算你撤销了你的修改，比如把值又改回去了，也不能改变你曾经把这个值改过的事实。现在很多架构都喜欢把原始数据抽象成为不可变的事件，比如我写的前两篇笔记[Lambda Architecture](/2016/01/09/lambda-architecture/)和[Kappa Architecture](/2016/01/16/kappa-architecture/)。总而言之，不管你是怎么使用数据库，你增加记录也好，修改记录也好，数据库在内部实现中很有可能就利用这种不可变的事件流来达到主从同步的目的。我们再来看下一个东西：

# 2. 二级索引
对于一个拥有多个字段的数据库表来说，我们可以在某些需要的字段上建立二级索引来达到我们快速定位某一条符合条件的记录的目的。简单来说，在我们调用了CREATE INDEX这个命令之后，数据库会对这个表进行全量扫描，对每个字段的索引生成一种新的辅助数据，通常是key-value结构。其中的key是索引字段的内容，value就是整个记录。这个简单的图可以清楚的描述这个过程：
![](images/turning-the-database-inside-out-index1.png)
这看起来很简单也很直白，但有几个特点是我们不能忽视的：

1. 二级索引是由数据库自身维护的，我们只需要告诉数据库我们需要哪些二级索引就行了。
2. 二级索引并没有添加新的信息到我们的原始数据上，它只是我们现有的那些数据的另外一种表现方式罢了（换句话说，如果我们删除了索引，对数据不会有任何影响）。索引是一种重复的信息，而且是可以从原始数据转变而来。

本质上，二级索引就是原始数据的一种transformation，我们只需要定义我们需要什么样的转化，其他的就交由数据库来完成就行了。
![](images/turning-the-database-inside-out-index2.png)
而且，这种转变是可持续的，它不是只在我们定义索引的时候做一次，而是在之后所有的添加删除操作发生之时，都会自动的更新和维护这份索引。

# 3. Caching
这里所说的caching主要指应用层面的，典型的如下图所示：
![](images/turning-the-database-inside-out-caching.png)
虽然大部分caching的核心实现都是这么简单的几句话，但是我们锁需要面对的问题可远远不止这些。

1. cache什么时候失效
2. cache一致性，race condition
3. 冷启动时没有cache

这几点不管哪个展开都能说上一箩筐，而且除了第三点之外，还都没有特别完美的解决办法。要么就是弄一些很复杂的逻辑，要么就是让用户在使用上进行取舍，是要比较强的一致性，还是能容忍短暂的不一致换取高可用。
说到这我们不得不把caching拿出来和上面介绍的二级索引进行一下对比，因为这两者并没有本质上的区别，都是在原始数据之外的一个额外辅助结构，都是由原始数据经过一些转化而来。具体来讲，二级索引将记录的某一个字段拿出来当成key，把原始记录作为value。而cache是对原始记录进行一些操作之后再保存下来，这些操作可能比较简单，也可能很复杂。但是这都不要紧，这不影响cache总是能从原始数据进行重新计算而获得这个事实。
![](images/turning-the-database-inside-out-caching-trans.png)

# 4. Materialized views
创建一个materialized views和创建一个普通的view的语法基本相同，不同的是，如果你创建的是一个普通的view，那么数据库只会在你对这个view进行操作时根据view的定义来改写你的query，而materialized views则不同。数据库会真的根据materialized views的定义产生一个实体的表来表示这个view，每次你对view的操作，会落在这个真实存在的表上。所以对一些开销比较大的view，使用materialized views的性能会更好一些，因为这些开销比较大的操作数据库已经帮你事先做掉了，你需要的只是过来读取你的结果而已。如果用相对比较数学的方式来描述materialized view，将会是下面这样：
![](images/turning-the-database-inside-out-materialized-view.png)
看着有没有很眼熟？是不是很像上面的Cache？其实他们俩还真就是一个东西，唯一的不同点是由谁来管理cache的有效性。Cache完全由用户自己来管理，比如何时失效，如何淘汰等等，而对于Materialized views你只需要告诉数据库你希望这个view长成什么样子，数据库就会帮你建立并且一直维护下去。就算原表的数据发生变化，数据库也会将这个变化的影响带到这个view上。但有一点Cache比较占优势的是，你可以在数据进入cache前有很多自定义的逻辑，这点数据库并不容易做到，虽然目前有些数据库可以自定义一些`JavaScript`的UDF，但把这些用户逻辑放在数据库内部终究不是特别好的做法。

由以上的介绍已经基本可以看出，这四种不同的技术有着不同的表现：
![](images/turning-the-database-inside-out-25.png)
Replication和二级索引不管是从设计还是实现来看，都做的比较好，也都已经是相对成熟的东西。而用户端的Cache一般来讲就是个大泥潭，你多多少少会碰到各种乱七八糟的问题，相比之下materialized views则有利有弊，好的是它的设计比较简单，能做到自更新，而不好的是它给数据库带来了额外的负担和不稳定因素。

# 重新思考Materialized views
从上面的总结可以看到，materialized views已经很接近一个比较完美的Cache，只是缺最后那么一小点，我们能不能重新设计和思考materialized views，让它变成一个可扩展的通用的Cache机制？在此之前，我们先来换一个问题，假如没有数据库的历史包袱，我们会怎么样来设计我们现在这些基于数据库的应用程序？
![](images/turning-the-database-inside-out-rethink.png)
基于分布式和可扩展性的考虑，首先我们得有一个leader，由它来接收所有的写请求，并把这些写请求同步到所有的follower，换言之我们在leader和follower之间建立了一条流。在一个传统数据库的架构里，应用的开发者是不需要关心这个流的，它属于数据库的“实现细节”，这样做是合理的，这有利于数据库的抽象，也是数据库库能流行这么多年的其中一个原因。但是，如果我们把这个流从“实现细节”这个范畴中拿出来，让它成为架构中对外可见的一部分会怎么样？想象一下现在的系统会长成什么样：
![](images/turning-the-database-inside-out-26.png)
你可以把这个流称之为“事务日志”，或者“事件流”，它由一些不可变的事件组成。所有的写请求只是增加一条不可变的事件。有很多这种事件队列系统供你选择，比如[Apache Kafka](http://kafka.apache.org/)。它具有良好的可扩展性，能轻松达到每秒上百万次的写入性能。现在写请求已经非常的高效并且可扩展，那么读呢？从一个流里面读取数据显然不是一个很好的做法。解法就是：给这条流创建materialized views。这些views就像我们上文讨论到的二级索引一样，从流里面的数据获取需要的信息，把数据转变成一种对读取更加友好的数据结构。这些views就像流里部分数据的cache一样，我们随时可以根据自己的需要进行重建。你可以创建各式各样的view，比如一个key-value存储，一个支持全文检索的索引，一个图索引等等。
如果你使用Kafka作为log的实现，那么materialized views该怎么来实现呢？这时候就可以用到[Apache Samza](http://samza.apache.org/)了（广告时间，巴拉巴拉巴拉）。所有的读请求都是对samza维护的一个或者多个materialized views进行。这就体现出了和传统数据库最大的不同：当你写的时候，你写的不是数据库本身，而是一个事件日志，而且有一些显式的自定义处理过程对这些日志进行处理，并把结果更新到materialized views里。
读和写的各自分离正是这种系统的关键点所在。所有的写已经变的非常简单和高效，我们不需要去修改一个全局的可变状态，因为这种行为通常比较昂贵并且伴随着各种一致性的问题，我们只需要在一个自增的日志流里新增一条日志即可。这种做法其实并不是非常新颖，已经有不少的成熟技术是这么做的了，而且我前面介绍到的[Lambda Architecture](/2016/01/09/lambda-architecture/)也是一个比较成熟的做法。
对于读来讲，我们首先要减少对数据库进行读取的想法，相反的更应该去想象我们是在消费一个流。但是传统数据库那种做法并不是完全的没用了，在某些OLAP场景下，我们需要运行一些ad-hoc的query，这种我们一般不能事先建立所有需要的materialized views，所以只能把所有的数据存储下来，然后想办法让每次的查询变的更快一些。但在大部分OLTP的场景下，当我们需要cache的时候，这种materialized views的想法会起到很大的作用。对比cache和materialized views，它们主要有以下几点不同：

1. 在使用materialized views时，我们需要一个把针对写进行优化的原始数据，转变为针对读进行优化的数据的一个处理层。这个处理层是一个相对独立的程序，我们可以对它单独进行开发，测试，监控等。而对于cache来讲，cache管理的代码会和你其他的代码逻辑搅和在一块，很容易引入bug并且难以理解。
2. cache只有在miss的时候进行填充，所以对一个数据的第一次请求总是特别慢。但是materialized views不一样，它会进行预先计算，也就是说对它来讲没有cache miss这种事情。当一个数据miss的时候，那么它就是不存在的，你不需要再到底下的数据库里进行查找。
3. 创建一个新的view变的很简单。如果你想对数据进行其他逻辑的计算，你要做的只是创建一个新的计算任务，使用新的逻辑，并且产生新的结果。你可以同时消费新老两种数据，或者在某个时候丢弃掉老的计算结果，这一切都很自然，不需要任何不必要的中断。

# 总结
作者提出的这种思路听起来还是比较“激进的”，而且用户需要做大量的修改才能把架构转变成现在这个样子。那么这种架构真正的好处在哪里呢？

## 好处1：Better data
- 分析更容易（所有的变更都被保存，而不是对一个全局状态的改变）
- 读和写分离（分别对读和写进行优化）
- 恢复能力更强（对历史事件进行重做）

## 好处2：Fully precomputed caches
- 没有锁竞争，没有复杂的cache失效逻辑
- 没有冷启动的影响，没有cache命中和失效的分别
- 更好的解耦和分离（增加一个新的处理层来试验新的想法，不影响现有逻辑）

## 好处3：Streams everywhere
- 建立原始数据到终端用户的流
- 从Request/Response变更为Subscribe/Notify的模式

这篇文章从传统数据库，以及围绕着传统数据库的cache出发，总结了这种做法的本质是维护一个全局共享的可变状态以及这种做法在规模变大后的瓶颈和复杂度。接着讨论了四个数据库的核心技术：replication，二级索引，cache和materialized views，让我们自然而然的想到了流式的事件日志的思路。最后，讨论了基于materialized views和流计算的思想如何能让我们程序的架构变得更简单，扩展性和容错性变得更好。
