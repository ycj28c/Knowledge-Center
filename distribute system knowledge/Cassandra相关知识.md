列式分布式数据库，扩展很容易，它的数据模式（列的增减）的改动的成本是非常低的。

一个应用就是比如user，还有很多follow的情况，那么user-follower关系表可以用cassandra，因为给一个用户增减follower消耗低不锁表。

缺点一样，NoSQL导致Transaction问题，然后分布式始终还有各个node数据统一的消耗，最终一致还是慢。

## Cassandra存储结构
Cassandra是一个三层结构(三元组结构)的NoSQL数据库:
插入数据：insert(row_key,column_key,value)

##### 第一层：row_key
又称为hash_key，cassandra会根据这个key计算一个hash值，然后决定这条数据存在哪  
是传统我们所说的key-value中的key  
任何查询都需要带上这个key，但无法进行range query  
最常用的row_key:uer_id  
##### 第二层：column_key
是排序的，可以进行range query  
可以按column指定顺序排序，支持按column范围查询query(row_key,column_start,column_end)  
可以是复合值，比如是一个timestamp+user_id的组合  
#### 第三层：value
一般是string  
如果需要存很多信息的话，可以自己做序列化  

Cassandra存储friendship:重要的信息，需要频繁查的信息不能放在value中，要放在column_key中  
Cassandra存储NewsFeed:将create_time存在column_key中可以按时间排序  

## 和Hbase的区别
两者都是wide column数据库，但是有重大区别。

虽然两者都是基于LSM Tree，也就是Google三家马车GFS的存储结构（到Yahoo后称之为HGFS了），但是Hbase是中央架构（master-slave）所以可以存放很大的文件，不过对应的存放的数据都是String，而且配置极其麻烦。而Cassandra使用的是去中心架构，每个node内部采用LSM Tree，不过却是consistent hashing的分布，扩展更容易，适合存放大量小文件。

当你仅仅是存储海量增长的消息数据，存储海量增长的图片，小视频的时候，你要求数据不能丢失，你要求人工维护尽可能少，你要求能迅速通过添加机器扩充存储，那么毫无疑问，Cassandra现在是占据上风的。但是如果你希望构建一个超大规模的搜索引擎，产生超大规模的倒排索引文件（当然是逻辑上的文件，真实文件实际上被切分存储于不同的节点上），那么目前HDFS+HBase是你的首选。

## LSM Tree知识点巩固
LSM Tree是一个架构，不是一种数据结构。基于GFS。  

特性有：  
1.顺写日志，速度很快  
2.先写入内存memtable，这里采用的是skip table来做，保证有序  
3.写满了memtable就写入到文件系统SSTable去，有顺序  
4.插入时候不删除key，每次只是找最后一个key  
5.每个memtable有bloomfilter，可以快速过滤掉不存在该chunk的key  
6.遍历搜索时候先搜索memtable，不存在则倒序搜索SSTable，每次都是先看bloomfilter，如果存在就尝试二分搜索  
7.定期进行compaction来清理数据，精简空间  

## Reference
[Dynamo扩展篇 —— Cassandra VS Dynamo](http://systemdesigns.blogspot.com/2016/01/cassandra-vs-dynamo.html)  
[从用户系统理解数据库和缓存](https://marian5211.github.io/2018/01/30/%E3%80%90%E4%B9%9D%E7%AB%A0%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1%E3%80%91%E4%BB%8E%E7%94%A8%E6%88%B7%E7%B3%BB%E7%BB%9F%E7%90%86%E8%A7%A3%E6%95%B0%E6%8D%AE%E5%BA%93%E5%92%8C%E7%BC%93%E5%AD%98/)  
[HBase、MongoDB、cassandra比较](https://www.cnblogs.com/yanduanduan/p/10563678.html)  

