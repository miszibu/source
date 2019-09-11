---
title: 算法_动态规划DP
date: 2018-06-14 11:18:55
tags: [算法]
---



Dynamic programming : **分阶段求解决策问题**的的数学思想。也就是说将复杂的问题简化为简单的问题来处理。动态规划有三个重要的概念，**最优子结构，边界，状态转移公式**。每个情况必须符合这三个条件方能求解。

<!--more-->

> Talk is cheap,Show me your code

话不多说，直接上题。

#### 爬楼梯问题

有一座高度是**10**级台阶的楼梯，从下往上走，每跨一步只能向上**1**级或者**2**级台阶。要求用程序来求出一共有多少种走法。 

* **最优子结构**：F(10)=F(9)+F(8) 
* **边界**：F(1)=1 F(2)=2 一个问题必须有一个边界，否则永远得不到有限的结果。
* **状态转移方程式**：F(n)=F(n-1)+F(n-2)  阶段与阶段之间的状态转移方程，这是动态规划的核心。决定了问题的每一个阶段和下一个阶段的关系。

```go
package main

import "fmt"

var step0  = 0
var step1  = 0
var step2  = 4
func main(){
	fmt.Println("递归DP：结果为",dp(10),"运行次数为：",step0)
	fmt.Println("递归DP：结果为",dp1(10),"运行次数为：",step1)
	fmt.Println("递归DP：结果为",dp2(10),"运行次数为：",step2)
}
// 递归调用解法：时间复杂度o(2^n) 空间复杂度o(1)
// 每一次调用逐级递归向下调用
func dp(height int)(value int){
	if height>1{
		step0++
		return dp(height-1)+dp(height-2)
	}else if height==2{
		step0++
		return 2
	}else {
		step0++
		return 1
	}
}
// 使用列表存储各个阶段的数据 从下往上遍历
// 时间复杂度为o(n),空间复杂度o(n)
func dp1(height int)(value int){
	stepList:=make([]int ,10)
	stepList[0]=1
	stepList[1]=2
	var i int =2
	step1+=4
	for  ; i<height; i++ {
		stepList[i]=stepList[i-1]+stepList[i-2]
		step1++
	}
	return stepList[height-1]
}
// 由于本题只需要两个数据 因此从下往上遍历时可以不保存用过的数据 更省
// 时间复杂度为o(n),空间复杂度o(1)
func dp2(height int)(value int)  {
	var a  = 1
	var b  = 2
	var i  =3
	var temp int
	for ;i<=height;i++  {
		temp=a+b
		a=b
		b=temp
		step2++
	}
	if a>b {
		return a
	}
	return b
}     

递归DP：结果为 89 运行次数为： 177
递归DP：结果为 89 运行次数为： 12
递归DP：结果为 89 运行次数为： 12
```

#### 国王和金矿

有一个国家发现了5座金矿，每座金矿的黄金储量不同，需要参与挖掘的工人数也不同。参与挖矿工人的总数是10人。每座金矿要么全挖，要么不挖，不能派出一半人挖取一半金矿。要求用程序求解出，要想得到尽可能多的黄金，应该选择挖取哪几座金矿？ 

动态规划的问题都需要**状态转移方程式**和**边界值**。此题中设N为金矿数量，W是工人数量，G为金矿的数量，P为金矿的所需入手。

**状态转移方程式**：**F(N,W)=MAX( F(N-1,W) , F(N-1,W-P(i) + G(i) ) )**

**边界值**：**F(n,w) = 0 (n<=1, w<p[0]);**

​		**F(n,w) = g[0] (n==1, w>=p[0]);**

​		**F(n,w) = F(n-1,w) (n>1, w<p[n-1])**

```go
package main

import "fmt"

var N = 5  //金矿数量
var W = 10 //工人数量
var G []int = []int{500,200,300,350,400}	//每个矿的金币数量
var P []int = []int{5,3,4,3,5}			   //每个矿的所需人手

func main(){
	fmt.Println("递归调用值为：",recursiveMine(5,10))
	fmt.Println("自底向上调用为：",bottomToTopMine(5,10))
}

func recursiveMine(n int, w int)(value int){//递归调用
	if  n<=1 && w<P[0]{
		return 0
	}else if n==1 && w>=P[0]{
		return G[0]
	}else if n>1 && w<P[n-1]{
		return recursiveMine(n-1,w)
	}
	return  max(recursiveMine(n-1,w),recursiveMine(n-1,w-P[n-1])+G[n-1])
}

func bottomToTopMine(n int ,w int)(a int){//自底向上调用，每增加一个金矿的情况都能用上一个金矿+当前金矿的值来计算 因此只要知道上一层金矿的情况 就可以计算当前金矿的最优解
	var value[10]int
	var nextValue[10]int
	var i,j = 0,1
	for ;i<w;i++{	//先给一个金矿  N个人的情况赋值
		if i+1>=P[0] {
			value[i]=G[0]
		}else {
			value[i]=0
		}
	}
	for ;j<n; j++ {	//第一行已经赋值 从第二行开始计算
		for i=0;i<10;i++{
			if i+1<P[j] {
				nextValue[i]=value[i]
			}else {
				if i+1>P[j]{
					nextValue[i]=max(value[i],value[i-P[j]]+G[j])
				}else {
					nextValue[i]=max(value[i],G[j])
				}
			}
		}
		value=nextValue
	}
	return value[w-1]
}
func max(a int ,b int)(value int){
	if a>b {
		return a
	}
	return b
}

递归调用值为： 900
自底向上调用为： 900
```

#### 相关引用

[漫画：什么是动态规划？](https://www.sohu.com/a/153858619_466939)