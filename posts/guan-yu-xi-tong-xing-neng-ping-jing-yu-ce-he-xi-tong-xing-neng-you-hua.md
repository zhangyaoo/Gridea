---
title: '系统性能优化的思考和总结'
date: 2020-06-17 21:05:11
tags: [性能调优]
published: true
hideInList: false
feature: https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1592810879330&di=84a8094e31dabae6a1cbe120e32baa9e&imgtype=0&src=http%3A%2F%2Fpcs4.clubstatic.lenovo.com.cn%2Fdata%2Fattachment%2Fforum%2F201610%2F05%2F151814x05lyhh1x0w8519w.jpg
isTop: false
---

### 前言
>   对于任何系统，都会存在系统性能瓶颈，这里笔者作为一名Java工程师列出了自己在工作中的优化思路，仅供参考。

  

## 一、系统性能预测
&emsp;任何一个系统都是从0到1慢慢发展的，当系统业务随着时间的推移，业务量和流量随之增大，系统性能就随着凸显出来。这个时候，开发人员和架构师要从架构层面、代码层面、产品业务层面等一一去演进业务系统来维持高流量下系统稳定性。

&emsp;现在服务都是微服务部署开发，如果要模拟服务压测的话要在本地开启相同的服务，前提是机器配置是一样的。而且需要将线上的持久化数据同样copy到本地数据库中，这样才能真正模拟线上的环境。拿单台机器进行压测，压测的对象可以是某个核心的接口或者业务模块（这个接口可以是日志服务中统计的访问量比较高的具体的接口API），压测的指标可以是吞吐量，平均响应时间，最大响应时间，TPS，QPS等等。

&emsp;通过性能指标可以度量目前存在的性能问题，同时作为性能优化的评估依据。具体的指标主要是要分析系统的QPS、TPS、平均响应时间以及最大响应时间，我们预测一个单体的应用能够承受多少的并发量，看这这些指标是否能够达到我们预期的值，比如作为一个健康的系统，最大响应时间不超过1s。后面进行压测时候，观察流量巅峰时刻观察系统的运行情况。以下就性能分析优化展开总结。
<br>

## 二、系统性能分析优化
### 1、硬件方面：
##### CPU：
 &emsp;在压测的时候观察CPU的占用情况，是否长期处于100%状态，正常来说80%以下是正常的。如果非常低，那么说明系统不是在做IO密集型运算动作，性能瓶颈是在其他方面，不是在CPU上面，具体的操作方法可以用top命令查看。
 &emsp;以笔者经验来看，一般CPU飙高的原因无非三种：
 1. 第一种就是代码中存在死循环，并且循环中有大量的CPU计算操作；
 2. 第二种就是多线程并发下，竞争相同的资源导致大量线程获取不到资源，如果此时线程进行自旋操作，不释放CPU资源，那就导致CPU飙升；
 3. 第三种就是代码中有内存泄漏，导致内存一直处于阈值状态，GC线程会持续GC，导致CPU飙高。
   
 &emsp;上面三种情况中第二种和第三种情况在日常开发工作中会遇到，对于第二种情况对于自旋锁情况，一般会用CAS乐观锁去实现，并且设置一定的超时时间和重试次数，然后返回失败或者进入阻塞队列释放CPU分片，防止线程一直占用CPU资源。
 &emsp;对于第三种情况，就是代码的BUG，开发过程中要注意泄漏的问题，就比如多线程操作链表，如果没有做同步的锁，那么很有可能导致链表的引用指针混乱，引起内存泄漏。
 &emsp;以上，如果我们开发工作中避免了上述几种情况，CPU就能够发挥它应该有的能力，提升系统性能。同时，开发人员做好硬件CPU监控是非常有必要的。
##### 内存：
  &emsp; 在Java中，内存JVM是一个很重要的指标，这个关乎到系统是否可以稳定运行。我们可以借助三方工具可以查看系统的JVM内存的运行情况，笔者提供几个通用的预测内存的运行情况的思路：
    - 每秒占用多少内存？
    - 多长时间触发一次Minor GC？
    - 多长时间触发一次Major GC？
    - Minor  GC耗时多久？Major  GC耗时多久？
    - 会不会频繁因为Survivor放不下导致对象进入老年代？
    
在日常开发中，开发人员需要关注的就是，判断系统JVM是否有频繁FULL GC和频繁YOUNG GC，如果有，那么会严重影响系统性能。笔者就这两个方面去分析一下
 &emsp;&emsp; 1、**频繁FULL GC** ：首先我们应该要了解到频繁FULL GC危害，一般的中大型系统，系统的JVM会设置很大，比如会给堆内存分配4~8G的空间，因为遍历对象图的过程中堆越大，遍历时间就会长，而且如果垃圾越多，垃圾回收也会拉长整个GC的时间，这就导致每一次FULL GC会有长时间的STW，影响系统稳定性。然后我们要清楚导致触发Full GC的场景，这里列出了可能会导致的几个场景：
&emsp;&emsp;1）大对象
&emsp;&emsp;2）方法区meta space空间占满
&emsp;&emsp;3）年轻代的存活的生命周期长对象一直汇入老年代，导致GC
&emsp;&emsp;4）内存泄漏导致空间不足进而GC
这里笔者就拿内存泄漏（内存泄漏指的是有引用无法被回收但是没有用的对象持续增长）来说，一般如果有内存泄漏。大概的内存监控图长这个样子![](https://zhangyaoo.github.io/post-images/1592463486063.png)
这样导致的后果就是，频繁的FULL GC，最后内存一直持续增长到爆满，然后FULL GC执行间隔缩短，最终会导致GC线程持续GC，CPU使用率会直线飙升，导致系统瘫痪。
 &emsp;&emsp;  2、 **频繁 YOUNG GC**  ：YOUNG GC如果过于频繁的话，一般是短周期小对象较多，这时候可以从 Eden 区/新生代设置的太小了这个方面考虑，看能否通过调整-Xmn、-XX:SurvivorRatio 等参数设置来解决问题
  

  **这里笔者以自己开发经验，提供一些“简单的”JVM优化拙见**：
1. 尽量将新生代的垃圾回收掉，不让存活对象进入老年代，因为老年代的GC代价比年轻代高，这里可以设置分代年龄-XX:MaxTenuringThreshold=XX
&emsp; 例子1：比如说业务上一分钟产生几百兆的数据，而且需要存活一分钟，如果一分钟YGC的次数少于默认分代年龄，那么对象会进去老年代引发FGC，FGC会引起更大的停顿时间
&emsp; 例子2：如果说对象都是一些短期对象，那么可以设置分代年龄更小，因为长期对象肯定是大对象或者单例对象永驻内存的，这样可以腾出空间给新生代GC，避免新生代频繁GC
2. 增加新生代内存的大小，防止导致频繁的minor GC，这样老年代的Major GC频率也会降低
3. 尽量将大内存的服务，拆分成几个相同服务，也就是多实例部署，分散堆内存资源，避免堆大内存导致GC时间过长（这个和G1分区回收思想相似）
4. 每个线程占用的内存不应过大或者过小，不然会导致OOM
&emsp; 如果线程内存过小，会导致线程里面的栈内存小，临时变量如果超出这个阈值就会无法分配栈，导致栈溢出，出现stackoverflow
&emsp; 如果线程内存过大，在多线程并发下，如果线程数量过多，会占用非常多JVM内存，有内存溢出的风险
5. 合理设置垃圾回收器，在大内存或者在内存碎片化环境下，G1垃圾回收器会有很好的效果
&emsp; G1垃圾回收器是Java9默认回收器，G1能够在指定的停顿时间内，根据每个region的回收价值，选择可以去回收的region，并且存活对象移动复制是多线程进行的。这里要注意如果设置停顿时间的话，不能设置太小，因为太小会导致每次进行回收的region太少，导致垃圾回收速度更不上垃圾生产的速度，这样随着时间推移，系统垃圾对象会越来越多，占满JVM
6. 对象生命周期的分布情况：如果应用存在大量的短期对象，应该适当增大年轻代 -Xmn；如果存在相对较多的持久对象，老年代应该适当增大。-Xms -Xmx
7. Xms和Xmx也设置为相同，这样可以减少内存自动扩容和收缩带来的性能损失
8. 设置大对象对象的大小，一般系统中大对象大部分都是一些系统的缓存，像这些对象尽早让它们的进入老年代，避免占用新生代的空间。

以上，合理分配JVM内存资源以及做好系统内存的监控机制是我们系统稳定性运行的保障。

##### 网络负载和IO
来一张IO发生场景图片：
![](https://zhangyaoo.github.io/post-images/1593333928024.png)
&emsp;对于磁盘IO，我们可以用Linux下的iostat命令去查看当前IO负载的情况，比如r_wait和w_wait指标，这些指标较大则说明I/O负载较大，I/O等待比较严重，磁盘读写遇到瓶颈。这个时候我们要看压测的接口是否有文件读取和写入的操作，如果有说明接口性能瓶颈在于文件读写，这个时候可以利用文件buffer缓存API等功能进行优化，或者可以用异步的方式进行文件读写。
&emsp;笔者在开发中就遇到因为IO问题带来的线程资源耗尽的线上问题：我们系统业务在借贷业务成功后，要生成借款协议，协议是一个PDF文件，当时主业务逻辑完成后同步调用生成PDF的逻辑，因为当时大流量并发，导致整个借贷业务性能瓶颈就在磁盘IO上，CPU处于空闲状态，借用网上的一个图，TOP命令可以看出IO花费的时间在76.6%，后面优化后就多线程异步处理。
![](https://zhangyaoo.github.io/post-images/1593334249512.webp)

&emsp;对于网络负载，因为网络负载或者网络堵塞是不受控制的，这个涉及到底层的TCP通信的优化（比如利用滑动窗口和拥塞控制），这个就不展开讨论。工程师可控范围可以是选择IO读写高效率的中间件，比如redis、tomcat、activeMq、nginx、dubbo、netty等，这些中间件的底层IO模型的是多路复用IO，多路复用IO指的是一个IO线程能够服务于多个socket连接，线程监听每个连接的资源描述符。如下图所示：![](https://zhangyaoo.github.io/post-images/1593334346151.png)

&emsp;分布式微服务环境下，服务之间的RPC同步调用会非常频繁，随之服务之间的网络负载会影响到整个系统的服务性能，因此，每个服务的机器放置到同一个局域网下性能效果会很好。而且，对于服务之间的调用，最好利用自研或者第三方中间件去监控服务链路调用的整体情况（比如Zipkin或者SkyWalking ）,并且要合理设置服务与服务之间的超时时间，避免因为网络原因导致服务线程池耗尽，导致OOM。


###  2、中间件层
&emsp; 这里中间件，泛指数据存储层，以笔者经验来看，大多数系统性能问题和瓶颈都是与数据存储相关，这里笔者就拿这方面展开讨论总结。
##### MySQL
一般来说MySQL在很多线程更新同一行的场景下，TPS性能曲线如图所示，参考丁奇的《秒杀场景下MySQL的低效》
![秒杀场景下MySQL的低效](https://zhangyaoo.github.io/post-images/1592893242022.png)
图中我们可以看到，线程数在6的时候TPS达到巅峰2W，随着线程数的增长，TPS会随之降低。在高并发场景下，可以根据这个结论来进行优化，比如，当有瞬间大流量冲击数据库时候，我们可以进行数据缓冲，比如用队列削峰，开启6个线程消费，然后访问数据库。
当然这个看业务场景，如果是对同一个资源进行竞争的话，这个证削峰是可行的。但是，如果场景是每一个线程对不同资源进行访问修改时候，不涉及资源竞争的话，那么就不要进行削峰处理，直接访问数据库即可，当然这个也要考虑到MySQL的性能问题。
举个例子，就拿光插入数据的性能测试来说（没有建唯一索引），4核4G的7200转的机械硬盘机器配置，最高能够承受7500的并发插入数。[参考MySQL性能压测]


以上算是一种在特定场景下的优化的思路，下面笔者讨论一下日常开发中通用的MySQL优化：
1、避免长期的事务锁占用，避免锁范围过大，避免单个资源的并发竞争
- 首先我们知道数据库Innodb存储引擎的RR和RC隔离级别下，类似update语句，锁的释放时机是在事务提交之后，这个叫做两阶段锁协议。所以为了避免事务之间锁同一行数据出现长时间的互相等待的场景，**要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放**。
举个例子，个人账户A转账给公共账户B，流程是：开启事务——》给 B 的账户余额增加钱；从账户 A 账户余额中扣除钱；记录一条交易日志——》结束事务。因为公共账户B可能被多个线程修改，所以可以优化为：从账户 A 账户余额中扣除钱；记录一条交易日志；给 B 的账户余额增加钱。
-  然后，要尽量将锁细化，一个大锁可以分割为多个锁，类似**分段锁机制**。拿笔者公司业务来说，比如APP上投资某一个产品标（包含了标的开始募集时间、结束时间、可投金额、年利率等等），在到达开始募集时间会有一段时间的高并发投标，这个时候会对具体标的行数据进行频繁的更新操作，就是扣减剩余可投金额，如果其他耗时操作中有对同一资源进行竞争的话，那么产品锁持有时间过长，导致性能低。如果有高并发秒杀下单等动作，会造成行锁抢占问题。
这个时候，优化思路是，将这个产品标在数据库分为10份，每一份的可投金额减少10倍，每一个投标请求进行随机路由分配到这10个小的产品标中。这样就减少锁的并发竞争问题，优化性能。
但是笔者因为遇到这种分段锁的问题导致的**死锁**问题，场景是这样的，当一份投标的金额大于其中一份产品金额的话，会持有这份产品的锁，并且循环获取下一份小产品，这个时候如果有两个线程都大于小产品金额的话，有概率会产生死锁问题，笔者最终通过顺序加锁以及加上锁的过期时间解决了这个问题。
- 最后，对同一个资源的并发竞争，举个例子，像12306抢票、商城活动秒杀等都是对同一个有限资源进行竞争的场景，笔者认为，这种场景是非常难处理的，需要考虑到锁同步数据安全、并发竞争性能瓶颈、超卖等问题，都是会影响C端用户实际的体验的。
  像这种场景，优化的思路就是——>**能用分段锁的就不要用悲观锁，能用乐观锁的就不要用悲观锁，能用无锁编程的就不要用锁，能用异步的场景就不要用同步的场景，能在内存操作的就不要再放到数据库磁盘层面操作**。当然，有些对于数据安全性要求很高的场景，比如金融，加锁是必要的。这个就是业务一致性和并发的折中考虑，这个需要考虑具体的业务场景。

2、关闭死锁检测
MySQL默认开启死锁检测，概念：每当一个事务被锁的时候，就要看看它所依赖的线程有没有被别人锁住，如此循环，最后判断是否出现了循环等待，也就是死锁。死锁检测对数据库有非常大的性能影响，会消耗CPU资源，最后会压垮数据库。
在这种并发场景下可以关闭死锁检测功能，会有明显的性能提升。当然关闭死锁检测也会带来问题，比如当死锁发生时，会一直持有锁资源，直至到超时时间后，释放，这段等待的时候可能会造成线程持续等待造成严重后果。所以为了避免死锁的发生，对行资源进行加锁的时候可以根据ID主键等进行**顺序加锁**。

3、SQL优化
- SQL避免多表连接查询、in和exits合理应用、考虑索引失效场景
- 在经常查询和排序的列上加索引，对离散度不高的不建议加锁，遵循索引规范
- 在写场景多余读场景的索引选择，唯一索引和普通索引的选择
- 尽量进行覆盖索引，避免回表查询
- 尽量建立联合索引，来进行索引复用
- 字符串的前缀索引的建立
- 用explain分析整个SQL的执行情况，包括执行计划、索引分析
- 修改数据比较多的字段场景尽量加索引，尽量使用行锁，避免表锁

4、在大数据量的情况下（一般单表超2000W）的优化思路：
- 加缓存，对于高并发读场景用缓存，一级缓存Redis或者二级缓存Cache
- 架构层面MySQL就做主从复制或主主复制，读写分离，可以在应用层做，效率高
- 垂直拆分，根据你模块的耦合度，将一个大的系统分为多个小的系统，也就是分布式系统；
- 水平切分，因为水平切分会增加代码开发复杂度，所以能尽量避免就不要做。针对数据量大的表，对于日志流水、配置型等类型数据，进行归档操作；对于状态业务数据进行分库分表，这里就不展开讨论。其次，要选择一个合理的sharding key，为了有好的查询效率，表结构也要改动，做一定的冗余；

##### Redis
Redis作为开发人员接触最频繁的中间件，首先，笔者先拿出官网给出的Redis性能测试结果：从下图可以得出结论：redis单机测试结果是TPS是7W,QPS只能比这个数据更高。
![](https://zhangyaoo.github.io/post-images/1592983835284.png)
当使用了管道pipline后性能大约提升了6倍，如下图所示
![](https://zhangyaoo.github.io/post-images/1592986207480.png)

根据官网的文档，影响Redis性能有以下因素，笔者认为开发人员做一些基准测试压测以及日常开发中使用Redis的时候，注意这些优化点，就能高效使用Redis：
1、网络延迟和网络带宽，作为运维人员，最好将服务器和Redis服务部署到同一个局域网内，降低网络延迟。作为开发人员，尽量不要使用大key和大value，因为随着这种数据越来越多，在网络传输的时候会占用大部分网络宽带，举个例子，如果一个redis对象大小超过1KB，当你的QPS达到100万，会把你的千兆路由器的带宽打满，因此网络带宽可能就会成为性能瓶颈。
2、使用pipline，当使用以太网访问Redis时，保据大小保持在以太网数据包大小（约1500字节）以下时，使用流水线进行聚合的命令特别有效。
3、延时删除，当某一个redisObject很大的时候，做删除操作会长时间占有线程持有时间，影响性能，redis新版本有延迟删除的功能。
4、使用scan代替keys，keys会造成严重的性能问题
5、设置内存的大小阈值并且设置好内存缓存淘汰的策略，线上设置LRU策略来淘汰缓存这样做是为了避免物理内存使用完后，造成卡顿的情况。并且线上要避免大量key同时失效的场景，因为redis删除失效的key是循环删除的，并且频繁的删除会促使内存管理器回收内存页，这样也会导致卡顿的现象。
6、connection客户端连接数量，Redis作为一个事件驱动模型，因为base epoll能够实现O(1)时间复杂度的响应操作，因此能够提供很好性能。Redis已经以超过60000个连接为基准，并且在这些条件下仍能够维持50000 q / s的吞吐量，而且具有30000个连接的实例只能处理100个连接可达到的吞吐量的一半，可参考下图（来源官网）：
![](https://zhangyaoo.github.io/post-images/1592991673063.png)

##### ElasticSearch
ElasticSearch可以解决大数据量下的搜索慢问题，这里笔者就拿 死磕ElasticSearch社区作者的优化建议，给出几点在日常开发ElasticSearch的优化方案：
1、尽量将所有数据的一半都缓存在内存当中file cache system 当中
2、将少量（查询字段比较频繁）字段放入ES，其他全量字段放入Hbase中，采用ES + Hbase方式提升查询效率，节省ES存储空间，file cache system的数据就会存的更多
3、缓存预热，可以做一个缓存预热系统，定时查询热点数据将其缓存在filesystem cache 中
4、冷热分离，将访问量高的和冷数据分别放置索引
5、ES ducoment设计，尽量避免连接、父子文档等连接操作，将数据准备好后再存入ES
6、不允许深度分页，页数越大，深度越深，从每一个shard返回的数据就越多，耗时越久。可以通过scroll api游标进行查询。
7、必须限制模糊搜索的长度，不然CPU会飙高，可参考 https://elasticsearch.cn/article/171

###  3、业务层方面
每个公司业务层面优化不相同，要根据具体业务场景去优化，别人的方案只能作为参考。
这里笔者就拿金融行业背景下，列举三个优化例子。
1、背景：企业借贷，会从用户的投的银行某个产品的资金池中匹配查找合适的资金，然后进行资金占有，银行真实转账后，生成终态的债权关系。
优化之前：使用同步锁，同一时间只能又一个线程去资金池中匹配资金，这种方案有严重的性能问题，其他线程没有拿到锁之前只能自旋尝试获取锁，损耗CPU资源。
优化之后：去掉同步锁，改成乐观锁，放到数据库做，资金表的字段增加一个标识表示是否占用，线程进来尝试匹配资金，乐观锁去预占资金表，成功表示匹配成功。这里要注意的是，尽量一个借贷匹配一比资金，这样资金池里面的资金锁行范围会减少（因为有多线程抢占资金资源），资金匹配的速度会加快，并且这样优化这样银行转账的次数会减少（如果匹配一批资金就要进行相同数量转账次数），防止多个投资人账户进行银行转账，减少整个借款业务线的耗时，避免其中一个转账出错导致全部要回滚这样的情况。
这里其实还可以进行优化，比如一笔一笔的占有资金，占有失败的continue继续下一笔资金含有，不用一次在一个事务里面占有大量资金，防止大事务执行失败以及出现其他会有死锁的可能性。这里优化的思想就是大事务拆成小事务，防止事务执行失败的概率。

2、背景：企业借贷，同步请求转异步
优化之前，三方企业借贷请求，是同步调用，因为一条完整的借贷业务线非常长，中间会RPC调用非常多的底层服务以及其他远程接口，这样的话请求到响应时间会拉的非常长，影响C端的用户体验。
优化之后，同步改异步，具体做法是，借贷请求进入系统后，会先生成一个进行中的状态借贷数据插入数据库，然后将唯一标识丢入MQ中，然后就返回成功。这样吞吐量会增加，用户端体验会非常好。
当然如果说要保证高可用，可以利用MQ的事务消息做，利用二阶段提交方式保证MQ能够收到消息。具体方案可以看笔者的这篇文章 [金融级业务下分布式事务保证数据一致性](https://zhangyaoo.github.io/post/jin-rong-ji-ye-wu-xia-fen-bu-shi-shi-wu-bao-zheng-shu-ju-yi-zhi-xing//)

3、背景：C端用户在APP上，某一个标开启募集资金后进行投标，这里的标类似支付宝的理财产品，开始募集的时候会有高并发流量涌入，当APP端用户同时投标，会有大量请求，这就形成了抢购的动作，因为一个产品标的可投金额是有限的，只有少数人能投标成功
优化之前：笔者在上文中提到的，尽管说频繁将更新行锁的数据放到事务的最后， 会有性能提升，但是随着并发数增长，MySQL也会成为性能瓶颈。
优化之后：利用redis的纯内存操作高性能的优点，将产品的可投金额放入缓存redis（这里redis里面的金额比数据库中少保证不超卖），利用redis的decrby命令或者lua脚本，保证产品标剩余可投金额能够进行原子减少，每一次减少成功后，将用户这一次投标的数据丢入MQ中异步处理，消费端做的就是将插入预状态的借款数据、冻结用户金额和减少产品的可投金额放入同一个事务中处理。
上面做的优化能够保证C端用户的良好体验，但是引入各种中间件的话会出现各种问题需要去解决，比如重复投标怎么办，Redis挂了怎么？MQ挂了怎么办？消息丢失怎么办？等一系列问题。
重复投标，可以利用用户标识的唯一token做，短信生成token，设置token失效时间（短信失效时间60s），投标时候校验token是否过期和使用过。
Redis挂，这时候就要考虑持久化和集群哨兵保持redis高可用。
MQ挂了，消息丢失，重复消费等，这时候就要考虑broker的持久化，生产端和消费端的重试机制和ack机制。
以上需要开发人员去应对每一个可能出现问题的场景。

## 三、后续流量增长系统性能优化思路
当流量激增的时候，首先要考虑到系统的稳定性和高可用，后续针对特定的场景，分析性能瓶颈，然后再去做并发的优化。这里笔者就**简单的**列举一下业界的做法，下面每一条读者都可以自行扩展大篇幅深入去了解。
#### 高可用
1. 使用反向代理和**负载均衡**实现分流，并且实现动态切换主备机器， 网关负载均衡，DNS多机房负载均衡
2. 通过**限流**保护应用免受雪崩之灾
3. 通过**降级**实现核心服务服务可用，牺牲非核心服务
4. 通过**隔离**实现故障隔离和资源隔离，比如线程隔离，对方法或者类具体分配线程数量，防止相互影响
5. 通过设置合理的**超时**调用与重试机制避免请求堆积造成雪崩
6. 通过**回滚**机制快速修复错误版本
7. Redis**集群**保证高可用，**哨兵**模式保证故障转移
8. Redis MQ消息中间件开启**持久化**，保证数据不丢；**ack机制**和**重试机制**保证数据的可靠性
9.  分布式服务环境**链路跟踪**，监控整个服务的服务质量
10. 硬件资源**监控**、CPU、内存、负载、IO、堆内存、JVM GC、线程池以及各种中间件监控等等


#### 高并发
1. 利用MQ同步转异步，流量削峰，多线程异步消费
2. 读场景比较多的接口可以利用缓存Redis，包括热点key缓存预热，多级缓存，比如分布式缓存，本地缓存和CDN缓存。还可以做集群、哨兵、高可用保证Redis性能
3. 对于强一致的同步下单场景，可以将对MySQL的操作改为Redis，然后利用MQ做异步
4. 优化JVM，包括新生代和老年代的大小、GC算法的选择等，尽可能减少GC频率和耗时
5. 非核心业务逻辑、延迟任务逻辑、三方调用逻辑，可以做异步
6. 对于架构方面，可以做负载均衡集群部署，MySQL主从、分库分表或者归档、Redis集群、多级缓存、分布式垂直拆分部署、搜索场景引入ES等多个方面考虑
7. 对于程序方面，可以从For循环的计算逻辑优化、批处理机制减少IO、采用时间复杂度更小的数据结构和算法、乐观锁和分段锁和无锁编程减少锁冲突等方面考虑

## 四、总结
以上，笔者从硬件层、代码层、中间件层、业务层等不同方向，简单的分析了影响系统性能各个因素，以及提供了简单的优化的思路和例子。因笔者工作经验能力有限，无法做到全面的分析，还望读者能够指正错误以及提供建议。


## 参考

1、秒杀场景下MySQL的低效——丁奇
2、死磕ElasticSearch社区——ElasticSearch优化
3、MySQL性能压测——https://my.oschina.net/u/867417/blog/758690
4、高可用系统方案——https://blog.csdn.net/hustspy1990/article/details/78008324
5、Redis性能测试——https://redis.io/topics/benchmarks

<!--参考 -->
[comment]: <> (6、彻底理解高并发—https://mp.weixin.qq.com/s?__biz=MzU2MTM4NDAwMw==&mid=2247484105&idx=1&sn=de4c763482aa65383dab59b221800cb5&chksm=fc78dde5cb0f54f39e1f278249d236ff2400330be573405435dba458404a5f771715319d694c&mpshare=1&scene=23&srcid=&sharer_sharetime=1593228785481&sharer_shareid=48702c70183b62662ee3edd289996b47#rd) 