## Designing a URL Shortening service like TinyURL 笔记

### 1.Why do we need URL shortening?
就是让url缩短，容易记忆的大小，甚至自定义。同时也起到隐藏初始url的作用。

### 2.Requirements and Goals of the System
要确保设计的就是面试人想要的，问清楚。

Functional Requirements：  
1.给一个URL，能生成一个短的独一无二的alias  
2.当用户访问link，服务能redirect到其初始link  
3.用户可以选择一个custom的link  
4.这些links在一个固定时间后会失效，用户必须设定过期时间

Non-Functional Requirement：  
1.系统高available，因为一旦宕机，依赖的URL redirections也失效了  
2.URL redirection应该延迟很小  
3.Shortened links不应该被猜到

Extended Requirement：  
1.分析；比如redirection发生次数的分析  
2.最好还能为其他服务提供REST APIs 

### 3.Capacity Estimation and Constraints
系统将是read-heavy的。假设这个比例是100：1读和写。  

Traffic estimates：假设有500M的新URL shortenings每月，根据100：1的读写比例，将会有50B的redirections每月。

What would be Query Per Second（QPS）for our system？
```
# 写
500 million / (30 days * 24 hours * 3600 seconds) = ~200 URLs/s
# 读
100 * 200 URLs/s = 20K/s
```

Storage estimates: 假设存放5年的URL shortening请求，那么
```
500 million * 5 years * 12 months = 30 billion
```
假设每项数据需要500bytes，那么需要total storage：
```
30 billion * 500 bytes = 15 TB
```

Bandwith estimate：因为我们期望200个新URL每秒，所以带宽是
```
# 写
200 * 500 bytes = 100 KB/s
# 读
20K * 500 bytes = ~10 MB/s
```

Memory estimates：假如我们需要cache部分热门URLs，那么需要多少内存？我们以天为单位进行cache，根据80-20法则
```
# 读总共每天
20K * 3600 seconds * 24 hours = ~1.7 billion
# 需要的cache
0.2 * 1.7 billion * 500 bytes = ~170GB
```
实际内存应该小于170GB，因为很多请求是重复的。

### 4.System APIs
可以使用SOAP或者REST APIs，例子
```
createURL(api_dev_key, original_url, custom_alias=None, user_name=None, expire_date=None)
```
api_dev_key(string)：就是注册账户所用的API developer key，可以用来进行throttle。 
original_url(string)：原始URL  
custom_alias(string)：可选项，用户自定义URL  
user_name(string)：可选项，用户名可以用来encoding  
expire_date(string)：可选项，每个shorten URL的过期时间

return(string)：成功时候返回shortened URL，否则返回error code

```
deleteURL(api_dev_key, url_key)
```
成功删除的返回“URL Removed”

How do we detect and prevent abuse？恶意用户可以消耗掉所有的URL keys，所以需要通过api_dev_keyl来限制每个用户的URL创建，还有redirections的次数（在一定周期内，可以看rate limiter那一部分内容）

### 5.Database Design
已知有以下需求：  
1.我们需要存放billions级别数据  
2.每个对象都很小  
3.每个记录之间没有联系  
4.read-heavy的系统  

假设这是一个需要登录用户的系统，那么两个表URL表和User表

URL大概有下列字段：  
PK: Hash: varchar(16)  
OriginalURL: varchar(512)  
CreationDate: datetime  
ExpirationDate: datatime  
UserID: int  

User大概有下列字段：  
PK: UserID: int  
Name: varchar(20)  
Email: varchar(32)  
CreationDate: datetime  
LastLogin: datetime  

What kind of database should we use？很显然应该使用NoSQL数据库比如DynamoDB，Cassandra或者Riak

### 6.Basic System Design and Algorithm
主要的问题是怎么生成既短又独一无二的URL。比如http://tinyurl.com/jlg8zpc

a.Encodinging actual URL  
我们计算unique hash可以使用算法比如MD5或者SHA256等。这个hash的结果可以encode方便显示。encoding可以是base36 ([a-z ,0-9]) or base62 ([A-Z, a-z, 0-9]) ，如果我们想要+和/，我们可以使用Base64编码（编码作用就是去掉特殊字符，比如base64就只保留64种字符，在密码和HTTP传输上很流行）。那问题是，选择几位编码呢？6，8，还是10位？

使用base64 encoding，6位编码可以容纳64^6=~68.7billion的可能结果，如果8位编码则有64^8=~281trillion的编码。我们认为6位就足够用了。

如果使用MD5算法作为hash函数，会产生128-bit的hash结果（就是32位版本的MD5，128bit就是32个characters，也有其他版本，可以得到16个character或者24个character）。再用base64编码（编码后3个characters变成4个characters），我们就有多于21个characters（看来文章使用的是16个字符的MD5编码）。而我们只有8个character的shortURL，怎么选择key？显然会产生重复。

What are the different issues with our solution？  
如果多个用户输入同一个URL，他们总是得到同样的shorten URL，这不符合需求  

Workaround for the issues：我们可以增加一个递增序列来让每个URL变成unique，然后再进行hash。可能的问题就是这个增长序列是否出overflow？太长也会影响性能。另一种办法就是把UserID增加到URL取，不过，如果用户没有登录，可能用户要选择一个不unique的key。之后，如果还有conflict，可以一直创建知道得到一个独一无二的key。

b.Generating keys offline  
我们可以用单独的Key Generation Service（KGS）提前生成6位String，并存放在数据库中（称之为key-DB）。当需要的时候，直接从Key-DB中获取就行。这样就能让结构非常简单。

Can concurrency cause problem？  
当key被使用的时候，需要再数据库中mark一下。如果有多个servers同时读取，如果解决同步问题？KGS可以使用2个tables来存放key：一个是还没用的，一个存放所有用过的keys。每当KGS服务提供了key，那么就将之移动到used keys table。这样就避免了同步问题（还可以将备用的key放在内存来加快读取，不过如果KGS服务宕机了，预读的keys就丢失了）。  

KGS还需要保证不要将同一个key给多个servers，所以在KGS内部需要有lock的机制。

What would be the key-DB size？  
使用base64编码，我们可以最多生成68.7B的6位字符keys。一个alpha-numeric字符需要one byte，所以总共需要6 (characters per key) * 68.7B (unique keys) = 412 GB.

Isn't KGS a single point of failure？  
是的，为了解决这个问题，可以使用一个standby replica。

Can each app server cache some keys from key-DB？  
是的，会加快速度，但是如果server宕机，cache的key就没了。

How would we perform a key lookup？  
就是去DB读取完整URL，发一个HTTP 302 redirect的状态给browser，让URL自动跳转。如果shorten URL不存在，就返回HTTP 404 Not Found状态

Should we impose size limits on custom aliasas？  
我们的服务支持自定义别名。用户可以选择任何shorten URL的key。不过必须限制custom alias的size来保证数据库的URL的一致性。我们假设用户可以最多16位的自定义长度。

### 7.Data partitioning and Replication
如果需要扩展我们的DB，我们就需要分区。所以要定义好分区的schema。

a.Range Based Partitioning：  
我们通过URL hash的first letter来分区。问题是这个可能导致DB负载不平衡。比如E相对其他的字符有太多的数据。

b.Hash-Based Partitioning：  
用某个hash function将url随机的分布到不同的partition去，比如，总是分布到[1,256]的区间。这个方案仍然可能会导致分区过载问题，需要通过consistent hashing解决。

### 8.Cache
显然可以在数据库层外加上cache层，比如使用Memcached。

How much cache memory should we have?  
基于80-20原理，一开始可以cache 20%的数据，前面计算过，我们需要170G来cache每日数据。一台机器基本也可以搞定，或者用多台小机器集群来做。

Which cache eviction policy would best fit our needs?  
通常就使用LRU算法作为替换算法，自己应用就是LinkedHashMap，leetcode都有题，不多说。如果为了增加efficiency，还可以增加replicate来分布load。

How can each cache replica be updated?  
当在cache中读不到的时候就会读取数据库，这个时候就更新到每个replica去，如果该replica已经有了就跳过，如果没有就加入新entry

### 9.Load Balancer (LB)
下面几个地方可以添加LB：  
1.client和application服务器之间  
2.application服务器和数据库之间  
3.application服务器和cache之间  
最简单的LB机制就是round robin，轮流分配，因为通过心跳检测，所以如果某个机器宕机了，round robin会自动移除。不过问题是不监控实际负载，如果某台服务器的超载了，只要不宕机，LB仍然会尝试发traffic过去，解决这个问题就是周期性的发送query来监控各个服务器的load，并按照结果分配traffic。

### 10.Purging or DB cleanup
怎么处理过期的url，一种是周期性的查询并删除，显然会产生不小的花销。另外一种就是Lazy cleanup了，只有读取到的url已过期的时候删除。

### 11.Telemetry
哪些信息值得记录？比如一个url用了多少次，访问者的国家，访问时间等等

### 12. Security and Permissions
用户是否可以生成private url，或者某个url只允许特定的用户访问？  
我们可以记录每个url的访问权限（public或者private），当然也可以创建一个单独的table用来记录可以访问该url的用户列表，如果用户无权限访问就返回401错误。在存储上因为使用的是NoSQL比如Cassandra，所以key是Hash（或者KGS生成的Key），而column值就是所有有权限访问的userID。

## 总结
核心问题就是key的产生，一种是动态的MD5+Base64实时产生，不过有重复问题（可以用加上时间戳解决，不过还是有容量问题），另外一种就是独立的KGS系统预先生成。另外一个核心就是concurrency的解决，多个服务器共同访问如果处理，处理方式就是一旦请求了url，KGS就放入到已用table。至于scale部分，都是老套路，nosql可以直接partition，增加LB啊，增加cache等等。

## Reference
[md5加密以后的字符串长度](https://zhidao.baidu.com/question/70714040)
[Encode to Base64 format](https://www.base64encode.org/)
[Base64介绍](https://www.jianshu.com/p/1fe4706aa6d6)