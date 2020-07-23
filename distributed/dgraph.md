# Dgraph: 同步复制,事务,分布式图数据库

- Version: 0.8 Last Updated: February 23, 2020

## 摘要

​		Dgraph 是一个分布式图数据库,它提供水平伸缩,全局范围内的ACID事务,低延迟的任意深度的join操作,同步复制,高可用性,故障恢复的能力。对于OLTP工作负载,Dgraph以一种对join与遍历友好的方式切分和存储数据,同时兼顾数据检索与聚合。Dgraph的独特之处在于无论集群或者结果集有多大,都可以在固定数量的网络调用中(通常只有一次)提供低延迟的任意深度的join操作.

## 简介

​		分布式系统或者数据库通常会遇到深度join的问题.也就是说,随着查询内关系遍历的次数的增加(在充分分片的数据集内),所需的网络调用次数也会增加,这通常是由于基于实体的数据分片,实体随机的(相对于查询来看)分布在所有的实体与属性的服务器上,这种方式在图查询的中间过程中收到高扇出结果集的影响,使他们跨集群的方式进行广播以对实体进行连接。因此一次图查询会导致网络请求的广播,故随着集群节点数的增长查询延迟也会增大。

​		Dgraph 是一个原生分布式的图数据库,他是唯一一个水平可伸缩的原生图数据库,支持整个集群范围内的AICD事务,事实上,Dgraph 是第一个经过Jepsen[5]事务一致性测试的图数据库。

​		随着数据量或者服务器数量的变化,Dgraph自动将数据分片,并将分片在所有服务器上负载均衡。它还支持raft协议所具有的同步复制,该协议可保证在故障转移期间查询的可用性。

​		Dgraph 以独特的分片机制解决了连接深度的问题。与大多数系统不同的是Dgraph按关系进行切分,而不是按实体进行切分。Dgraph对数据进行分片的灵感来自于Google[1]的研究,该研究表明,查询的总延迟大于最慢组件的延迟。查询要执行的服务器越多,查询延迟就越慢.通过基于关系分片,Dgraph可以在一次网络调用中执行连接或遍历(如果第一次调用延迟较高,则会尝试使用副本的备用网络调用),而与列簇或者输入实体集的大小无关。也就是说 Dgraph在一个没有网络广播,中心节点仲裁数据的情况下执行任意深度的连接查询。这使得查询速度更快,延迟更低且可预测。

## Dgraph体系结构

​			Dgraph 由`Zero` 和 `Alpha` 两种节点角色组成.`Zero`服务器组成专有的Raft服务组第0号,`Alpha`服务器可以组成各个Raft小组,分别为第一组,第二组及以上.每个小组内遵循Raft协议的要求,集群节点的数量可以是1,3,5等奇数个.对集群内所有的状态变更操作都会基于Raft共识算法来保证一致性,按raft的日志顺序应用于每个小组的领导者与追随者。

​			Zero 存储和传播有关集群的元数据,而Alpha存储用户数据.具体来说,Zero负责成员的身份信息,该信息跟踪每个Alpha服务器正在服务的group,以及在集群内通信的内部IP,正在服务的分片等信息。zeros不跟踪Alpha的健康状态并对其进行操作,这被认为是运维人员的操作,使用这些信息zero可以告诉新的alpha加入并服务于现有的group,或者新建一个group。

​			整个集群的元数据配置,会由zero传播给所有alpha节点。alpha通过这些配置信息,路由或者命中查询(或突变)。集群中的所有节点彼此连接,产生2*N^2个连接(N位集群节点的数量)。如下图[1]所示:

![图1 Dgraph架构,由一个zero group和多个alpha group组成,每个组都是由一个或多个成员节点组成的raft小组](/Users/sunquan/Library/Application Support/typora-user-images/image-20200713125801688.png)

​			 这种联系的使用取决于他们之间的关系,例如一个raft主从之间会有心跳和数据流(100毫秒),而一个alpha只会在处理查询和突变时与另一个组的alpha节点交换信息.每个开发连接都有轻量级的健康检查,以避免在目标服务器的通信时响应失败,Alpah与Zero都公开了一个GRPC端口用于内部通信,此外Alpha还使用外部端口用于与GRPC外部官方客户端进行通信。				Zero还运行着一个oracle,为集群中的事务分配单调递增的逻辑时间戳(与系统时间无关),Zero组的领导者通常会通过Raft协议出租一块时间戳带宽,然后在没有任何进一步的协调的情况下严格的从存储器中为时间戳请求提供服务,Zero oracle 用于跟踪协助事务提交的附加内容,这将在第五节详细说明。
​				Zero从每个组的raft领导者那里获得每个组中数据大小的信息,它用这些信息来做出关于分片移动的决策,这将在2.4节中详细说明.

### 数据的格式

​			Dgraph支持JSON或者RDF NQuad格式输入数据。Dgraph将Json映射分解成更小的块,每个Json 的键值对等价于RDF三元组的记录.解析RDF或JSON时数据直接转换为内部的Protobuf协议,并且不再二者之间来回转换.

```json
// JSON
{
 "uid": "0xab",
 "type": "Astronaut",
 "name"  : "Mark Watney",
 "birth" : "2005/01/02",
 "follower": { "uid": "0xbc", ... },
}
// RDF
<0xab> <type>  "Astronaut" .
<0xab> <name>  "Mark Watney" 
<0xab> <birth> "2005/01/02" .
<0xab> <follower> <0xbc> .
```

​		 三元组通常表现为<主-谓-宾>或者<主-谓-值>,主语是一个节点,谓语表示关系,宾语可以是另一个节点或者是某个原始类型的值,谓语表示一种有方向的边,从主语指向宾语或者指向一个值.在上面的例子中`name`三元组是<主-谓-值>,而follower三元组则是<主-谓-宾>的类型。Dgraph 在处理这两种类型的记录时没有区别,在内部这被认为是一种记录单位,一个典型的json记录将被拆分为多个这样的记录单位.

​		 可以使用GraphQL[7]从Dgraph中检索数据,GraphQL的修改版本称之为GraphQL+-,具有大部分GraphQL相同的属性,并且添加了很多对数据库操作有价值的特性,如查询变量,函数,块,有关查询语言是如何产生的以及GraphQL和GraphQL+之间的区别的更多信息，可以在这篇博客文章中找到[4]

​		 如第2节所述,Dgraph中的所有内部和外部通信都通过GRPC和协议缓冲区运行。Dgraph还公开HTTP端点,支持异构客户端的使用。HTTP端点提供的功能与Dgraph公开的api是等价的.

​        根据GraphQL的规范,HTTP端口和GRPC端口查询响应的格式都是JSON的。

### 数据的存储

​		Dgraph数据存储在称为Badger [3]的可嵌入的键值数据库中,用于在磁盘读写数据。 Badger是基于LSM树的设计,但与其他方法不同之处在于,Badger会选择性地将Key与Value分开存储,从而得到更小的LSM树,以便于降低读写的放大,运行的各种基准测试表明Badger提供了比其他基于LSM的DB相同或更快的写入速度,同时提供了与基于B +Tree的DB相当的读取延迟(与LSM树相比,它们提供了更快的读取速度)

​		如上所述,具有相同谓词的所有记录形成一个分片。 在分片中,共享同一<主语-谓词>的记录在Badger中被分组并压缩为一个单一的KV记录.该值称为一个postingList,这是搜索引擎中常用的术语,指的是包含搜索词的文档ID的排序列表。 ,postingList作为值存储在Badger中,键由主语和谓语组成.

```bash
<0x01> <follower> <0xab> .
<0x01> <follower> <0xbc> .
<0x01> <follower> <0xcd> ....
key = <follower, 0x01>
value = <0xab, 0xbc, 0xcd, ...>
```

​		在 Dgraph 的所有节点(主语或宾语)都被分配了一个全球唯一的id,称为`uid`。一个uid存储为64位无符号整数(uint64),以便于 Go 语言在代码库中进行高效的本地处理。Zero负责分发Alpha所需的uid,并且保证像生成事务时间戳一样的单调递增的方式处理(第2节:预分配策略)。已经被分配的UID不会再次被分配.因此Dgraph中的每个节点都可以被唯一的整数引用。	

​		对象值存储在Postings中,每个Postings都有一个integer ID。当Postings持有一个对象时,id就是分配给该对象的uid。 当Postings保存一个原始类型值时,值的uid是根据谓词的模式确定的, 如果谓词允许多个值，则该值的integer ID将是该值的指纹。 如果谓词用Language存储值，则整数id将是语言标签的指纹。 否则，整数id将被设置为最大uint64(2^64-1)的值,uid和整数id从未设置为零。

​		值可以是许多受支持的数据类型之一：整型、浮点型、字符串、日期时间、地理类型等。数据被转换为二进制格式，并与有关原始类型的信息一起存储在Postings中。 Postings还可以包含多个Facets。 Facets是边上的键值标签,可将其视为附件。

​		在谓词指向宾语的常见情况下，PostingsList将主要由uid组成, 这些都是通过执行整数压缩进行优化的。 这些UID被分组为256个整数的块(可配置)，其中每个块都有一个基本BLOB和一个二进制BLOB。 BLOB是通过取当前值与上一个值的差值并以使用可变长整数编码的字节为单位存储差值来生成的。 这样产生的数据压缩比为10。在进行交叉点时，我们可以使用这些块进行二进制搜索或块跳转，以避免对所有块进行解码。 排序整数编码是一个热门的研究课题,在性能方面有很大的优化空间,目前正在进行使用RoaringBitmap[10]来组织该数据.

![图2:存储在组varint-encoded-block中的发布列表结构](/Users/sunquan/Library/Application Support/typora-user-images/image-20200715133847024.png)

​		多亏了使用这样的技术,一次边缘遍历只对应于一次Badger查询。例如,查找X的所有关注者的列表将涉及到查找<Follower,X> key，该键将给出包含所有关注者的uid的Postings List。 可以进行进一步的查找,以获得关注者发布的PostingList。 通过进行两次查找,然后相交<Follower，X>和<Follower，Y>的排序后的INT列表,可以找到X和Y之间的共同关注者。 注意,分布式连接和(基于对象的)遍历只需要通过网络传输UID,这也是非常高效的。 所有这些都允许Dgraph非常高效地执行这些操作,而不会向典型的`select * from table where X=Y`方式的查询

​		这种类型的数据存储在join和range方面具有优势,但也带来了高扇出的额外问题。 如果有太多记录具有相同的<SUBJECT,PRESCEATE> key，则整个PostingList可能会增长到无法维持的大小。 这通常只是边的问题(对于值来说不是那么大的问题)。 我们通过在其磁盘大小达到特定阈值时对PostingList进行二进制拆分来解决此问题,分割PostingList将作为多个key存储在Badger中,并进行优化以避免在操作需要时才进行拆分合并, 尽管存在存储差异,但PostingList继续通过API提供与未拆分的PostingList相同的排序迭代.

### 数据的分片

​	  虽然Dgraph具有很多NoSQL和分布式SQL数据库的许多功能,但它在处理记录的方式上有很大的不同.在其他数据库中，行或文档将是最小的存储单元(保证存储在一起)，而分片可以像将这些最小单元组成大小相等的页或者块一样简单。Dgraph的最小记录单元是一个三元组(主语-谓语-宾语，如下所述),每个谓语的整体构成一个分片。 换句话说，Dgraph逻辑上使用相同的谓词对所有三元组进行分组，并将它们视为一个分片。 然后为每个分片分配一个组(1..N)，然后该组可以由服务于该组的所有Alpha提供服务，如第2节中所述。此数据分片模型允许Dgraph在单个网络调用中执行完全join,而无需调用者跨服务器获取任何数据。 这与磁盘上以独特方式分组的记录相结合，将通常由昂贵的磁盘迭代执行的操作转换为更少、更便宜的磁盘寻道，使得Dgraph内部工作相当有效。为了进一步阐述这一点，考虑一个包含关于人们居住的地方(谓语：“Lives-in”)和他们吃什么(谓语：“Eats”)的数据集。 数据可能看起来像这样：	

```bash
<person-a> <lives-in> <sf> .
<person-a> <eats> <sushi> .
<person-a> <eats> <indian>.
...
<person-b> <lives-in> <nyc> .
<person-b> <eats> <thai> .
```

​		在本例中，我们将有两个分片lives-in 和 eats。 作为最坏情况的总和，其中群集非常大，以至于每个碎片都驻留在单独的服务器上。 对于询问[People Who Living in SF and Eat Sushi]的查询,Dgraph将对包含Lives-in的服务器执行一次网络调用，并对居住在SF的所有人执行一次查找(*<Lives-in><SF>)。 在第二步中，它将获取这些结果并将其发送到包含寿司的服务器，执行一次查找以获得所有吃寿司的人(*<eats><sushi>)，并与上一步的结果集交叉以生成来自sf的吃寿司的最终列表。 以类似的方式，然后可以进一步过滤/联接该结果集，每个联接在一个网络调用中执行。

![图3：数据分片](/Users/sunquan/Library/Application Support/typora-user-images/image-20200716140949346.png)

​		

### 	数据的再平衡

​			如上所述，每个分片在整体上都包含一个整体谓词，这意味着Dgraph分片的大小可能不均匀。 分片不仅包含原始数据，还包含其所有索引。 Dgraph 组包含许多分片，因此组的大小也可能不均匀。 组和分片大小会定期传达给zero。 ze ro使用启发式方法，使用此信息尝试在组之间实现平衡。 当前正在使用的信息只有数据的大小，其思想是大小相等的组将允许在为这些组提供服务的服务器之间使用公平的资源。 其他启发式方法，尤其是关于查询流量的启发式方法,可以在未来的工作中尝试.

​			为了达到平衡,Zero会将分片从一个组移动到另一个组。它通过将分片标记为只读,然后要求源组同时迭代底层的键值存储并将其流式传输到目标组的领导者来实现。 目标组的领导者通过Raft提交这些键值,从而获得随之而来的所有正确性。 一旦所有提案已成功由目标组应用，Zero将标记目标组正在服务的分片。 然后Zero将告诉源组从其存储中删除该分片,从而完成该过程。

​			虽然这个过程听起来相当直接,但是这里有许多竞争和边界条件可能会导致一致性问题,如 Jepsen 测试[5]所示。我们将在这里展示一些错误行为:

1. 当Alpha稍稍落后时,可能会发生错误,服务器会认为它仍在为分片提供服务(尽管分片已移动到另一组),并允许在其自身上运行突变。 为了避免这种情况,所有事务状态都会保留写入的分片和组信息(以及它们的冲突键，我们将在第5节中看到)。 然后，Zero检查分片组信息，以确保事务观察到的内容(通过它与之对话的Alpha)和Zero 的内容是相同的 如果不匹配将导致事务中止.

2. 另一个错误发生在将分片置于只读模式之后的事务提交-这将导致该提交在分片传输期间被忽略。 Zero通过将时间戳分配给移动操作来进行此操作。任何时间戳较高的提交(在此分片上)都将被中止,直到分片移动完成并且分片返回到读写模式

3. 当目标组接收到低于迁移时间戳的读取，或者源组在删除碎片之后接收到读取时，可能会发生另一次错误。 在这两种情况下,都不存在可能导致读取错误地返回空值的数据。 Dgraph通过通知目标组移动时间戳来避免这种情况，它可以使用该时间戳来拒绝对分片的任何读取。 类似地，zero包括成员资格标记，源Alpha在该组可以删除碎片之前在该成员标记处阻塞，因此,该组的每个Alpha成员在删除它之前将知道它不再服务于数据.

   

   总体而言,事实证明,在分片移动期间同步成员信息的机制在事务正确性方面是最难正确的。

## 索引

​		Dgraph被设计为应用程序的主要数据库,因此它支持大多数常用索引。 特别是对于字符串，它支持正则表达式，全文搜索，术语匹配，精确匹配和哈希匹配索引。 对于时间，它支持年，月，日和小时级别索引。对于geo，它支持附近，内部等操作，并很快...
​		所有这些索引都由Dgraph使用上述相同的postinglist格式来存储.索引和数据之间的区别是关键。 数据键通常是<predicate，uid>，而索引键是<predicate，token>。 使用索引token生成器从数据值派生token.

```go
type Tokenizer interface {
  Name() string
  // Type returns the string representation of the typeID that we care about.
	Type() string
  // Tokens return tokens for a given value. The tokens shouldn’t be encoded with the byte identifier.
  Tokens(interface{}) ([]string, error)
  // Identifier returns the prefix byte for this
  // token type. This should be unique. The range
  // 0x80 to 0xff (inclusive) is reserved for// user-provided custom tokenizers
  Identifier() byte
  // IsSortable returns true if the tokenizer can
  // be used for sorting/ordering. 
  IsSortable() bool
  // IsLossy() returns true if we don’t store the
  // values directly as index keys during
  // tokenization. If a predicate is tokenized
  // using a lossy tokenizer, we need to fetch
  // the actual value and compare.
  IsLossy() bool
  }
```

​		每个令牌生成器都有一个全局唯一的标识符（Identifier()字节)，包括由操作员提供的自定义令牌生成器。 生成的令牌以令牌化标识符为前缀,以便能够遍历仅属于该令牌化器的所有令牌。 在对不等式查询（大于，小于等）进行迭代时，这很有用。请注意，不等式查询仅在令牌化器可排序时才能进行（IsSortable()bool). 例如,在字符串中,精确索引是可排序的，但哈希索引不是.

​		根据谓词在模式中设置的索引，该谓词中的每个突变都将调用这些令牌生成器中的一个或多个来生成令牌。 请注意，索引仅对值起作用，而不对对象起作用。 将使用前突变值生成一组令牌，并使用后突变值生成另一组令牌。 将添加突变以从标记前的发布列表中删除uid，并将uid添加至after标记中。

​		注意所有索引都有对象值,因此它们在uid中主要处理。 尤其是索引可能会出现高扇出问题，可以使用2.2节中所述的发布列表拆分来解决。

## 多版本并发控制

​		如第2.2节所述，数据存储在Badger的发布列表中,该列表由按整数ID排序的发布组成。 所有发布列表写入都使用提交时间戳作为存储到Badger的版本号。 请注意，整个数据库中的时间戳是全局单调递增的，因此任何将来的承诺都保证具有更高的时间戳。	

​		由于多种原因无法本地更新这个列表,其一是Badger(和大多数LSM树)写入内存不可变的table,它与文件系统和rsync配合得非常好。第二，在排序列表中添加条目需要移动后面的条目,这取决于条目的位置，代价可能很高。 第三，随着PostingList的增长,我们希望避免每次发生突变时都重写较大的值(对于索引,这种情况可能会频繁发生)。

​		Dgraph 将PostingList视为一种状态。然后,每个延迟的更新操作都被存储为具有更高时间戳的 delta。一个数据删除通常包括一个操作(设置或删除)的Post。为了生成一个PosingtList,Badger 将按降序迭代这些版本,从读取时间戳开始,选择所有的 deltas，直到找到最新的状态。为了运行 postinglist 迭代，事务的正确记录将被选中，按整数 id 排序，然后在这些增量记录和底层 postinglist 状态之间运行合并排序操作(权权:可见Dgraph在多事务读取的时候会多做一些操作,但这避免了更新大post带来的抖动,相当于为每个postingList维护了一个预写日志)

​		此机制的早期版本为保持增量层按整数ID排序,将其覆盖在状态之上以避免在读取期间进行排序-所做的任何添加或删除都将基于增量层和状态中已有的内容进行合并。 事实证明，这些迭代过于复杂，以至于团队无法维护它们，而且很难发现错误。 最终,这个概念被抛弃了,取而代之的是一个简单易懂的解决方案,即选择正确的Post进行阅读并在获取之前对其进行排序。 此外，较早的API同时实现了向前和向后迭代，从而增加了复杂性。 随着时间的流逝，很明显只需要向前迭代，从而简化了设计。

​		避免在每次写入时重新生成PostingList状态有很多好处. 同时,随着增量的积累，列表重新生成的工作被委托给读者，从而降低了读取速度。 为了找到平衡，避免无限期地获取增量，我们增加了Rollups机制。

​		Rollups：读取key后,Dgraph将有选择地重新生成PostingList,这些PostingList的delta数量最少,或者有一段时间没有重新生成。 更新是通过从最新状态开始，然后按顺序遍历增量并将它们与状态合并来完成的。 然后，在最新的增量时间戳记上写回最终状态，重新放置增量并形成新状态。 然后可以丢弃该key的所有以前的deltas状态以回收空间。此系统允许Dgraph提供MVCC。 每个读取操作都在数据库的不变版本上进行。 较新的deltas会有更大的时间戳,并且在读取时会跳过较低的时间戳

## 事务

​		Graph的设计目标是操作简单。 因此,目标之一是不依赖任何第三方制度。 事实证明,当不仅为数据而且为事务提供高可用性时,这很难实现。

![图4:MVCC](/Users/sunquan/Library/Application Support/typora-user-images/image-20200721145948325.png)

​			在Dgraph中设计事务时，我们研究了Spanner [13]，HBase [11]，Percolator [17]和其他人的论文。 Spanner最著名的是使用原子钟为事务分配时间戳。 这是以不具有基于GPS时钟同步机制的商品服务器上较低的写入吞吐量为代价的。 因此，我们拒绝了这个想法，而是希望拥有一个zero服务器,该服务器可以更快地处理逻辑时间戳。为避免zero成为单个故障点，我们运行了多个zero实例，形成了一个Raft组。 但是，这带来了一个独特的挑战，即在领导人选举的情况下如何进行切换。 Omid，Reloaded [11]（称为Omid2）文件通过利用外部系统来解决此问题。 在InOmid2中，它们运行备用时间戳服务器以接管领导者发生故障的情况。 该备用服务器不需要获取最新的交易状态信息,因为Omid2使用了Zookeeper [2]，这是用于维护事务日志的集中式服务。 同样，TiDB构建了TiKV，它对键值使用基于Raft的复制模型。 这样，每个TiDB的writeDB都会自动被认为是高度可用的。类似地，Bigtable [12]使用Google Filesystem [15]进行分布式存储。 因此，不需要直接的信息传输就可以在构成仲裁的多个服务器之间进行。

​			尽管此概念在数据库中实现比较简单,但由于两个原因,我们对此并不完全满意。其一，我们有一个明确的目标，即不依赖任何第三方系统来简化Dgraph的运行,并认为存在一个解决方案 无需在Badger（存储）中推送同步复制就可以实现。 其次，除非有必要，否则我们希望避免接触磁盘。 通过使Raft成为Dgraph流程的一部分，我们可以发现何时将事务写入状态以达到更好的效率。 实际上,我们的事务实现直到提交后才写入磁盘上的数据库状态（仍写入Raft WAL).

​		   我们仔细研究了HBase的论文([14],[11])以寻找其他想法，但它们并不能直接满足我们的需求。 例如，HBase将大量事务信息推送回客户端，为客户端提供了关于应该或不应该读取哪些内容来维护事务保证的关键信息。然而，这使得客户端库更难构建和维护，这是我们不喜欢的。 最重要的是，一个图形查询可以在中间步骤中触及数百万个键，跟踪所有这些信息并将其传播到客户端的成本很高。

​		   Dgraph客户端库的目标是保持尽可能低的状态，以允许不熟悉Dgraph内部结构的开源用户以我们不熟悉的语言(例如，Elixir)构建和维护库。

​			如果不假设存储层是高可用的，我们就找不到一篇论文来描述如何构建一个简单易懂、高可用的事务系统。所以，我们必须想出一个新的解决方案。我们的第二个迭代仍然面临许多问题，这已经通过 Jepsen 测试得到了证实。因此，我们将第二次迭代简化为第三次迭代，如下所示

### 无锁的高可用性事务

​			Dgraph遵循无锁事务模型。每个事务并发地执行其进程，从不阻塞其他事务，同时读取位于或低于其开始时间戳的提交数据。 如前所述,Zero Leader维护Oracle，Oracle将逻辑事务时间戳分发给alpha。 Oracle还跟踪提交映射<存储冲突键→最近提交时间戳>。 如算法1所示,每个事务向Oracle提供冲突键列表,以及事务的开始时间戳。 冲突键派生自修改后的键，但并不相同。 对于每次写入，都会根据schema计算一个冲突键。 当事务请求提交时，Zero将检查这些键中是否有任何一个的提交时间戳高于事务的开始时间戳。如果满足条件,则中止事务。 否则,Oracle将租用一个新的时间戳,并在更新映射中的提交时间戳和冲突键时设置该时间戳。

```bash
Algorithm 1 Commit (Ts,Keys)
foreach key k∈ Keys do
		if lastCommit(k) > Ts then
					Propose(Ts←abort)
					return 
		end if
end for
Tc ← GetTimestamps(1)
for each key k ∈ Keys do
		lastCommit(k) ← T
end for
Propose(Ts ← Tc)
```

​			然后,zero领导者以Start→Commit(提交=0表示中止)的形式向跟随者建议此状态更新(提交或中止)，并达到法定人数。 一旦完成，Zero Leader就会将此更新发送给Alpha Leader的订阅者。为了保持设计的简单性,zero不会推动任何Alpha领导者,(无论是谁)最新的Alpha领导者的工作是从零开始建立输入流以接收交易状态更新。

​			在更新事务状态的同时，Zero Leader还发送了一个Max Assigned时间戳。 使用算法2计算Max Assigned，该算法维护所有分配的时间戳(开始时间戳和提交时间戳)的堆。 当达成共识时，时间戳被标记为完成，最大分配的时间戳被推进到最大时间戳，在此之前，一切都根据需要达成共识。 请注意，开始时间戳通常不需要达成共识(除非需要更新租约)并将其标记为立即完成。 提交时间戳始终需要达成共识，以确保Zero组达到关于事务状态的法定数量。 这允许zero follower成为领导者，并且完全了解交易状态。 正如我们将在下面看到的那样，这种排序对于实现交易保证至关重要。

``` bash
Algorithm 2 Watermark: Calculate DoneUntil (T,isPending)
if T/∈ MinHeap then
		MinHeap <- T
end if
pending(T) <- isPending
curDoneTs <- DoneUntil
for each minTs ∈ MinHeap.Peek() do
		if pending(minTs) then
			 break 
		 end if
		 MinHeap.Pop()
		 curDoneTs <- minTs
end for 
DoneUntil <- curDoneTs
```

​			一旦Alpha领导者收到此更新,他们将以相同的顺序向他们的追随者提出，并应用更新。Alpha中的所有raft提案申请都是按顺序完成的。Alpha也有一个Oracle，用于跟踪待处理的事务。 它们维护开始时间戳，以及将所有更新的发布列表保存在内存中.

![图5:Max Assigned 用圆圈表示，实心圆圈表示完成。 开始时间戳1、2和4立即标记为完成。 提交时间戳3开始，并且在完成之前必须达成共识。 最高时间戳的标志保持串行,在该时间戳和低于该时间戳的所有工作都是如此。](/Users/sunquan/Library/Application Support/typora-user-images/image-20200722133837102.png)



![图6：MaxAssigned系统确保可线性化读取,高于当前MaxAS-Signed(MA)时间戳的读取必须阻止,以确保在应用readTimestamp之前向上写入。 TXN2接收开始TS3，并且读ATTS3必须确认直到TS2的任何写入](/Users/sunquan/Library/Application Support/typora-user-images/image-20200722133958107.png)

​				在事务中止时,只需丢弃缓存。在事务提交时,使用提交时间戳将投递列表写入Badger。 最后,更新MaxAssignedTimestamp。

​				每个读取或写入操作都必须具有开始时间戳。 当新的查询或变异击中Alpha时，它将询问zero以分配时间戳。 通常对这一操作进行分批处理，以便每个Alpha仅允许一个对Zero Leader的未决分配呼叫。 如果新接收到的查询的开始时间戳高于该Alpha注册的MaxAssigned，则它将阻止查询，直到其MaxAssigned到达或超过开始ts为止。 该解决方案很好地解决了各种极端情况，包括Alpha从对等方退回或进入网络分区后,或者在崩溃后重新启动等。在所有这些情况下，查询都将被阻止，直到Alpha看到 所有更新直到查询的时间戳,从而保持了对事务和可线性化读取。

​				为了正确起见,只允许zero领导者分配时时间戳,UID等。在特殊情况下,zero追随者会错误地认为他们是领导者并提供状,Dgraph会做多件事来避免这些情况:

​				1.如果zero领导更迭,新领导将出租比上一任领导更高的一系列时间戳。 然而,老领导坚持的旧提交建议可以被转发给新的提交建议。 这可能允许提交发生在较旧的时间戳，从而导致事务保证失败。 我们通过禁止zero跟随者向领导者转发请求并拒绝这些建议来避免这一点.

​				2.从Zero流传输的每个成员状态更新都需要读取仲裁(与Zero对等点核对以找到该组看到的最新的Raft index更新)。 例如,如果Zero是一个分区，它将无法达到此法定人数并发送成员更新。alpha期望定期更新，如果在几个周期后没有收到zero leader的消息,他们会认为zero leader已经不存在，取消连接，并重新尝试与一个(可能不同的)健康领袖建立连接。

## 一致性模型

​				Dgraph支持MVCC,读取快照和分布式ACID事务。 事务是跨通用数据集的群集范围的，不受任何关键级别或服务器级别的限制。 交易也是无锁的。 它们不会阻止/等待通过未提交的事务操作看到挂起的写入。 它们都可以并发进行，zero会根据冲突选择提交或中止它们.

​			考虑到跟踪单个图查询读取的所有数据(可能是数百万个键)的开销,Dgraph不提供可序列化快照隔离。 相反，Dgraph提供快照隔离跟踪写入,这是一个比读取包含更多内容的集合。				

​			Dgraph会为事务（由Tx表示）单调增加时间戳（由T表示）。因此，如果有任何事务T(xi)在T(xj) start之前提交，则T(xi).commit <T(xj).start。 如果T.read > T.commit，则保证任何客户端在timestamp T.read的读取都可以看到T.commitis上的任何提交。 因此，Dgraph读取是线性的。 此外,所有读取都是整个群集中的快照，可以完整查看所有先前提交的事务.

​			如前所述，Dgraph 读取是可线性化的。虽然这对正确性很有帮助，但是当大量的读写操作同时进行时，可能会导致性能问题。所有的读都应该被阻塞，直到 Alpha 看到所有的写，直到读的时间戳。在许多情况下，开发人员希望性能超过实现线性。	

​			Dgraph提供两个加快读取速度的选项:

​			1.典型的读写事务会给客户端分配一个新的时间戳。 这将更新“最大任务”，然后通过“zero领导”流向alpha领导，然后提出建议。 只读事务仍然需要从zero开始的读取时间戳，但是zero会随机性地将相同的读取时间戳分发给多个调用者,这样 Alpha 就可以分摊到多个查询之间的最大分配成本。

​			2.尽力而为事务是只读事务操作的变体,它将使用Alpha的观察到的Max Assigned Timestamp作为读取时间戳。 因此,接收方Alpha根本不必阻塞,并且可以继续处理查询。 这相当于其他数据库中典型的最终一致性模型。 最终,每个Dgraph读取都是整个分布式数据库的快照,并且所有读取都不会违反快照保证.

## 复制

​		Dgraph的大多数更新都是通过RAFT完成的。 让我们从Alpha开始，它可以通过系统推送大量数据。 所有突变和事务更新都是通过RAFT提出的，并成为RAFT预写日志的一部分。 在崩溃和重启时，RAFT日志从最后一个快照重放，以使状态机恢复到正确的最新状态。 另一方面，日志越长，Alpha在重新启动时重放它们所需的时间就越长,从而导致启动延迟。 因此,必须通过快照来精简日志,该快照指示该点之前的状态已持续存在,不需要在重新启动时重放.

​		如上所述，Alpha将突变写入RaftWAL，但将其保留在事务缓存中的内存中。 提交事务后，会将突变在提交时间戳记中写入状态。 这意味着在重新启动时，必须通过Raft WAL将所有未决的事务带回内存。 这需要进行计算以选择正确的Raft索引来修剪日志，这将所有待处理的交易完整保留在日志中。

​		我们在解决Jepsen问题时吸取的教训之一是，为了提高复杂分布式系统的可调试性，系统应该像时钟一样工作。 换句话说，一旦一个系统中的事件发生了，其他系统中的事件几乎应该是可预测的。 这个指导原则决定了我们如何生成快照。

​		raft可以使领导者和追随者彼此独立地生成快照. Dgraph曾经这样做,但那给系统带来了不可预测性并进行了困难的调试.因此,根据可预测性原理的经验教训,我们对其进行了更改，以使领导者计算快照索引并提出此结果。 这样一来，领导者和跟随者就可以在相同的索引上（确切地说,通常是在同一时间）拍摄快照。 此外,这种组级别快照事件随后被传达为ze ro，以允许它通过删除快照时间戳下面的所有条目来修剪冲突映射。 在日志中遵循此事件链可以极大地提高系统的可调试性。

​		Dgraph仅将元数据保留在Raft快照中，实际数据单独存储。 Dgraph在快照期间不会复制该数据。 当关注者落后并需要快照时，它会要求领导者提供快照，并且领导者将快照从其状态流式传输（像Dgraph一样，Badger支持MVCC，并且在特定时间点进行读取时,将在逻辑快照上运行数据库）。 在以前的版本中，跟随者在接受领导者的更新之前会先清除其当前状态。 在新版本中，领导者可以选择仅将增量状态更新发送给跟随者，这可以大大减少传输的数据.

## 高可用性与扩展性

​		Dgraph的架构围绕用于更新日志序列化和复制的RAFT组。 在CAP吞吐量中，这遵循CP，即在网络分区中，Dgraph将选择一致性而不是可用性。 然而,CAP定理的概念不能与高可用性混为一谈,高可用性取决于在不影响服务的情况下丢失多少实例,在三节点raft组中,Dgraph每组可以丢失一个实例，而不会对数据库的性能造成任何可测量的影响。 然而,考虑到所有更新都要通过RAFT，丢失同一组中的两个实例会导致Dgraph阻塞。 在由五个节点组成的组中，在不影响功能的情况下可以丢失的实例数量是两个。 我们不建议运行超过5个的raft组.考虑到Dgraph Zero的中心管理角色，可以假设Zero将会成为单点故障。但是，事实并非如此。 在zero跟随者死亡的情况下,什么都不会改变。 如果Zero领导者崩溃，其中一名Zero追随者将成为领导者，续订其时间戳和uid分配租约,拿起交易状态日志(通过RAFT存储)，并开始接受Alpas的请求。 在此转换过程中唯一可能丢失的是试图使用最后zero提交的事务.它们可能会出错,但可以重试。 alpha也是一样。 所有Alpha追随者都有与Alpha领导者相同的信息,组中的任何成员都可以在不丢失任何状态的情况下失败.

​	 	Dgraph可以支持由32位整数表示的尽可能多的组(即使这是人为的限制)。 每个组可以有一个、三个、五个(可能更多，但不建议)副本。 系统中可以存在的UID(图形节点)的数量受64位无符号整数的限制，事务时间戳也是如此。 所有这些都是非常宽松的限制,并不能成为影响可伸缩性的原因.

## 查询

​		 一个典型的 Dgraph 查询可以击中许多 Alphas,这取决于谓词的位置。每个查询被细分为多个任务,每个任务负责一个谓词.

### 	遍历

​		Dgraph查询任务(以下称为任务)一般是围绕在遍历期间转换UID列表矩阵的机制构建的。 查询可以有一个要遍历的UID列表，执行引擎将同时在Badger中进行查找，以获得每个UID的发布列表(请注意,谓词始终是任务的一部分)，将每个UID转换为列表。 因此,任务查询将返回UID矩阵。如果谓词包含一个值(例如，Predicate name)，则Uid List返回一个值列表，也称为Value Matrix。一个谓词可以只允许一个uid/值，也可以允许多个uid/值。 这种机制在这两种情况下都能正常工作。 如果发布列表只有一个uid/值，则结果列表将只有一个元素。 在这种情况下,只有一个1*1的矩阵,每个列表包含零个或一个元素。 请注意，Uidin列表的索引与UidMatrix中的列表索引之间存在奇偶校验。 因此，Dgraph可以准确地维护这些关系.

​		Value Matrix通常是任务树中的叶子。 一旦有了值,我们只需要在结果中对它们进行编码即可,但是具有UidMatrix结果的任务通常会有子任务。 这些子任务将需要查询UidList进行处理。 Dgraph将把UidMatrix合并排序到单个的Uid排序列表中，然后将其复制到子任务中。 每个子任务可以类似地在相同或其他谓词上运行扩展。

###    函数

​		Graph还支持函数,当全局uid空间需要限制在较小的集合(甚至是单个uid)时，这些函数提供了一种简单的方式来查询Dgraph。 函数还提供高级功能，如正则表达式、全文搜索、可排序数据类型上的等式和不等式、地理空间搜索等。这些函数也编码到任务查询中，只不过这次它们不是以UidList开头。 相反，任务查询包含从与这些函数正在使用的索引对应的记号器派生的记号(如上所述)。 大多数函数需要某种类型的索引才能操作，例如,正则表达式查询使用三元组索引，地理空间查询使用基于S2单元的地理索引，依此类推...如上一节所述，索引中的键编码谓词和令牌，而不是谓词和uid。 因此，填充矩阵的机制与任何其他任务查询中的机制相同。我们使用token列表而不是Uid列表作为查询集。		

### 	过滤器

​		上述技术适用于遍历, 但是，过滤器(交叉点)是用户查询的一大部分。 每个任务包含一个UidList作为查询，一个矩阵作为结果。 Task还存储一个结果uid列表，该列表可以存储来自结果UidMatrix的uid集。 根据是否应用了筛选器，此uid集可以与merge-sorted UidMatrix 相同，也可以是其子集。筛选器本身就是一棵树。 Dgraph支持AND、OR和NOT过滤器，可以进一步组合以创建复杂的过滤器树。 过滤器通常由可以请求更多信息的函数组成，并表示为任务。 这些任务在上面描述的相同机制下执行，但是要做另外一件事。 任务还包含UID的源列表(筛选器要应用到的父任务的结果集)。 此UID列表作为筛选任务的一部分发送。 任务使用这些UID在目标服务器上执行任何交集，仅返回结果的子集，而不是检索任务的所有结果。 这可以显著减少结果有效负载大小，同时还允许在过滤任务执行期间进行优化以加快速度。 返回结果后，协调服务器将使用AND、OR或NOT运算符对结果进行拼接



### 	交集

​		 UID交集本身使用三种算法,根据结果大小与源UidList的大小之比在线性扫描, block jump或二分搜索之间进行选择，以提供最佳性能。当两个列表的大小相同时，Dgraph在两个列表上使用线性扫描。 当一个列表比另一个列表长得多时，Dgraph将迭代较短的列表,并在较长的列表上执行二进制查找。 对于介于两者之间的某个范围，dgraph将在较短的列表上迭代，并在较长的列表上执行正向查找block jump。 Dgraph基于块的整数编码机制使得所有这些都非常高效。



##   未来的工作

​		由于严重的读写争用，我们已经从Dgraph中删除了数据缓存，并构建了一个新的,无争用的GO缓存库来帮助我们进行读取。 将其与Dgraph集成的工作正在进行中。 Dgraph没有任何查询或响应高速缓存,在MVCC环境中很难维护这样的高速缓存，在MVCC环境中，每次读取根据其时间戳可能有不同的结果。排序整数编码和交集是一个热门的搜索主题，在性能方面有很大的优化空间。 正如前面提到的，切换到 Roaring Bitmap的实验工作正在进行中。我们还计划开发一个查询优化器，它可以更好地确定执行查询的正确顺序。到目前为止，GraphQL的简单性质让操作员可以手动优化他们的查询-但Dgraph肯定可以更好地了解数据的状态。未来的工作是允许在碎片移动期间写入，这取决于碎片的大小可能需要一些时间	

##    贡献者

​	   如果没有核心开发团队和扩展社区的不懈贡献，Dgraph是不可能实现的。 如果没有我们投资者的资金，这项工作也是不可能的。 此处提供了完整的投稿人名单`https://github.com/dgraph-io/dgraph/graphs/contributors`

Dgraph是一个开源软件，可从`https://github.com/dgraph-io/dgraph`

有关Dgraph的更多信息，请访问:`https://dgraph.io`

## References

[1]Achievingrapid   responsetimesinlargeonlineserviceshttps://static.googleusercontent.com/media/research.google.com/en//pubs/archive/44875.pdf.

[2]  Apache zookeeper.https://zookeeper.apache.org.

[3]  Badger: Fast key-value db in go.

[4]Buildinganativegraphqldatabase:Challenges,   learn-ingsandfuturehttps://blog.dgraph.io/post/building-native-graphql-database-dgraph/.

[5]Dgraph’sjepsenanalysishttps://jepsen.io/analyses/dgraph-1-0-2.

[6]Graphql+-:   Dgraph  query  languagehttps://docs.dgraph.io/query-language.

[7]Graphqlspec:https://graphql.github.io/graphql-spec/June2018/.

[8]grpc: A high performance, open-source universal rpc frameworkhttps://grpc.io/.

[9]Protocol buffers: A language-neutral, platform-neutral extensible mech-anism for serializing structured data.https://developers.google.com/protocol-buffers.

[10]Roaring bitmaps: A better compressed bitset https://roaringbitmap.org/.

[11]BORTNIKOV, E., HILLEL, E., KEIDAR, I., KELLY, I., MOREL, M.,PARANJPYE, S., PEREZ-SORROSAL, F.,ANDSHACHAM, O.Omid,reloaded:  Scalable and highly-available transaction processing.   In15th USENIX Conference on File and Storage Technologies (FAST 17)(Santa Clara, CA, 2017), pp. 167–180.

[12]CHANG, F., DEAN, J., GHEMAWAT, S., HSIEH, W. C., WALLACH,D.  A., BURROWS, M., CHANDRA, T., FIKES, A.,ANDGRUBER,R. E.Bigtable: A distributed storage system for structured data.ACMTrans. Comput. Syst. 26, 2 (June 2008), 4:1–4:26.

[13]CORBETT,  J.   C.,  DEAN,  J.,  EPSTEIN,  M.,  FIKES,  A.,  FROST,C., FURMAN, J.  J., GHEMAWAT, S., GUBAREV, A., HEISER, C.,HOCHSCHILD, P., HSIEH, W., KANTHAK, S., KOGAN, E., LI, H.,LLOYD,  A.,  MELNIK,  S.,  MWAURA,  D.,  NAGLE,  D.,  QUINLAN,S.,  RAO,  R.,  ROLIG,  L.,  SAITO,  Y.,  SZYMANIAK,  M.,  TAYLOR,C., WANG, R.,ANDWOODFORD, D.Spanner: Google’s globally-distributed database. InProceedings of the 10th USENIX Conferenceon Operating Systems Design and Implementation(2012), OSDI’12,pp. 251–264.

[14]FERRO, D. G., JUNQUEIRA, F., KELLY, I., REED, B.,ANDYABAN-DEH, M.Omid: Lock-free transactional support for distributed datastores.  InData Engineering (ICDE), 2014 IEEE 30th InternationalConference on(2014), pp. 676–687.

[15]GHEMAWAT, S., GOBIOFF, H.,ANDLEUNG, S.-T.The google file sys-tem. InProceedings of the Nineteenth ACM Symposium on OperatingSystems Principles(2003), SOSP ’03, pp. 29–43.

[16]ONGARO, D.,ANDOUSTERHOUT, J.In search of an understandableconsensus algorithm. In2014 USENIX Annual Technical Conference(USENIX ATC 14)(Philadelphia, PA, 2014), pp. 305–319.[17]PENG, D.,ANDDABEK, F.Large-scale incremental processing usingdistributed transactions and notifications.  InProceedings of the 9thUSENIX Symposium on Operating Systems Design and Implementation(2010).11
