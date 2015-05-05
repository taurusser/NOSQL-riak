###简要背景介绍
---

Dynamo: Amazon’s Highly Available Key-value Store
-----
<i>
Giuseppe DeCandia, Deniz Hastorun, Madan Jampani, Gunavardhan Kakulapati, Avinash Lakshman, Alex Pilchin, Swaminathan Sivasubramanian, Peter Vosshall and Werner Vogels 
Amazon.com
</i>

出自这篇在2007年，发表在Werner Vogels的博客上的论文,受这个论文的启发，
出现了：
* Cassandra  -> Fackbook [https://github.com/apache/cassandra]
* Voldemort  -> Linkedin  [https://github.com/voldemort/voldemort]
* beansDB     -> 豆瓣网     [https://github.com/douban/beansdb]
* Nuclear       -> 人人网 [闭源]
* Riak             -> Basho      [https://github.com/basho/riak]

####CAP
![cap](https://github.com/taurusser/Stuff/blob/master/riak/img/cap.jpg)

* Consistency（一致性）：即数据一致性，简单的说，就是数据复制到了N台机器，如果有更新，要N机器的数据是一起更新的。
* Availability（可用性）：好的响应性能，此项意思主要就是速度。
* Partition tolerance（分区容错性）：这里是说好的分区方法，体现具体一点，简单地可理解为是节点的可扩展性

<b>定理</b>：任何分布式系统只可同时满足二点，没法三者兼顾。

<b>忠告</b>：架构师不要将精力浪费在如何设计能满足三者的完美分布式系统，而是应该进行取舍。



#Cluster

<b>一致性哈希环</b>

![Consistent Hash's Ring](https://github.com/taurusser/Stuff/blob/master/riak/img/consistent-hashing.png)

<b>智能冗余</b>

![Intelligent Replication](https://github.com/taurusser/Stuff/blob/master/riak/img/riak-data-distribution.png)

###从Riak的NRW看CAP法则

* N：复制的次数；
* R：读数据的最小节点数；
* W：写成功的最小分区数。

这三个数的具体作用是用来灵活地调整Dynamo系统的可用性与一致性。

举个例子来说，如果R=1的话，表示最少只需要去一个节点读数据即可，读到即返回，这时是可用性是很高的，但并不能保证数据的一致性，如果说W同时为1的 话，那可用性更新是最高的一种情况，但这时完全不能保障数据的一致性，因为在可供复制的N个节点里，只需要写成功一次就返回了，也就意味着，有可能在读的这一次并没有真正读到需要的数据（一致性相当的不好）。如果W=R=N=3的话，也就是说，每次写的时候，都保证所有要复制的点都写成功，读的时候也是都读到，这样子读出来的数据一定是正确的，但是其性能大打折扣，也就是说，数据的一致性非常的高，但系统的可用性却非常低了。如果R + W > N能够保证我们“读我们所写”，Dynamo推荐使用322的组合。

针对一些经常可能出现的问题，Riak还提供了一些解决的方法。

* hinted handoff

数据的加入：在一个节点出现临时性故障时，数据会自动进入列表中的下一个节点进行写操作，并标记为handoff数据，在收到通知需要原节点恢复时重新把数据推回去。这能使系统的写入成功大大提升。

* vector-clock

向量时钟来做版本控制：用一个向量（比如说[a,1]表示这个数据在a节点第一次写入）来标记数据的版本，这样在有版本冲突的时候，可以追溯到出现问题的地方。这可以使数据的最终一致成为可能。（Cassandra未用vector clock，而只用client timestamps也达到了同样效果。）

* Merkle tree

提速数据变动时的查找：使用Merkle tree为数据建立索引，只要任意数据有变动，都将快速反馈出来。
![Merkle tree](https://github.com/taurusser/Stuff/blob/master/riak/img/Hash_Tree.svg.png)

* Gossip Protocol

一种通讯协议，目标是让节点与节点之间通信，省略中心节点的存在，使网络达到去中心化。提高系统的可用性。


##Riak
<b>Buckets</b>

![桶](https://github.com/taurusser/Stuff/blob/master/riak/img/buckets.jpg)

<b>Configuration</b>

* allow_mult（一键多值）
* n_val ( the number of copies each objects stored in cluster. default = 3.）
* last_write_wins
* precommit（写之前）
* postcommit （写之后）
* vector clock
* backend （Memory,bitcask, leveldb ）
* strong consistent
* datatypes （ flags, registers, counters, sets, and maps）
....

 in terminal :
curl http://localhost:8098/types/default/props | jq '.'


## Riak vs neo4j

<b>Riak - 主要以key/value键值对为存储对象的分布式数据库</b>

* 值类型
* Solr Search
* Secondary Indexes...

<b>neo4j</b>

* 图形数据库，用于存储、处理网络结构的信息的数据库，尤其是社交网络相关数据


### Features
#### 扩展性
##### Riak

* 扩展  on the fly

轻松从1到N个节点的横向扩展，当一个节点加入到一个族时，Riak族会为其中的每一个物理服务器自动平衡负载，重新分布，这个过程也类似于当一个节点被移出时。

#####  neo4j HA(High Architecture)

![neo4j's HA](https://github.com/taurusser/Stuff/blob/master/riak/img/ha-architecture.svg)

to switch the creation of the GraphDatabaseService from GraphDatabaseFactory to HighlyAvailableGraphDatabaseFactory

#### 数据模型
##### Riak

* semi-document (csv, xml, json) 
e.g. 用户配置， 商品订单 , AJAX 请求返回结构
* Type-based (flag,counts,binary,set,map)

##### neo4j
相对来说，neo4j并不是为一般目的数据的存储系统
* nodes
* relationships
* properties
 Java原型数据类型( int, float, byte, String...) , 或者由这些类型组成的数组
* 关系是自定义类型的 e.g. A KNOWS B; B FRIENDS C;

#### 写冲突
##### Riak
* vector-clock
在一个分布式环境里，常常出现两个或多个session 同时update一个数据时，这种情况可能是客户端更新了cache中的一个对象，又或者网络问题导致客户端的写延迟，Riak通过vector-clock来仲裁冲突。
对于任何时候的写操作，Riak总会返回成功，哪怕是与簇中的部分节点网络不可达，只要客户端能与其中之一的节点可达，则写操作仍然可成功，其代价就是，其后的读操作得解决一个冲突，这种解决策略可配置在文件中，或让客户端动态地选择。

details: http://docs.basho.com/riak/latest/theory/concepts/context/#Vector-Clocks

##### neo4j
相对来说，neo4j可配置为ACID，类似如RDBMS，可在一个隔离的环境中update一个对象。
如果多个事务 update 同一个对象，neo4j会同步这些事务:
lock object -> update object -> release the lock.
如果这些事务之间有数据依赖，那可能会导致死锁。
neo4j的方式是不允许其冲突发生，其代价就是，在写操作不成功时，得重试这个事务，但是注意，
neo4j (ver1.9.4, latest: 2.3.0, stable: 2.2.1) 的事务是只能影响某个物理节点。


#### Querying
##### Riak
###### Riak-ways
* Simple key/value
* Secondary Indexs
* Solr Search
* Customized MapReduce func in Erlang

##### neo4j
neo4j的查询非常屌，可以轻松地找出比如说，你朋友的朋友，通过一个单一规则的递归joins操作，可以从单一条记录，找出相关的N条记录。

###### neo4j-ways
* Tight Integration with Lucene
* JTA (Java Transaction API) compliant XA( Distribution Transaction)
