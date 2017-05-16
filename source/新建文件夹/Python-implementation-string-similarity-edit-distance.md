title: Python实现字符串相似度-编辑距离
mathjax: true
tags: [Python3]
categories: Algorithms
date: 2015-07-07 21:14:59

---
###编辑距离
关于维基上的定义参见[这里](https://zh.wikipedia.org/zh-cn/%E7%B7%A8%E8%BC%AF%E8%B7%9D%E9%9B%A2)。编辑距离，又称`Levenshtein`距离，是指两个字串之间，由一个转成另一个所需的最少编辑操作次数。通常许可的编辑操作包括:
* 将一个字符替换成另一个字符
* 插入一个字符
* 删除一个字符

例如将kitten一字转成sitting：
* sitten （k→s）
* sittin （e→i）
* sitting （→g）

俄罗斯科学家[`Vladimir Levenshtein`](https://zh.wikipedia.org/wiki/Vladimir_Levenshtein)在1965年提出了这个概念。

###算法
动态规划经常被用来作为这个问题的解决手段之一。伪代码如下
```
整数 Levenshtein距离(字符 str1[1..lenStr1], 字符 str2[1..lenStr2])
   声明 int d[0..lenStr1, 0..lenStr2]
   声明 int i, j, cost

   for i = 由 0 至 lenStr1
       d[i, 0] := i
   for j = 由 0 至 lenStr2
       d[0, j] := j

   for i = 由 1 至 lenStr1
       for j = 由 1 至 lenStr2
           若 str1[i] = str2[j] 则 cost := 0
                                否则 cost := 1
           d[i, j] := 最小值(
                                d[i-1, j  ] + 1,     // 刪除
                                d[i  , j-1] + 1,     // 插入
                                d[i-1, j-1] + cost   // 替换
                            )

   返回 d[lenStr1, lenStr2]
```
###Python实现
```python
# -*- coding: utf-8 -*-
"""
Created on 2015/7/7  10:08
使用动态规划算法实现编辑距离的计算
@author: Wang Xu
"""
import numpy as np


class LevenshteinDistance:
    def leDistance(self, input_x, input_y):
        xlen = len(input_x) + 1  # 此处需要多开辟一个元素存储最后一轮的计算结果
        ylen = len(input_y) + 1

        dp = np.zeros(shape=(xlen, ylen), dtype=int)
        for i in range(0, xlen):
            dp[i][0] = i
        for j in range(0, ylen):
            dp[0][j] = j

        for i in range(1, xlen):
            for j in range(1, ylen):
                if input_x[i - 1] == input_y[j - 1]:
                    dp[i][j] = dp[i - 1][j - 1]
                else:
                    dp[i][j] = 1 + min(dp[i - 1][j], dp[i][j - 1], dp[i - 1][j - 1])
        return dp[xlen - 1][ylen - 1]


if __name__ == '__main__':
    ld = LevenshteinDistance()
    print(ld.leDistance('瓦罐蹄膀饭', '瓦罐焖蹄饭'))  # Prints 2
    print(ld.leDistance('', 'a'))   # Prints 1
    print(ld.leDistance('b', ''))   # Prints 1
    print(ld.leDistance('', ''))    # Prints 0
    print(ld.leDistance('杭椒小炒肉面', '外婆小肉面'))  # Prints 3
    print(ld.leDistance('外婆小肉面', '杭椒小炒肉面'))  # Prints 3
```
