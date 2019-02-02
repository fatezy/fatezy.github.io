---
title: ReverseInteger解题报告
date: 2016-06-25 21:44:52
tags: [leetcode,算法,interesting] 
categories: 算法

---

##### 题目概述： 
Reverse digits of an integer.
-  Example1: x = 123, return 321
-  Example2: x = -123, return -321

##### 题目解释： 逆序一个整数，其实这道题的难度，并不大，绝大多数人看到这题，都可以快速得到解题思路。但是这道题的核心是考察程序员对于边界条件的思考判断。

##### 解法一：
这是在leetcode讨论区看到的大牛的解法,一般需要考虑边界条件的解法，会有很多if判断，特殊条件之类的影响，写出的代码也是阅读性极差，而且容易丢三落四。这也是最考验代码水平的地方。

<!-- more -->


```
public int reverse(int x)
{
    int result = 0;

    while (x != 0)
    {
        int tail = x % 10;
        int newResult = result * 10 + tail;
        if ((newResult - tail) / 10 != result)
        { return 0; }
        result = newResult;
        x = x / 10;
    }

    return result;
}
```

##### 解法二

解题思路：题目的重点是对于末尾为0和越界这两个边界条件的判断。会导致程序异常。考虑到这点，我们可以巧妙的利用java的异常处理机制。虽然解法并不优秀。但胜在思路奇特。

```
 public static int reverse(int x) {
        if (x<0){

            String s = String.valueOf(x*(-1));
            StringBuffer stringBuffer = new StringBuffer(s);
            try {
                int result = Integer.parseInt(stringBuffer.reverse().toString());
                return result*-1;
            }catch (Exception e){
                return 0;
            }

        }else {
            String s = String.valueOf(x);
            StringBuffer stringBuffer = new StringBuffer(s);
            try {
                return Integer.parseInt(stringBuffer.reverse().toString());
            }catch (Exception e){
                return 0;
            }
        }
    }
```
