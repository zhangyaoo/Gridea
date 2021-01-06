---
title: '概率相等的拼手气红包算法实现'
date: 2020-12-17 22:17:09
tags: [数据结构和算法]
published: true
hideInList: false
feature: /post-images/shou-lu-yi-ge-hong-bao-suan-fa.jpg
isTop: false
---
### 1、要求
 红包算法 function：拼手气红包
 1、每个红包获得的数学概率要一样
 2、红包最小值：1分钱

 ### 2、思路：
利用切片的思想，比如有5个红包，20分，就从0-20的范围内获取四个随机数，就被分成5份，然后顺序抽取。这样保证每个红包的的概率一样
这种方案需要考虑到几个注意点：
   1）重复的随机切片如何处理；
   2）需要考虑1分钱如何处理；

### 3、实现：
 遇到随机切片重复:重新生成切片直至不重复
 1分钱处理：判断相邻间隔的大小

 ###4、 代码实现：
 利用treeMap的顺序有序性

直接上代码（笔者已经调试过没有问题，可以运行）
 ``` java
public class RedPackage {
    /**
     * 红包最小金额
     */
    private long minAmount = 1L;

    /**
     * 最大的红包是平均值的N倍
     */
    private static final long N = 2;

    /**
     * 红包最大金额
     */
    private long maxAmount;

    /**
     * 红包金额 分
     */
    private long packageAmount;

    /**
     * 红包个数
     */
    private long packageSize;

    /**
     * 是否抢完
     */
    private boolean finish;

    /**
     * 存储红包的金额顺序
     */
    private final TreeMap<Long, Long> treeMap = Maps.newTreeMap((o1, o2) -> o1 > o2 ? 1 : o1.equals(o2) ? 0 : -1);

    /**
     * 构造函数不写业务逻辑
     */
    public RedPackage(long packageAmount, int packageSize){
        this.packageAmount = packageAmount;
        this.packageSize = packageSize;
        maxAmount = (packageAmount * N)/ packageSize;
    }

    public RedPackage(long packageAmount, int packageSize, long minAmount){
        this.packageAmount = packageAmount;
        this.packageSize = packageSize;
        this.minAmount = minAmount;
    }

    /**
     * 获取金额
     */
    public synchronized long nextAmount(){
        // 前置校验，初始化
        if(!finish && treeMap.size() == 0){
            treeMap.put(packageAmount, 0L);
            for (int i = 0; i < packageSize - 1; i++) {
                // 随机抽取切片
                long splitNum = RandomUtils.nextLong(minAmount, packageAmount);
                Long higher = treeMap.higherKey(splitNum);
                higher = higher == null ? packageAmount : higher;
                Long lower = treeMap.lowerKey(splitNum);
                lower = lower == null ? 0 : lower;
                // 相同切片重新生成,和上一个或者下一个切片间隔小于minAmount的重新生成
                while (higher - splitNum <= minAmount
                        || splitNum - lower <= minAmount
                        || treeMap.containsKey(splitNum)){
                    splitNum = RandomUtils.nextLong(minAmount, packageAmount);
                }
                // value放入上一个entry的key,组成链条，防止再次循环
                treeMap.put(splitNum, lower);
                treeMap.put(higher, splitNum);
            }
            System.out.println("init finish");
        }
        Map.Entry<Long, Long> entry = treeMap.pollFirstEntry();
        if(treeMap.size() == 0){
            // 用完红包
            this.finish = true;
            if(entry == null){
                return 0L;
            }
        }
        return entry.getKey() - entry.getValue();
    }

    public static void main(String[] args) {
        RedPackage redPackage = new RedPackage(1500L, 10, 10L);
        long result = 0;
        for (int i = 0; i < 15; i++) {
            long amount = redPackage.nextAmount()；
            System.out.println(amount);
            result = result + amount;
        }
        System.out.println(result);
    }
}
 ```

 ### 5、TODO 设置单个红包最大限额值
 目前还没有实现思路