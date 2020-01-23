---
title: leetcode习题总结
date: 2020-01-08
categories:
- coding
tags:
- test
---

### [Remove Comments](https://leetcode.com/problems/remove-comments/)

* 目标：对给定的一个c++代码字符串数组，删除其中注释内容

* 思路：没有简单思路，分类目处理

	1. 对每一个字符串先检测其中是否含有符号"//","/* */"
	
	2. 对"//"删除该字符后面内容，前面的放到vector中
	
	3. 对"/*"则记一个全局标志位，如果这个标志位存在，则跳过任何字符
	
	4. 对"*/"则去除全局标志

	
### [Simplify Path](https://leetcode.com/problems/simplify-path/)

* 目标：给一个绝对路径字符串，将其简化

* 思路：先处理特殊情况，再按"/"分割处理即可

	1. 先处理3种特殊情况"//"，对这种符号只需简单替换即可
	
	2. 再按"/"分割字符串，一个一个处理即可

hash table：

### [Longest Substring Without Repeating Characters](https://leetcode.com/problems/longest-substring-without-repeating-characters/)

* 目标：对给定的字符串，找到最长无重复子字符串

* 思路：滑动窗口+Hashtable
	
	1. 使用left,right双指针遍历字符串；right指针主动检测下一个字符，left指针由right检测到重复元素时，被动更新；hashtable中保存不同字符最后一次出现的位置
	
	2. 当right指针主动向右扩展，首先判断新元素有没有在hashtable中，如果是，则将left更新为hashtable中保存的值；否则，检测目前目前[left,right]字符串是否最长

* 失败思路：

	1. 双重for循环，对以字符串中每个字符为起点，计算它的最长无重复字符串；即可找到最大者。算法复杂度为O(n2)，无法AC
	

	
stack

贪婪算法

### [Delete Columns to Make Sorted II](https://leetcode.com/problems/delete-columns-to-make-sorted-ii/)

* 目标：对给定字符串数组，找一组序列；使得字符串数组中每个字符串以序列为下标删除字符后，为升序

* 思路：按下标贪婪，分别计算按每一下标为首的升序排列需要删除的列数即可



回溯算法

### [Additive Number](https://leetcode.com/problems/additive-number/)

* 目标：判断给定数字字符串是否为"加法数"

* 思路：使用数组装载加法数，对下标进行回溯

	1. 递归参数：已经满足加法数的数组、寻找下一个加法数起始下标、是否已经找到、字符串
	
	2. 在递归函数中，对数字进行贪婪即可；满足out.size() >= 2 && curNum != out[n - 1] + out[n - 2]，即可再找下一个数
	
动态规划

### [Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray/)

* 目标：给定一个数组，找出其中乘积最大子数组

* 思路：用两个DP数组，max[i]：子树组[0,i]范围内并且一定包含nums[i]数字的最大子树组乘积；min[i]：子树组[0,i]范围内并且一定包含nums[i]数字的最小子树组乘积

	1. max[0]=min[0]=nums[0]
	
	2. 递推：max[i] = max{max[i-1]*nums[i], min[i-1]*nums[i], nums[i]},min[i] = min{max[i-1]*nums[i], min[i-1]*nums[i], nums[i]}
	
	3. 结果就是max数组中最大值

* 失败思路：有点类似Longest Substring Without Repeating Characters，第一眼看着就是要滑动窗口

	1. 两个for循环，分别寻找以各个点为起始点的子数组最大乘积，复杂度：O(n2)
	
### [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/)

* 目标：给定字符串，找出其中最长的回文子字符串

* 思路：DP,建立递推关系, dp[i][j]表示字符串区间[i,j]是否为回文串

	1. dp[i,j] = 1                                      if i == j

    2. dp[i,j] == (s[i] == s[j])                        if j = i + 1

    3. dp[i,j] == (s[i] == s[j] && dp[i + 1][j - 1])    if j > i + 1     

* 失败思路：对以每个字符为中心，向两边扩展找到最长回文子字符串，复杂度：O(n2)

### [coin change](https://leetcode.com/problems/coin-change/)

* 目标：类似背包问题，但是这个问题是要确定N个coin能否组合出M块钱

* 思路：DP，dp[i]表示钱数为i时的最小硬币数

	1. 递推：两个for循环，外层对钱数i循环，内层对硬币种类循环；dp[i] = max{dp[i], dp[i-coin[j]]+1}
	
	2. 由于是两层for循环，在dp[i-coin[j]]中，已经是考虑包含coin[j]后的最小硬币数，因此不用像完全背包问题，还需要考虑coin[j]的数量
	
	
### 0/1背包问题

* 目标：有n件物品和容量为m的背包，给出n件物品的重量w[i]及价值v[i]，求解放入背包的物品重量不超过容量m的最大价值

* 思路1：DP，nums[i][j]表示前i种物品能组成不超过j重量的最大价值

	1. 递推公式：nums[i][j] = max{nums[i-1][j-w[i]]+v[i], nums[i-1][j]}
	
	2. nums[i-1][j-w[i]]+v[i]：表示在加入i物品不会超重的前提下，加入i物品后的最大价值
	
* 思路2：DP：f[j]表示重量j情况下最大价值

	1. 递推公式：随着遍历所有物品，f[j] = max{f[j], f[j-w[i]]+v[i]}


### 完全背包问题

* 目标：类似0/1背包，但是所有物品可以无限选用

* 思路：DP，nums[i][j]表示前i种物品能组成不超过j重量的最大价值

	1. 递推公式：nums[i][j] = max{nums[i-1][j-k*w[i]]+k*v[i], 0<=k && k*w[i]<=j}
		
	2. 复杂度：f[j] = max{f[j], f[j-k*w[i]]+k*v[i], 0<=k && k*w[i]<=j}
	

### 多重背包问题

* 目标：类似完全背包问题，但是所有物品有一定数量限制num[i]

* 思路：类似完全背包，k值限制即可



递归

### [Valid Tic-Tac-Toe State](https://leetcode.com/problems/valid-tic-tac-toe-state/)

* 目标：给定一个3*3棋盘，分析棋盘是否是正确的；X先走，一旦横、竖、斜3连子，则游戏结束

* 思路：比较直观的思路就是两个for循环遍历棋盘，对于几种不可能的情况逐一检查即可

	1. 用一个变量turns来标识下棋顺序；读到X时，turn++；读到O时，turn--；最后turn必为0/1，否则异常
	
	2. 用xwin、owin分别标识按棋盘看X、O是否赢了；当然X赢时，turn必为1，否则也异常
	
	
###
