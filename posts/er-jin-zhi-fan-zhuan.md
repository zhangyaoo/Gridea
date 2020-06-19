---
title: 'O(N)时间复杂度下，二进制反转'
date: 2020-06-19 16:11:36
tags: [数据结构和算法]
published: true
hideInList: false
feature: 
isTop: false
---
#### 题目描述
给定一个32位整数 . 输出二进制表示反转后的值.
例如 input 43261596（二进制 00000010100101000001111010011100）
返回 output 964176192（二进制 00111001011110000010100101000000）

目前笔者就想到了时间复杂度在O(N)的解决思路：
1. 循环判断输入数据的低位是0还是1，具体判断方法是和1进行与操作
2. 如果判断是，返回的结果+1，不是1那么不做任何处理
3. 每次循环，input的数据向左移一位，output数据向右移动一位
4. 循环32次，返回结果
   
```java
/**
    *  二进制数据反转
    */
public class BitReverse {

    public static int reverse(int n) {
        int result = 0;
        for (int i = 0; i < 32; i++) {
            result = result << 1;
            if ((n & 1) == 1) {
                result++;
            }
            n = n >> 1;
        }
        return result;
    }

    public static void main(String[] args){
        System.out.println(reverse(1<<30));
        System.out.println(1<<30);
    }
}
```

