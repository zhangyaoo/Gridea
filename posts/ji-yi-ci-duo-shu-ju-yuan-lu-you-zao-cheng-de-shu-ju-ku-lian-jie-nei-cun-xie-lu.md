---
title: '记一次多数据源路由造成的数据库连接泄露排查过程'
date: 2021-02-02 11:32:16
tags: []
published: true
hideInList: false
feature: /post-images/ji-yi-ci-duo-shu-ju-yuan-lu-you-zao-cheng-de-shu-ju-ku-lian-jie-nei-cun-xie-lu.jpg
isTop: false
---
### 前言
之前一篇文章讲了自己写了一个多数据源路由组件给公司内部使用，进行快速迭代。文章URL是 [SaaS系统多数据源路由优雅解决方案](https://zhangyaoo.github.io/post/saas-xi-tong-duo-shu-ju-yuan-lu-you-you-ya-jie-jue-fang-an)
随着时间推移，某一天运维找上门说数据库连接打满，就刚好这一台机器上装了Java服务导致的。
下面文章就是讲的这次连接泄露导致的数据库hang住问题以及后面的解决过程。

### 一、案发当时的情况：
线上MySQL session 逐渐增加，不活跃的的数量逐渐增加，导致的后果：占用链接，导致链接满了无法分配新的连接
![](https://zhangyaoo.github.io/post-images/1612261925869.jpg)

### 二、紧急处理方式：
用自研的应用层网关将流量切换为旧的版本中，并且不停机切换，然后将当前的版本的服务停调，然后观察阿里云的数据库连接数量变回正常。

目前线上存在多个版本存在docker，然后用K8s部署，实现快速不停机切换上线。
自研网关这篇文章有介绍： [应用层网关设计和实现](ying-yong-ceng-wang-guan-she-ji-yu-shi-xian)

###  三、第一次查找问题并且解决的过程：
1、开启本地服务，让它飞一会，或者多线程并且切租户查询数据库，dump内存快照进行分析
![](https://zhangyaoo.github.io/post-images/1612253583270.png)

2、然后查看本地服务相关数据库连接的对象
        1）分析sqlsession 对象，发现sqlsession对象为0，而且日志中有打印及时关闭sqlsession。结论：sqlsession正常，说明正常关闭了sqlsession
        ![](https://zhangyaoo.github.io/post-images/1612237357285.png)
        2）分析datasource 对象，发现datasource对象和数据库连接配置中的max-pool-size一致。结论：datasource正常，说明正常关闭了datasource
        ![](https://zhangyaoo.github.io/post-images/1612237380060.png)
        3）分析connection对象，发现HikariProxyConnection对象和数据库配置一致。结论：HikariProxyConnection正常
        ![](https://zhangyaoo.github.io/post-images/1612237411657.png)

3、在分析本地内存快照，发现一个类随着时间的推移，逐渐增多，并且没有被回收，正是connectionImpl对象，这个对象是MySQL底层连接的实现，来自com.mysql.cj.jdbc
![](https://zhangyaoo.github.io/post-images/1612237433473.png)
（图片中指的类有100多个对象）

4、跟踪了一波源码，发现HikariProxyConnection对象，实质上底层就是new了一个并管理connectionImpl对象，猜测某个因为参数原因导致HikariProxyConnection及时释放，而connectionImpl没有释放，积累没有及时清除导致的。

5、经过Google查找问题，怀疑是max_lifetime导致的问题。max_lifetime官网解释：一个连接的生命时长（毫秒），超时而且没被使用则被释放（retired）。建议比数据库的wait_timeout小几分钟。默认30分钟
![](https://zhangyaoo.github.io/post-images/1612237640791.png)

6、查看线上配置和测试环境配置，果然是只配置了1分钟。![](https://zhangyaoo.github.io/post-images/1612262047685.jpg)
随后将其改成30分钟，然后继续多线程并发跑测试用例。测试环境验证，dump内存，观察connectionImpl对象并没有随着时间的推移增加。发现connectionImpl对象并没有随着时间的推移增加。验证了个人的猜想。
![](https://zhangyaoo.github.io/post-images/1612237699007.png)

7、最终将配置更新后上线，过了没2个小时，连接数又飙升。初步观察还是connectionImpl对象增多，连接没有释放。所以认为，不是max_lifetime配置的问题。还是得从源码中入手和线上的dump入手看。

第一次查找问题并**没有解决根本问题**，对此总结了一下：
1. 自己的猜想缺少实际的数据支持和多方位的有力证明
2. 对源码研究不够深入，只是停留在表明
3. 对官网的配置参数理解不够透彻

###  四、第二次查找问题并且解决的过程：
本地环境观察没有问题，正式环境观察就有问题。像这种问题算是比较难解决的，为了快速解决问题，避免把线上数据copy到本地进行测试，直接down线上的dump文件下来进行仔细分析。

1、查找线上的内存，dump数量，刚好100个，并且随着时间偏移，这个类对象越来越多，个人猜想就是这个对象没有被回收导致连接数未释放。有强引用引用这个对象。
![](https://zhangyaoo.github.io/post-images/1612420897419.png)

2、查找这个对象GC root对象，右键选择Merge Shortest Paths to GC roots -> exclude all phantom/weak/soft etc.reference(排除所有虚弱软引用)，发现这100个对象大部分被一个名字叫做housekeeper线程所强引用，如下图。个人验证猜想是线程没有及时回收关闭或者是没有关闭线程池。
![](https://zhangyaoo.github.io/post-images/1612423498302.png)

3、这个时候dump文件里面就没法发现更多有用的信息了，然后就去看源码看这个housekeeper线程为什么一直强引用。查找源码发现在获取datasource 的 getConnetcion方法，会初始化HikariPool连接池。
![](https://zhangyaoo.github.io/post-images/1612423896839.png)

初始化连接池里面会初始化housekeeper连接池，刚好对应了上面在根引用的对象。
![](https://zhangyaoo.github.io/post-images/1612423966034.png)
![](https://zhangyaoo.github.io/post-images/1612425624943.png)

看了下这个housekeeper对象的作用，目的是为了维持最少的空闲的连接，说白了就是根据配置参数idleTimeout和minIdle来维持最小的空闲连接数。
![](https://zhangyaoo.github.io/post-images/1612425545450.png)

4、至此，可以初步得出结论，是由于应用程序不停的new HikariPool，然后没有及时close导致的问题。然后顺着这个结论去查看应用程序的BUG

5、在程序发现下面逻辑，下面这一段逻辑就是根据租户来获取缓存中租户对应的datasource对象
```java
   /**
     * 获取已缓存的数据源
     */
    private Optional<DataSource> getDataSourceIfCache(SelectRdsConfigDto rdsConfigDto) {
        String key = getUniqueMysqlKey(rdsConfigDto.getHost(), rdsConfigDto.getPort());
        if (Objects.nonNull(dataSourceCachePool.get(key))) {
            DataSourceCache dataSourceCache = dataSourceCachePool.get(key);
            //cache中已经缓存了租户的连接并且没有修改rds config信息
            if (dataSourceCache.verifyRdsConfig(rdsConfigDto)) {
                return Optional.ofNullable(dataSourceCachePool.get(key).getDataSource());
            }
            //cache中已经缓存了租户的连接，但是校验不通过
            else {
                dataSourceCachePool.remove(key);
                return Optional.empty();
            }
        }
        return Optional.empty();
    }

   // 校验
   private boolean verifyRdsConfig(SelectRdsConfigDto rdsConfigDto) {
        return rdsConfigDto.getAccount().equals(this.account) &&
                rdsConfigDto.getHost().equals(this.host) &&
                rdsConfigDto.getPort().equals(this.port) &&    rdsConfigDto.getPwd().equals(this.pwd) &&；

    //  dataSourceCachePool的key组成，一个MySQL连接对应的key
    private String getUniqueMysqlKey(String host, Integer port){
        return host + ":" + port;
    }

 }
```
这里逻辑有一段是校验不通过，会remove对应的datasource对象，问题就出现在这里，这里没有及时close。校验不通过的原因就是，一个host port作为一个dataSourceCachePool的key，因为线上有一个MySQL实例多个账号，导致校验总是不通过误以为成是修改了用户名或者密码，随后就是一直new datasource对象。


###  五、根本原因
所以综上，最终得出结论是，因为程序问题导致HikariDataSource对象增多，而且因为HikariDataSource对象内部有一个线程池，如果外部丢失了对这个HikariDataSource对象的引用，也不会被垃圾回收，导致HikariDataSource对象不释放，然后结果就是数据库连接未释放。

### 六、最终处理方式：
1、修改代码：
1） dataSourceCachePool的key组成由host port 改成 host port account pwd 四个维度作为一个key。

2）关闭datasource对象，remove那段代码的逻辑修改成下面这个样子
```java
//cache中已经缓存了租户的连接,但是修改了rds config信息
        DataSource dataSource = dataSourceCachePool.remove(key).getDataSource();
        log.info("remove datasource:{}, {}", key, rdsConfigDto.toString());
        new Thread(() -> {
            while (true) {
                try {
                    TimeUnit.SECONDS.sleep(10);
                } catch (InterruptedException e) {
                    log.error("e:", e);
                }
                if (dataSource instanceof HikariDataSource) {
                    if (((HikariDataSource) dataSource).getHikariPoolMXBean().getActiveConnections() > 0){
                        log.info("ActiveConnections > 0, continue");
                        continue;
                    }
                    log.info("HikariDataSource close...");
                    ((HikariDataSource) dataSource).close();
                } else if (dataSource instanceof DruidDataSource) {
                    if (((DruidDataSource) dataSource).getActiveCount() > 0) {
                        log.info("ActiveConnections > 0, continue");
                        continue;
                    }
                    log.info("DruidDataSource close...");
                    ((DruidDataSource) dataSource).close();
                } else {
                    log.error("close datasource|datasource is wrong");
                    throw new RuntimeException("close datasource|datasource is wrong");
                }
                break;
            }
        }).start();
```

Q：为什么要判断活跃的连接 ActiveCount > 0 ？
A：因为close的时候要判断是否有正在使用的connection对象，如果强制关闭，那么会出现一个线程查询的时候，connetion突然不可用，导致错误。

2、改配置，改成和官方默认的配置
![](https://zhangyaoo.github.io/post-images/1612426469625.png)

最后，重新上线后，观测对象内存数据，正常。

### 七、总结
底层的数据库组件的代码编写，要注意使用数据库连接资源的时候，一定要检查代码中是否释放，不然会造成严重的事故。而且平常还要熟悉官方配置，并且多研究源码。以免出现这种情况时束手无策。
可以的话还可以叫身边的资深大牛给review代码，做到双重保障。