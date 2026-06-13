
#### 2. 说一下MapReduce的运行机制
MapReduce包括输入分片/map阶段/combine阶段/shuffle阶段和reduce阶段.分布式计算框架包括client,jobtracker和tasktracker和调度器.
* 输入分片阶段,mapreduce会根据输入文件计算分片,每个分片对应一个map任务
* map阶段会根据mapper方法的业务逻辑进行计算,映射成键值对
* combine阶段是在节点本机进行一个reduce,减少传输结果对带宽的占用
* shuffle阶段是对map阶段的结果进行分区,排序,溢出然后写入磁盘.将map端输出的无规则的数据整理成为有一定规则的数据,方便reduce端进行处理,有点像洗牌的逆过程.  https://blog.csdn.net/ASN_forever/article/details/81233547
* reduce阶段是根据reducer方法的业务逻辑进行计算,最终结果会存在hdfs上.
