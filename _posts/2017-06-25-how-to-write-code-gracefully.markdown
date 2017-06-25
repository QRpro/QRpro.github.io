---
layout:     post
title:      " 如何更优雅的撸码?! "
subtitle:   "持续更新..."
date:       2017-06-25 
header-img: "img/post-bg-tips.jpg"
catalog:    true
author:     "Neil"
tags:
    - Tips
---  


## 写在前面的话
> 本文是个人在学习、工作中发现的一点小技巧，个人觉得不错就贴上来了。  
> 如果哪里描述的不准确请及时与我联系更正！  
> 本文持续保持更新！  

## 目录  
1.  [返回空集合](#empty)
2.  [SQL行转列](#listTorow)
3.  [变量命名](#abbreviation)
4.  [遍历Map](#iteratorMap)
5.  [大于两个true返回](#return)
6.  [减少变量重复计算](#chongfujisuan)
7.  [循环内不要不断创建对象引用](#objectref)
8.  [字符串常量equals的时候写在前面](#strequals)
9.  [COUNT(*)与COUNT(1)](#count)
10. [SQL中的SUM(expr)](#sumexpr)

## 详细描述

<span id='empty'> 
**1. 在返回Collection中有可能为空时请返回**

```java
Collections.emptyList();
Collections.emptyMap();
Collections.emptySet();
```

<span id='listTorow'>
**2. 在查询中如果多个值分开获取后进行统计请尝试使用行转列思维**

```sql
SELECT Date,
MAX(CASE 列名 WHEN 列值 THEN XX ELSE 0 END ) 别名,
MAX(CASE 列名 WHEN 列值 THEN XX ELSE 0 END ) 别名 
FROM TabName  
GROUP BY Date
```


<span id='abbreviation'>
**3. 下面是用于创建缩写的原则(来自编码大全)：**

1.使用标准缩写（列在字典中的常见缩写）  
2.去掉所有非前置元音(computer->cmptr，screen->scrn，apple->appl，integer->intgr)  
3.去掉虚词(and,or,the)  
4.使用第一个或前几个字母  
5.保留每个单词第一个和最后一个字母  
6.去除无用的后缀 --ing,ed等  
7.保留每个音节中最引人注意的发音  
8.基于发音来缩写(skating->sk8ing，before->b4,execute->xqt)  

<span id='iteratorMap'>
**4. 遍历Map尽量使用EntryMap**

keySet其实是遍历了2次，一次是转为Iterator对象，另一次是从hashMap中取出key所对应的value。而entrySet只是遍历了一次就把key和value都放到了entry中，效率更高。如果是JDK8，使用Map.foreach方法。当然如果你只取key的话请使用keyset
```java
public void foreachMap(){
	Map<String , String **out = new HashMap<String, String>();
	for (Map.Entry<String, String**empty : out.entrySet()) {
		String key = empty.getKey();
		String value = empty.getValue();
	}
}
```

<span id='return'>
**5. 在返回有遇到大于两个true返回**

在开发中难免会遇到这种情况，给3个布尔变量，当其中有2个或者2个以上为true才返回true，请使用下面这种写法  
最笨的写法：
```java
boolean atLeastTwo(boolean a, boolean b, boolean c) 
{
    if ((a && b) || (b && c) || (a && c)) 
    {
        return true;
    }
    else
    {
        return false;
    }
}
```
请使用下方写法：
```java
return (a==b) ? a : c;
OR
return a ^ b ? c : a;
```
<span id='chongfujisuan'>

**6. 尽量减少对变量的重复计算**

```java
for (int i = 0; i < list.size(); i++)
{...}
建议替换为
for (int i = 0, length = list.size(); i < length; i++)
{...}
```

<span id='objectref'>
**7. 循环内不要不断创建对象引用**

这种做法会导致内存中有n份Object对象引用存在，n很大的话，就耗费内存了，建议为改为
```java
for (int i = 1; i <= n; i++){
	Object obj = new Object();
}
应改为
Object obj = null;
for (int i = 0; i <= count; i++) { 
	obj = new Object(); 
}
```

<span id='strequals'>
**8. 字符串常量equals的时候写在前面**

```java
String str = "123";
if (str.equals("123")) {
	...
}
建议修改为：
String str = "123";
if ("123".equals(str)){
	...
}
```


<span id='count'>
**9. Mysql中count(*)与count(1)**

`count(*)`与`count(1)`并没有什么不同并不是`count(*)`会采用全表扫描通过通过

```java
EXPLAIN SELECT COUNT(*) FROM `user`;
EXPLAIN SELECT COUNT(1) FROM `user`;
EXPLAIN SELECT COUNT(ID) FROM `user`;
```

以上SQL执行结果如下
![img](/img/tips/count.png)  

<span id='sumexpr'>

**10. SQL中的 SUM(expr)**

```sql
SELECT SUM(ID>1000) FROM `user`;
SELECT COUNT(*) FROM user WHERE ID **'1000';
```

具体效率细节如下

![sqlsum-1](/img/tips/sqlsum-1.png)  

![sqlsum-1](/img/tips/sqlsum-2.png)

可见带where字句的扫描了更少的行，但在数据量较小时使用SUM(expr)更加便捷


---

— Neil 最后编辑与 2015.06