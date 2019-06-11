### Zookeeper作用

通过zookeeper cluster可以获取当前系统表.META.的存储所对应的regionserver信息

保证任何时候，集群中只有一个master（负责多HMaster的选举）

存贮所有Region的寻址入口

实时监控RegionServer的状态、将RegionServer的上线和下线信息实时通知给Master（服务器之间状态同步）

存储Hbase的schema（元数据信息）。包括有哪些table、每个table有哪些column family等

### HMaster的作用
(主要负责table和region的管理工作)

Region分裂后，为RegionServer分配新的Region

负责RegionServer的负载均衡，调整region的分配

发现失效的region server并重新分配其上的region 

管理用户对table的增、删、改、查操作

监听zk,基于zookeeper感应rs的上下线

监听zk,基于zookeeper来保证HA

处理schema更新请求 （管理用户对表的增删修改）

不参与对表的读写访问

负载很低

无SPOF(单点故障)

在一个RegionServer死机后，负责失效节点的Region的迁移

### HRegionServer作用

(主要负责响应用户对其上region的I/O请求，向HDFS读写数据，HbasHBase核心模块)

HRegionSserver维护Master分配给它的region

维护region的cache

处理region的flush、compact、split

内部管理一系列的HRegion对象

一个HRegionServer会有多个HRegion和一个HLog

### Region 

每一个HRegion又由很多的Store组成，每一个Store存储的实际上是一个列簇（ColumnFamily）下所有的数据。此外，在每一个Store（又名HStore）中有包含一块MemStore。MemStore驻留在内存中，数据到来时首先更新到MemStore中，当到达阈值之后再flush（默认64M）到对应的StoreFile（又名HFile）中，所以每一个Store包含多个StoreFile，StoreFile负责的是实际数据存储，为HBase中最小的存储单元。

达到某个阈值时，分裂（默认256M）。所以一个HRegionServer管理多个表，一个表下有多个Region，一个HRegion有多少个列族就有多少个Store,Store下有多个StoreFile文件，是HBase中最小的存储单元

每个Column Family单独存储：storeFile；(storefile的数量一多（到达阀值），就合并（同时合并版本以及删除之前要删除的数据）；合并后大小到达阀值就split）

当某个Column Family累积的大小（具体的数据量） > 某阈值时，自动分裂成两个Region；合并之后，旧数据也不是立即删除，而是复制一份并同内存中的数据一起写到磁盘，在之后，LSM-Tree会提供一些机制来回收这些空间

如何找到某行属于哪个region呢？META表

### HFile
（HFile基于Hadoop的TFile类实现，模仿Google的BIgTable论文中的SSTable格式。）

HBase中KeyValue数据的存储格式，是hadoop的二进制格式文件。

HFile文件是不定长的，长度固定的只有其中的两块：Trailer和FileInfo。

Trailer中有指针指向其他数据块的起始点，FileInfo记录了文件的一些meta信息。 - Data Block是hbase io的基本单元，为了提高效率，HRegionServer中有基于LRU的block cache机制。

每个Data块的大小可以在创建一个Table的时候通过参数指定（默认块大小64KB），大号的Block有利于顺序Scan，小号的Block利于随机查询。

每个Data块除了开头的Magic以外就是一个个KeyValue对拼接而成

Magic内容就是一些随机数字，目的是防止数据损坏 
