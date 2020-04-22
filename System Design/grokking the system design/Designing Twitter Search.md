## Designing Twitter Search 笔记

### What is twitter Search?
Twitter用户可以随时发推，每个推文主要包含了plain text，我们的目标就是设计一个系统允许搜索所有用户的tweets

### Requirements and Goals of the System
前提，假设Twitter有15亿用户，8亿日活用户; 每天平均有4一条推特，每条推特大概300bytes; 假设每天有5亿次搜索，搜索的语句可以包含多个words以及AND/OR条件。 我们要考虑设计系统的存储和query效率。

### Capacity Estimation and Constraints
存储：  
400M * 300 >= 120GB/day  
120GB / 24 hours / 3600sec \~= 1.38/second

### System APIs
可以用SOAP或者REST APIs，大致的API如下
```
search(api_dev_key, search_terms, maximum_restuls_to_return, sort, page_token)
```
api_dev_key(String)就是API developer注册账号了。  
search_terms(String)就是search的query了。  
maximum_results_to_return(number)就是最多返回多少个结果。  
sort(number)类似于排序种类，比如Lastest first就是0，Best matched是1，Most liked是2。  
page_token(String)就是当前的分页了

返回使用JSON格式，结果数据大致包含了UserID，username，tweet text，tweet ID, creation time, number of likes等等

### High Level Design
主要是画图，大致就是clients请求application server，然后application server从storage server取数据，这里核心是需要增加一个Index Server，用来根据Key word来快速定位推文。

### Detailed Component Design
1.Storage存储：  
每天有120G的新数据是很大的，我们要让这些数据分布到多个服务器中，如果我们计划存放5年的数据，那么需要  
120GB * 365days * 5years \~= 200TB

因为空间要有点裕富，所以大概需要250TB的总空间，加上Copy作为fault tolerance，则一共需要500TB。 当前一个服务大概存放4TB数据，我们将需要125个服务器来保留未来5年的数据。  

从最简单的设计开始，用MySQL为例，存放tweets的表有两个columns：TweetID和TweetText，我们通过TweetID分区，比如用hash function。 如果创建system-wide unique TweetIDs（全局ID）？因为每个有4亿条新tweet，所以  

400M * 365days * 5years => 730 billion  

所以需要5bytes大小的数字来让TweetIds uniquely。 形成ID后就可以通过hash function将ID均衡到服务器了。

2.Index索引  
因为推文主要是文本，我们可以创建单词到Tweet的映射。大致估算有300K个英文字和200K个名字，我们就需要500K的单词Index了。假设平均单词有5个characters，那么存放所有500K到内存需要内存：  
500K * 5 => 2.5MB  
假设我们要将过去两年的所有index存放在内存，因为5年有730B的推特，两年大概292B。因为每个TweetID占用5bytes，所有存储所有关系需要空间：  
292B * 5 >= 1460GB  
所以我们的index也会是一个大号分布式hash表，key就是word，value就是list of TweetIDs。 假设一条推特有40个单词，而且我们不保存小单词比如'the','an','and'，那么每条推特大概又15个单词需要index。 这也意味着每条TweetID要存放15次，所以需要空间：  
(1460 * 15) + 2.5MB \~= 21TB  
假设一台服务器有144GB内存，那也需要152台服务器来存放这些index。





