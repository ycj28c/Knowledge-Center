## Designing Pastebin 笔记

试用了一下pastebin.com，我们可以输入一段text，pastebin可以将这段text转换为一个随机url让其他人浏览，或者提供可以嵌入的html或者iframe。感觉和tinyurl是差不多的，不过这个url是提供text而不是redirect服务。

### Pastebin是什么
Pastebin类型的服务允许用户存储plain text或者images，并生成unique URLs来访问这些数据。

### 确认需求
Functional Requirements：  
1.用户可以上传或者paste数据并或者unique URL和访问。  
2.用户只能上传text。  
3.这些数据和links在一定时间后过期；用户可以指定超时。  
4.用户可以选择使用一个自定义alias。  

Non-Functional Requirements:  
1.系统是高可靠reliable的，任何上传数据不会消失。  
2.系统是高可用available的。个别服务器当了，用户还是能访问。  
3.用户可以实时访问他们的pastes。  
4.Paste的link不应该很容易被猜到。  

Pastebin是基于URL Shortening service，不过增加了一些额外功能。用户上传的文件上限设置为10MB，个别url可以允许更大的文件大小上限。


### 容量估算和约束
这是一个read-heavy的服务；大概是5：1的读写比例。

Traffic esimates：  
假设大概5million读请求每天。  
平均每秒新pastes：1M / (24 hours * 3600 seconds) ~= 12 pastes/sec  
平均读取每秒：5M / (24 hours * 3600 seconds) ~= 58 reads/sec  

Storage estiamtes：  
文件上限是10MB，假设平均每个paste是10KB。这样，我们需要    
1M * 10KB => 10 GB/day   
如果我们需要存放10年的数据，那么需要36TB空间，一共有3.6billion的pastes。每个文件一个url。假设我们使用base64编码([A-Z, a-z, 0-9, ., -])，如果是6位数，那么可以生成：   
64^6 ~= 68.7 billion unique strings  
如果使用one byte来存放个url，那么3.6billion就需要：  
3.6B * 6 => 22 GB  
来存放所有生成的URLs，22GB比起36TB可以忽略不计。不过我们假设这个是70%的capacity modal，那么会需要额外30%的空间，也就是51.4TB总空间。  

Bandwidth estimates：  
在写请求，我们预计12pastes/sec，所以大概是120KB/s。读的话是写的5倍，也就是0.6MB/s的Bandwidth，并不是非常大。  

Memory estimates：  
我们可以cache热门的paste用于快速读取，根据80-20原则，我们cache 20%的每日流量。因为平均每天有5M的读取，20%就是：  
0.2 * 5M * 10KB ~= 10GB cache 

### 系统APIs
增加Paste的API如下：
```
addPaste(api_dev_key, paste_data, custom_url=None user_name=None, paste_name=None, expire_date=None)
```
api_dev_key(string)：注册账号的API key，可以用来rate limiter之类。  
paste_date(string)：paste的文本数据。  
custom_url(string)：可选，是自定义URL。  
user_name(string)：可选，用来生成URL的用户名。  
paste_name(string)：可选，是paste的名称。  
expire_date(string)：可选，设置paste的过期时间。  

返回：(string)  
成功返回url，否则返回错误代码。  

访问Paste的API：  
```
getPaste(api_dev_key, api_paste_key)
```
这里的api_paste_key就是该paste的主键。

删除Pasted的API：  
```
deletePaste(api_dev_key, api_paste_key)
```
成功返回true，失败返回false

### 数据库设计
几个关键点：有billion的数据量，每个记录metadata很小，但是每个文件是MB级别，数据和数据之间没有强联系，并且是read-heavy系统。

Database Schema：  
Paste表  
URLHash：varchar（16）PK    
ContentKey：varchar（512）  
ExpirationDate：datetime  
UserID：int  
CreationDate：datetime  

User表  
UserID：int PK  
Name：varchar（20）  
Email：varchar（32）  
CreationDate：datetime  
LastLogin：datetime  

比较重要的就是URLHash就是URL键值和TinyURL一样，而ContentKey就是一个指向外部object（存储paste内部）的key了

### 高级设计  
就是画图，结构非常简单。

Client -- Application server -- Object storage
                |
				|
			Metadata storage

这里的Object storage可以是Amazon S3，方便易用。

### 详细设计
a.应用层设计（和tinyURL几乎一样）  

How to handle a write-request？  
每当受到一个写请求，服务器就生成一个6位随机string作为paste的key。然后将内容和key存放到数据库。当插入成功，将key返回给用户，如果失败（比如用户自定义的key重复）就返回错误代码。  
可以使用类似TinyURL的Key Generation Service（KGS）来预生成随机6位数字，并存放在一个key-DB。那么和tinyURL一样的方式，用2个表，一个表存放没用过的key，一个表存放用过的keys。同样也可以将一些key放在内存中这样服务器可以快速读取。如果KGS宕机了，那么内存中的keys就永远丢失了，不过因为数据不大可以忽略。

Isn't KGS a single point of failure？  
是的，所以需要一个replica，主备模式。生成key会很快，所以单机就够了。  

Can each app server cache some keys from key-DB？  
可以，应该会更快。不过如果该服务器宕机这些key也就丢失了。

How does it handle a paste read request？  
通过key从数据库直接读就行了，没读到就返回错误。

b.数据库层  
可以分为两个数据库：  
1.Metadata数据库：  
可以使用关系型数据库比如MySQL或者分布式key-value数据库比如Dynamo(key-value的代表数据库)或者Cassandra(列式数据库代表)。  
使用关系型数据库可以保证事务的一致性，用户完成请求总能得到正确的返回。缺点是强一致性，速度慢。而pastebin的服务主要是public生成，所以无法对用户进行很好的shard，造成扩展也是很困难。  
使用非关系型数据库可以无脑扩展，保证性能和吞吐量，有大量用户也能很快响应。缺点是在集群情况下是基本使用最终一致性，主用户可以直接write后获得最新数据，但是其他地域的用户马上使用这个url可能不能马上获取到。另外这个情况因为还需要存储text，肯定需要一部分的返回时间，所以noSQL应该不明显吧。
2.Object存储：  
可以将内容存放到Amazon S3。可以方便扩容。

画图：
Clients -- LB -- App Server -- LB -- Object storage
					|			\ 		 |	
					|			 \ -- Block Cache
					|			\-- Metadata Cache
				   KGS					 |
					|			\--  Metadata storage
					|					 |
Key-DB(stand-by)-- Key-DB --- Cleanup Service

其中的block Cache没有细说

### 其他功能

数据清理：  
Purging or DB cleanup见TinyURL章节，大致就是在expire后删除并回收key，放回到Key-DB去。默认两年expiration时间。

数据分区和备份：  
data partitioning和Replication见TinyURL章节，大致是说可以用URL的first letter进行分区，但是可能导致不平衡。也可以使用URL的Hash结果分区，服务器超载仍然可能发生，可以使用consistent hashing来增加系统的可用性。使用hash来分区的场景都可以用上consistent hashing来平衡。		

缓存和负载均衡：  
Cache and Load Balancer见TinyURL章节，大致就是使用memcached，用LRU，20%的cache之类。量级大的Application server和数据库前加load balance。	

安全和权限：  
Security and Permissions见TinyURL章节，大致就是可以给vip用户提供私有URL，需要登录后才能访问，比如session或者oauth之类。


### 总结
基本上tinyURL一样，可以直接看tinyURL。  
什么时候用dynamo和cassandra需要再研究下。
					

