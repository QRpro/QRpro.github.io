---
layout:     post
title:      " 如何更优雅的撸码?! "
subtitle:   "持续更新ing"
date:       2017-06-25 
header-img: "img/post-bg-tips.jpg"
author:     "Neil"
tags:
    - Tips
---  



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

<p id='empty'>  

* 在返回Collection中有可能为空时请返回
{% highlight java %}
Collections.emptyList();
Collections.emptyMap();
Collections.emptySet();
{% endhighlight %}

<p id='listTorow'>
* 在查询中如果多个值分开获取后进行统计请尝试使用行转列思维
{% highlight sql %}
SELECT Date,
MAX(CASE 列名 WHEN 列值 THEN XX ELSE 0 END ) 别名,
MAX(CASE 列名 WHEN 列值 THEN XX ELSE 0 END ) 别名 
FROM TabName  
GROUP BY Date
{% endhighlight %}


<p id='abbreviation'>
* 下面是用于创建缩写的原则(来自编码大全)：

<br>1. 使用标准缩写（列在字典中的常见缩写）  
<br>2. 去掉所有非前置元音(computer->cmptr，screen->scrn，apple->appl，integer->intgr)  
<br>3. 去掉虚词(and,or,the)  
<br>4. 使用第一个或前几个字母  
<br>5. 保留每个单词第一个和最后一个字母  
<br>6. 去除无用的后缀--ing,ed等  
<br>7. 保留每个音节中最引人注意的发音  
<br>8. 基于发音来缩写(skating->sk8ing，before->b4,execute->xqt)  

<p id='iteratorMap'>
* 遍历Map尽量使用EntryMap
<br>keySet其实是遍历了2次，一次是转为Iterator对象，另一次是从hashMap中取出key所对应的value。而entrySet只是遍历了一次就把key和value都放到了entry中，效率更高。如果是JDK8，使用Map.foreach方法。当然如果你只取key的话请使用keyset
{% highlight java %}
public void foreachMap(){
	Map<String , String > out = new HashMap<String, String>();
	for (Map.Entry<String, String> empty : out.entrySet()) {
		String key = empty.getKey();
		String value = empty.getValue();
	}
}
{% endhighlight %}

<p id='return'>
* 在返回有遇到大于两个true返回
在开发中难免会遇到这种情况，给3个布尔变量，当其中有2个或者2个以上为true才返回true，请使用下面这种写法

最笨的写法：
{% highlight java %}
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
{% endhighlight %}
请使用下方写法：
{% highlight java %}
return (a==b) ? a : c;
OR
return a ^ b ? c : a;
{% endhighlight %}
<p id='chongfujisuan'>

*  尽量减少对变量的重复计算
{% highlight java %}
for (int i = 0; i < list.size(); i++)
{...}
建议替换为
for (int i = 0, length = list.size(); i < length; i++)
{...}
{% endhighlight %}

<p id='objectref'>
* 循环内不要不断创建对象引用
这种做法会导致内存中有n份Object对象引用存在，n很大的话，就耗费内存了，建议为改为
{% highlight java %}
for (int i = 1; i <= n; i++){
	Object obj = new Object();
}
应改为
Object obj = null;
for (int i = 0; i <= count; i++) { 
	obj = new Object(); 
}
{% endhighlight %}

<p id='strequals'>
* 字符串常量equals的时候写在前面
{% highlight java %}
String str = "123";
if (str.equals("123")) {
	...
}
建议修改为：
String str = "123";
if ("123".equals(str)){
	...
}
{% endhighlight %}


<p id='count'>
* Mysql中count(*)与count(1)
count( * )与count(1)并没有什么不同并不是count( * )会采用全表扫描通过通过

{% highlight sql %}
EXPLAIN SELECT COUNT(*) FROM `user`;
EXPLAIN SELECT COUNT(1) FROM `user`;
EXPLAIN SELECT COUNT(ID) FROM `user`;
{% endhighlight %}

以上SQL执行结果如下
![count( * )](img/tips/count.png)  

<p id='sumexpr'>
 SQL中的SUM(expr)

{% highlight sql %}
SELECT SUM(ID>1000) FROM `user`;
SELECT COUNT(*) FROM user WHERE ID > '1000';
{% endhighlight %}

具体效率细节如下

![sqlsum-1](/img/tips/sqlsum-1.png)  

![sqlsum-1](/img/tips/sqlsum-2.png)

可见带where字句的扫描了更少的行，但在数据量较小时使用SUM(expr)更加方便<br>


---

—— Neil 2015.06