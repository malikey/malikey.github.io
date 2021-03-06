---
layout:     post
title:      "算法设计技术基础"
subtitle:   "分治、动态规划、贪心、回溯 归纳笔记"
category : basictheory
date:       2017-02-02
author:     "Max"
header-img: "img/post-bg-algorithms.jpg"
catalog:    true
tags:
    - algorithms
---

前一段时间回顾了[算法设计与分析](https://www.coursera.org/learn/algorithms/home/welcome)，
现对分治、动态规划、贪心、回溯四种基础算法设计技术做一个归纳。


# 1. 分治策略

## 1.1 适用条件

原问题能够规约为独立求解的子问题。

## 1.2 主要步骤

1. 将原问题规约为独立求解的子问题。注意子问题要划分均衡，且与原问题性质相同。
2. 计算初始子问题。
3. 综合子问题的解，得到原问题的解。

## 1.3 简单实例

快速排序是分治策略的典型应用。

给定乱序数组A[1...n]，利用首元素将原问题划分为两个独立子问题。伪码如下：

```
QuickSort(A, p, r)
{
  if p < r
    then q <- Partition(A, p, r)
         QuickSort(A, p, q-1)
         QuickSort(A, q+1, r)

  return A
}

Partition(A, p, r)
{
  x <- A[p]
  i <- p
  j <- r+1
  while true do
    repeat j <- j-1 until A[j] <= x
    repeat i <- i+1 until A[i] > x
    if i < j
      then exchange A[i] <-> A[j]
      else exchange A[j] <-> A[p]
           return j
}
```
最初调用是QuickSort(A, 1, n)。

## 1.4 复杂度分析

一般用递归来实现，求解递推方程。

上述实例，最坏情况下划分出规模为n-1和0的两个子问题，$W(n)=W(n-1) + n - 1$，$W(1) = 0$，时间复杂度是$O(n^2)$。

## 1.5 改进途经

1. 减少独立计算的子问题个数
2. 增加预处理

## 1.6 典型用例

二分检索，归并排序， 芯片测试，幂乘，矩阵乘法， 最邻近点对，多项式求值等。

# 2. 动态规划

## 2.1 适用条件

求解过程是多阶段决策过程，每步处理一个字问题，可用于处理组合优化问题；

问题要满足优化原则或最优子结构，即一个最优决策序列的任何子序列本身一定是相对于子序列的初始和结束状态的最优子序列。

## 2.2 主要步骤

1. 确定子问题边界
2. 确定将问题归结为更小子问题的方法
2. 列目标函数的递推方程及初值
3. 确定计算顺序
3. 备忘录存储
4. 标记函数及解的追踪

## 2.3 简单实例

（二叉检索树问题）给定集合S为排序的n个元素 $x_1 < x_2 < ... < x_n$，将这些元素存储在一棵二叉树的节点上，以查找x是否在这些数中；若不在，确定x在哪个空隙。

将子问题边界定为$(i,j)$，对应数据集 $S[i,j]=<x_i, x_{i+1}, ... , x_j>$ ，概率分布
 $P[i,j]=<a_{i-1}, b_i, a_i, b_{i+1}, ... , b_j, a_j>$ （a对应空隙概率，b对应节点概率）。令 $ w[i,j] = \displaystyle\sum_{p=i-1}^{j} a_p + \displaystyle\sum_{q=i}^{j} b_q $ 是$P[i,j]$中所有概率之和， $m[i,j]$是最优二叉检索树的平均比较次数。

选定 $x_k$ 为根，即可把问题规约为两个子问题，$S[i,k-1]$，$P[i,k-1]$ 和 $S[k+1,j]$，$P[k+1,j]$。这样得到$m[i,j]$的递推公式：

$$
m[i,j]=\min \left\{ m[i,k-1] + m[k+1,j] + w[i,j] \right\}   ,  1 \leq k \leq j \leq n \\\
m[i,i-1]=0 ,  i=1,2,3,\cdots
$$

使用 $j\geq i-1$ 的表项$m[i,j]$和表$root[i,j]$来记录序列$S[i,j]$对应子树的根。为避免重复计算，设计备忘录表 $w[1..n+1, 0..n]$。


## 2.4 复杂度分析

上述实例中，$i$、$j$共$n^2$种组合，备忘录有 $\Theta(n^2)$ 项（空间复杂度），每项需要常数时间计算。综合整个过程，时间复杂度在 $\Theta(n^3)$ 量级。

## 2.5 典型用例

矩阵链相乘，投资问题，背包问题，最长公共子序列，图像压缩，最大子段和，最优二分检索树，
生物信息学应用等。

# 3. 贪心法

## 3.1 适用条件

组合优化问题，多步判断求解。

问题要满足贪心选择性质，即一个全局最优解可以通过局部最优解（贪心）选择来达到。

## 3.2 主要步骤

1. 确定局部优化策略
2. 算法正确性证明（直接证明，数学归纳法，交换论证）

## 3.3 简单实例

霍夫曼设计了一种构造最优前缀码的贪心算法。

假定C是一个包含n个字符的集合，每个字符c出现的频度为f[c]。算法以自底向上的方式构造出最优编码所对应
的树T。Q是一个以f为关键字的最小优先级队列，用来识别出要合并的两个频度最低的对象。

```
HUFFMAN(C)
{
  n <- |C|
  Q <- C
  for i <- 1 to n-1
    do allocate a new node z
      left[z] <- x <- EXTRACT-MIN(Q)
      right[z] <- y <- EXTRACT-MIN(Q)
      f[z] <- f[x] + f[y]
      INSERT(Q, z)
  return EXTRACT-MIN(Q)
}
```

## 3.4 复杂度分析

上述实例中，假设Q是最为最小二叉堆实现的，对于包含n个字符的集合C，初始化在$O(n)$时间完成。for循环
执行$n-1$次，每次堆操作需要$O(\lg n)$时间，整个循环需要$O(n \lg n)$时间。这样，HUFFMAN(C)算法的总运行
时间是$O(n \lg n)$。

## 3.5 典型用例

活动选择，装载问题，最小延迟调度，最优前缀码，最小生成树，单源最短路径问题等。

# 4. 回溯与分支限界

## 4.1 适用条件

求解搜索或优化问题，多步判断求解，满足多米诺性质。

多米诺性质可以描述为若$k+1$维向量 $<x_1, x_2, ..., x_{k+1}>$ 满足约束条件，则$k(0<k<n)$维向量 $<x_1, x_2, ..., x_k>$
也满足约束条件。其逆否命题同样成立，即若$k(0<k<n)$维向量 $<x_1, x_2, ..., x_k>$ 不满足约束条件，
扩张向量至$k+1$维仍旧不满足，可以回溯。

## 4.2 主要步骤

1. 确定可行解解结构
2. 确定搜索树结构和搜索顺序
3. 确定节点分支搜索的约束条件与代价函数
4. 确定存储路径

分支限界，即停止分支回溯父节点的依据，是节点不满足约束条件，或者对于极大化问题节点的代价函数小于当前界（极小化问题相反）。

## 4.3 简单实例

给定n种不同面值的邮票，每个信封最多贴m张，试给出邮票的最佳设计，使得从1开始，增量为1的连续邮资区间达到最大。

##### 可行解

n维向量 $<x_1, x_2, ..., x_n> , x_1=1, x_1<x_2<...<x_n$

##### 搜索策略

深度优先

##### 约束条件

在节点 $<x_1, x_2, ..., x_i>$ 处，邮资最大连续区间为 ${1, ..., r_i}$ ，$x_{i+1}$
的取值范围是 $ {x_i+1, ..., r_i+1} $。若 $ x_{i+1} > r_i+1 $，$ r_i+1 $的邮资将没法支付。

若 $ y_i(j) $ 表示用最多m张面值 $x_i$ 的邮票加上 $x_1,x_2,...,x_{i-1}$ 面值的邮票贴 $j$
邮资时的最少邮票数，则 $ r_i $ 满足如下条件：

$$
y_i(j) = \min \left\{  t+y_{i-1}(j-tx_i) \middle| 1 \leq t \leq m \right\}  \\
y_1(j) = j  \\
r_i = \min \left\{ j \middle| y_i(j)\leq m, y_i(j+1)> m \right\}  
$$

##### 存储路径

以一维数组`x[n]`来记录已确定的邮票面值，`r`表示此时m张邮票连续付的最大邮资。最优情况存储为`ans[n]`和`max`。

##### 伪代码

关键步骤BACKTRACK(i)表示1..x[i-1]面值已确定的情况下，搜索x[i]的解空间以寻找最优解。

因为对于每个可行解的y、r各不相同，需要注意回溯过程中y、r的备份与恢复。

```

BACKTRACK(i)
{
    if i == n
      then if r > max
             then max <- r
                  for j <- 0 to n-1
                      do ans[j] <- x[j]
           return

    for next <- x[i-1]+1 to r+1
      do x[i] <- next
         BACKUP_Y_R();

         for j <- x[i] to x[i]*m
            do for t <- 1 to m
                 do index <- j-x[i]*t
                    if index >= 0
                      then cost <- t+y[index]
                           if y[j] > cost
                             then y[j] <- cost;

          j <- 0;
          while y[j] <= m
            do r <- j++

         BACKTRACK(i+1);
         RESTORE_Y_R;     
}

```

## 4.4 复杂度分析

时间复杂度是每个节点计算量与节点数的乘积，最坏情况下与蛮力算法无异，但是平均看来较好。

空间复杂度较低。

## 4.5 典型用例

N皇后问题，背包问题，货郎问题，装载问题，最大团问题，圆排列问题等。
