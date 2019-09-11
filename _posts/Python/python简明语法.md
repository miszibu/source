---
title: python简明语法
date: 2018-09-23 00:02:41
tags: python
---

> 在shell脚本的学习编写过程中，受到了沈老师的推荐，把py作为个人的胶水语言，以当前个人的学习方向而言，Java是重点,作为工程开发语言。Golang作为区块链方向首选语言，并且打算阅读ETH和docker源码。
>
> 作为服务器脚本编程需要一门解释性语言，shell可以，但是太过于薄弱，干脆直接学了python，写脚本，写爬虫都很方便。而且python作为一门优秀的解释性语言已经有了相当强大完备的社区。

<!--more-->

## 语言逻辑 结构 函数等

```python
# -----------------字符串------------------#
# 声明字符串 单引号 双引号都可以声明字符串
message = " hello \"'world'\" "
# \n换行符 \t制表符号
author = "\n\tzibu "
# strip()==java trim()
print((message + author).strip())
# hello "'world'"
#   zibu

# -----------------数字------------------#
# ** 代表次方 +-*/通用
print(10 ** 6)
# 10000000

# -----------------数组操作------------------#
# 声明数组 增加元素 根据下标删除成员 修改 根据数据删除成员
# pop任意索引位置的数据（getAndRemove）
guestList = ['a', 'b', 'c', 'd']
guestList.append('e')
del guestList[0]
guestList[0] = "b+"
guestList.remove("c")
print(guestList.pop(0), "剩下的数组元素", guestList)
# b+ 剩下的数组元素 ['d', 'e']

# -----------------数组排序------------------#
# sort() 正序排序
guestList.sort()
# 降序排序
guestList.sort(reverse=True)
# 临时排序 不修改数组内元素位置 返回排序后的数组
sorted(guestList)
# 反向排序
guestList.reverse()
# 获取长度
len(guestList)

# -----------------for in循环------------------#
for guest in guestList:
	print(guest + "\t")
print("py没有分号和花括号，通过缩进判断for循环结束")

# 1 2 3 4 range从开头到结尾前一个
for value in range(1, 5):
	print(str(value) + "\t")

# -----------------range()-------------------#
# 输出2-10的 步长为2
# list()通过range直接生成数组
even_numbers = list(range(2, 11, 2))
print(even_numbers)

# min max sum等统计计算方法
print(min(even_numbers))
print(max(even_numbers))
print(sum(even_numbers))

# 列表解析 方括号表示
squares = [value ** 2 for value in range(1, 10)]
print(squares)
# 1, 4, 9, 16, 25, 36, 49, 64, 81]

# -----------------切片---------------------#
nameList = ['zibu', 'tom', 'jack', 'lucy', 'fancy', 'shelton']
# 输出zibu ,tom ,jack 相关索引与range类似
print(nameList[0:3])
# 从头输出到第三个元素
print(nameList[:3])
# 从第二个元素一路输出到结尾
print(nameList[1:])
# 输出倒数三个元素
print(nameList[-3:])
# 输出全部元素
print(nameList[:])
# 复制列表 通过切片会取出对应数据存放到内存中 实现deep copy
newNameList = nameList[:]

# -----------------元组---------------------#
# 元组即为不可变的数组 使用圆括号赋值
# 元组数据不可修改 但是可以通过整体重新赋值来进行覆盖
dimensions = (200, 50)
dimensions = (400, 100)
# (400, 100)
print(dimensions)

# -----------------条件判断---------------------#
# if and or  == != True False while不赘述了
# in 判断是否一个元素存在一个列表中
print("zibu" in nameList)

# -----------------字典---------------------#
# {}声明字典 []添加，获取，修改字典值 del function删除字典元素
alien_0 = {"color": "green", "point": 5}
alien_0["height"] = 10
print(alien_0["height"])
del alien_0["height"]

# for in遍历字典key value
for key, value in alien_0.items():
	print(key)
	print(value)

# for in遍历字典key 注：默认遍历字典 遍历字典的key
for key in alien_0.keys():
	print(key)

# for in遍历字典的value
for value in alien_0.values():
	print(value)

# 使用set来给字典去重
for value in set(alien_0.values()):
	print(value)


# -----------------函数---------------------#
# 函数设定默认值
# *表示接收任意长度的参数 实际上会将这些参数作为元组存放起来
def greet_user(name="zibu", *extraParam):
	print("hello " + name)
	if len(extraParam)>1:
		print(extraParam)


# 获取用户输入 作为名字传入
greet_user(input("hey ,body give me your name"))
# 使用函数默认值作为名字
greet_user()
#
greet_user("zibu", 1, 2, 3, 4, 5)

# -----------------类型转换---------------------#
int("20")
str(20)


```



---

## 模块 类 文件 测试

```python
# 模块导入
# import zibu_module
# zibu_module.func1()

# 模块导入 导入其中所有函数
# 可直接调用所有方法，但是小心方法重名 不推荐
# form zibu_module import *
# fun1()

# 模块导入 模块重命名
# import zibu_module as zibu

# 模块导入 导入部分方法
# form zibu_module import fun1,fun2


```

