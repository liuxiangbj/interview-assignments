#短链服务技术方案

##1、技术选型
8位最多8031810176个长链。
生成短链基本过程：
1、生成短链；
2、判断短链是否已经存在；
3、写入长链-短链映射关系。

###1.1、生成短链id生成方案选型
####1.1.1、hash函数生成短链
####1.1.2、随机函数生成短链
两者的比较：
hash函数生成的短链是带有原数据特征生成的hash，不重复的概率更高，更稳定。
随机函数生成的短链，生成速度很快，但是随着数据量变大，重复的概率不可控，不稳定。

为了防止hash冲突，分别按照一下算法顺序生成短链，如果冲突采用下一种算法生成。
md5-> crc32 -> mur_mur2 -> random
如果四种都不成功，则说明数据量已经快达到上限，需要报警出来，增加短链的位数。

###1.2、判断短链是否已经存在方案选择
####1.2.1、直接数据库查询短链是否存在
####1.2.2、使用布隆过滤器判断
两者的比较：直接查库结果最准确，但是性能瓶颈在数据库，性能比较差。
布隆过滤器较少的内存判断大量数据速度快，但是有一定的判断误差，减少一定长链的可用数量。

最终使用布隆过滤器来实现，因为是单机应用，暂时不采用redis作为统一存储，直接采用本机guava的实现。

###1.3、存储方案
持久化：mysql存储，暂时用内存数据库h2代替mysql进行持久化
缓存：本地缓存 -> redis(单机服务暂时不用) -> db
CREATE TABLE `short_url_mapping` (
  `id` bigint(20) unsigned NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `short_url` varchar(8) NOT NULL COMMENT '短链url',
  `long_url` varchar(2048) NOT NULL COMMENT '长链url',
  `create_time` bigint(20) unsigned NOT NULL DEFAULT '0' COMMENT '创建时间',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_short_url` (`short_url`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='短链-长链映射表';

##容量评估
1、数据库容量评估
一行数据<2KB, 8031810176需要160G的存储。最好需要分表存储，每张表存储不超过1G，最终分128张表比较合适。
但是在业务量没有那么大之前可以先分8张表，节省资源。本服务因为是h2内存数据库模拟，因此不做分表。

2、布隆过滤器占用内存评估
最终8000000000数据
假如有0.01的误差，大概容量需要940MB内存。
假如有0.001的误差，大概容量需要1.4GB内存。
本单机数据暂时将容量设置小一些，数据800000000，误差0.01，大概需要内存<100MB。

3、本地缓存占用内存评估
缓存个数1000000个，需要内存<200MB.

##运维方案
1、需要统计请求耗时，当服务TP99有明显上升趋势，需要调整算法策略，或者数据库增加分表。
2、统计生成短链使用超过2中hash算法的数量，以便随时调整算法策略。

##接口设计
长链接生成短链接 /long2short post请求 
根据短链接查询长链接 /short2long/{shortUrl} get请求 

##单元测试
controller层测试用spring-test mock测试
biz层测试 用power-mockito mock测试
db层测试 直接在h2内存数据库测试

##性能测试结果
环境：jvm 4C 4G 
使用jmeter 读-写接口同时压测 
10线程并发
TP99 21ms
Tp95 6ms
avg  3ms

30线程并发
TP99 44ms
Tp95 37ms
avg  14ms