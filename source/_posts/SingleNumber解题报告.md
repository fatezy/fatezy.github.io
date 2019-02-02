---
title: SingleNumber解题报告及java位运算
date: 2016-04-24 12:01:37
categories: 
- java
tags: [leetcode,java,算法]

---
#### 概述
题目概述：Given an array of integers, every element appears twice except for one. Find that single one.

中文释意：给你一个数组，里面仅有有一个数字重复了一次，其余的数字都出现了两次，找出那个仅出现了一次的数字。

注：虽然题目很水，但是解决之后对于自己对java的位运算有了新的认识，故写下这篇解题报告

<!-- more -->

##### 解法一

解题思路：先对数组进行排序，然后找出奇数项和偶数项不同的数字
评价： Arrays排序的效率并不高，故排序之后再进行遍历超时，无奈之下只好进行新的尝试。
```java
 public int singleNumber(int[] nums) {
         Arrays.sort(nums);
        if (nums.length==0)
            return 0;
        if (nums.length==1)
            return nums[0];
        for (int i = 0; i <nums.length ; i=i+2) {
            if (nums[i]!=nums[i+1]){
                return nums[i];
            }

    
        }

       return 0;
    }

```
<!-- more -->
##### 解法二

解题思路：观察数组易想到，先用set集合保存所有的数字一份，然后扩大两倍减去原数组所有数字之和，便可得到要求的结果

评价： 因为set集合底层使用的是hashmap故效率较高，并未超时

```java
 public int singleNumber3(int[] nums) {
        int sum = 0;
        for (int i:
             nums) {
            sum+=i;
        }
        Set<Integer> set = new HashSet<>();
        for (int i:
             nums) {
            set.add(i);
        }
        Integer sum2 =0 ;
        Iterator iterator  = set.iterator();
        while (iterator.hasNext()){
            sum2 += (int)iterator.next();
        }
        return sum2*2-sum;

    }

```

题目做到这，虽然做完了，但是还是效率不够高，故有了这个让人眼前一亮的第三种解法


##### 第三种解法
思路：java位运算^
评价： 此方法效率奇高，数学很重要 -.-|||

```java
 public int singleNumber2(int[] nums) {
        int res = 0;
        for(int num : nums) {
            res ^= num;
        }
        return res;
    }
```
注： [java位运算](http://fatezy.github.io/2016/04/24/java位运算/)
