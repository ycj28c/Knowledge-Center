## Designing Youtube or Netflix 笔记

### 确认需求
功能需求：  
1. 用户需要能上传视频
2. 用户需要能够分享和查看视频
3. 用户需要能够根据视频名称搜索
4. 我们服务能够记录视频状态，比如like/dislike，total views等
5. 用户能够增加评论

非功能性需求：  
1. 系统高可靠，视频上传不丢失
2. 系统高可用，一致性可能受影响
3. 用户实时看视频，感受不到lag

### 容量估算和约束
假设1.5billion用户，800million每日活跃用户。加入每个用户每日平均看5个视频，那么每秒 
``` 
800M * 5 / 86400 = 46k videos/sec
```
假设我们的上传观看率是1：200，每个视频有200观看数，那么需要至少上传230视频每秒
```
46K / 200 = 230 videos/sec
```

存储估算：假设每分钟有500小时的视频上传到youtube，平均每分钟的视频需要50mb的空间（需要存放为多种格式），那么总共空间需要
```
500 hours * 60 min * 50MB >= 1500GB/min(25GB/sec)
```
这个估计是在忽略了压缩和复制备份的前提下

带宽估算：每分钟500小时的视频，假设每个视频占用10MB/min带宽，我们需要300G的上传带宽每分钟
```
500 hours * 60 mins * 10MB >= 300GB/min(5GB/sec)
```

### 系统API设计
我们可以用SOAP或者REST APIs来展示服务功能，比如

上传函数：
```
uploadVideo(api_dev_key, video_title, vide_description, tags[], category_id, default_language, 
                        recording_details, video_contents)
```
这个参数可以自己设计，返回值是String，如果接受返回HTTP 202(request accepted)，当视频encode结束就发送email。

搜索函数：
```
searchVideo(api_dev_key, search_query, user_location, maximum_videos_to_return, page_token)
```
参数可以自由发挥，返回值可以是JSON，包含了视频列表，每个视频可以有title，thumbnail，create date等

Stream函数：（跳播）
```
streamVideo(api_dev_key, video_id, offset, codec, resolution)
```
返回的是stream，也就是在某个offset的视频chunk

### 高级设计
1. Processing Queue： 每个上传的视频都会先放入queue中，等轮到了再进行后续的encode，生成thumbnai，存储等
2. Encoder： 将一个视频转换为多种格式
3. Thumbnails generator： 生成视频信息
4. Video and Thumbnail storage： 将视频和缩图存放到分布式系统
5. User Database： 存放用户信息，比如name，email等
6. Video metadata storage： 视频元数据库，记录格式化的数据比如title, file path，uploading user, total views等，还有所有的comments

### 数据库设计
主要是metadata storage的设计，可以如下：

Videos表:  
VideoID  
Title  
Description  
Size  
Thumbnail  
Uploader/User  
Total number of likes  
Total number of dislikes  
Total number of views  

Comments表：  
CommentID  
VideoID  
UserID  
Comment  
TimeOfCreation  

User表：
UserId  
Name  
Email 
Address  
Age  
Registration Detail  

### 详细设计
