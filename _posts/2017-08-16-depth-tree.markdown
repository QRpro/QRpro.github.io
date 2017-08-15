---
layout:     post
title:      "数据库设计-深度树"
subtitle:   "经验之谈-无限级分类数据表设计方案"
date:       2017-08-16 
header-img: "img/depth-tree-bg.jpg"
catalog:    true
author:     "Neil"
tags:
    - 经验之谈
    - 数据库
---  


## 写在前面的话  
　　最新在开发新的会员2.0系统，所以在研究许多新的知识比如ActiveMQ，Dubbo，Zookeeper，MyCat这些东西，笔记有一直在整理，但是由于时间原因没有调整格式弄成MD格式，等后期空闲时间慢慢在整理复习一遍放上来。
##   问题
　　今天要说的东西是我在处理一个查询会员推荐等级关系的需求的时候，发现查询结果异常缓慢，10W条数据居然用了2s左右，由于SQL语句比较复杂并且还需要除了关联自己外还需要关联其他表查询，首先我用 EXPLAIN 分析了一番，发现存在如下问题，关联表中的USER_ID 缺少主键，导致全表扫描了9W+条数据，后来我在该字段加了条索引，发现准确的匹配索引只扫描了一条数据，优化后运行结果大概在0.2s+，处理完索引的问题再来处理数据结构的问题，发现需要查询下级时需要多次自关联查询sql语句比较复杂，研究了一下也看了一些文章。发现目前主流的数据库设计处理这种递归关系数据多采用以下四种方式
　　以下的所有例子都是以这个结构作为测试，只介绍简单的查询子节点SQL 其他SQL也不复杂可以自行摸索
![img](/img/blogarticles/depth-tree/tree.png)
####  邻接表
比如说文章的评论，我们公司也是采用这种形式，无限级分类也大多采用此类方式
![img](/img/blogarticles/depth-tree/Adjacency.png)
我们来分析以下缺点吧  就单单查询一个节点的所有后代(求子树)怎么查呢？
比如说找D的下两级  

```
SELECT SUB_ID FROM tb1 WHERE P_ID ='D'
UNION
SELECT SUB_ID FROM tb1 WHERE P_ID IN (SELECT SUB_ID FROM tb1 WHERE P_ID ='D')
```  
每需要查多一层，就需要联结多一次表，当然你也可以选择用MySQL递归CTE(公共表表达式)，当然这个东西是MySQL8.0版本才新加入的，如果你愿意为了一个小需求换个数据库版本的话….
再来分析下优点：首先添加记录是非常方便的只需要提供父节点的ID就OK，移动删除也非常简单。
####  路径枚举 
　　路径枚举的设计是指通过将所有祖先的信息联合成一个字符串，并保存为每个节点的一个属性。路径枚举是一个由连续的直接层级关系组成的完整路径。如"/home/account/login",其中home是account的直接父亲，这也就意味着home是login的祖先。大概就是这个样子
![img](/img/blogarticles/depth-tree/path.png)  

假设我们需要查询某个节点的后代，SQL语句可以这样写(假设查询D的后代)  

```
SELECT * FROM tb1 
WHERE Path LIKE 'A/D/%'
```
　　插入也比较方便复制一份要插入节点的逻辑上的父亲节点路径，并将这个新节点的Id追加到路径末尾就可以了  
　　再来看看它的缺点：
- 要依赖高级程序来维护路径中的字符串，并且验证字符串的正确性的开销很大。
- VARCHAR的长度很难确定。无论VARCHAR的长度设为多大，都存在不能够无限扩展的情况。

####  嵌套集  
　　通过集合的包含关系，嵌套结合模型可以表示分层结构，每一个分层可以用一个Set来表示（一个圈），父节点所在的圈包含所有子节点所在的圈。我知道这个比较不好理解，其实就是这个样子的一个东西
![img](/img/blogarticles/depth-tree/Nested.png)  
![img](/img/blogarticles/depth-tree/Nested-table.png)  
> 查询方法(查询指定节点及子节点)  

```
SELECT node.sub_id
FROM nested_category AS node,
        nested_category AS parent
WHERE node.lft BETWEEN parent.lft AND parent.rgt
        AND parent.sub_id = 'B'
```  
优点自然不必多说查询子树非常方便，而且删除节点也比较方便，不用考虑子节点问题。缺点嘛，比如现在有一个需求查询C的直接父节点…

#### 闭包 
额外创建了一张表(空间换取时间)，它包含两列，每一列都是主表中的id的外键，也就是说用另一张表存储了所有祖先-后代的关系的记录大概就是这个样子，第二张表罗列祖先和全部后代。
![img](/img/blogarticles/depth-tree/Closure.png) 
这种方式查询删除都比较简单，但是占用空间大，操作不直观。

#### 总结 

> 总结一下上述四种方式：
- 邻接表：记录父节点。优点是简单，缺点是访问子树需要遍历，发出许多条SQL，对数据库压力大。
- 路径枚举：用一个字符串记录整个路径。优点是查询方便，缺点是插入新记录时要手工更改此节点以下所有路径，很容易出错。
- 闭包表：专门一张表维护Path，缺点是占用空间大，操作不直观。
- 嵌套集：记录左值和右值，缺点是复杂难操作。

## 深度树结构
　　那么我们说下最终选定的方案：采用前序遍历方式进行存储，这样方便我们取数, 左边的树结构，映射在数据库里的结构见右图表格，注意每个表格的最后一行必须有一个END标记，level设为0，group是用来分组的，有N个顶级节点就有N颗树N个group
![img](/img/blogarticles/depth-tree/depth-tree.png)  

> 获取D节点及其所有子节点：  

```
select * from tb2 where groupID=1 and 
  line>=7 and line< (select min(line) from tb2 where groupid=1 and line>7 and level<=2)
删除和获取相似，只要将sql中select * 换成delete即可。
```

> 查L节点的上一级父节点：

```
select * from tb2 where groupID=1 
  and line=(select max(line) from tb2 where groupID=1 and line<11 and level=3) 
```

> 插入新节点：例如在J和K之间插入一个新节点T：

```
update tb2 set line=line+1 where  groupID=1 and line>=10;
insert into tb (groupid,line,id,level) values (1,10,'T',4);
```  

总结
> **优点**  

1.  是无限深度树
2. 虽然不具有所见即所得的效果，但是依然具有直观易懂，方便调试
3. 能充分利用SQL，查询、删除、插入非常方便
4. 只需要一张表。
5. 兼容所有数据库。
6. 占用空间小  

> **缺点**  

1. 节点移动就比较麻烦，不过恰好会员推荐关系不会移动节点。    


方案选定了，重头戏来了。怎样把现有邻接表中的数据初始化到这张表中?  

> **方案1**  

先从上往下找查询A节点下第一个子节点用Limit来控制并递归查询，当最后一个的子节点为空时从下往上找父节点并查询父节点的第二个子节点，再次递归查询第二个子节点的第一个子节点没有的话继续找父节点第三个子节点。有点不好理解。这个方案存在一个问题，每次取一条，取得是第几条很难把控，存储过程写完了发现这个问题挺头疼，所以想了想别的方案，因为有bug这里我就不粘上来了。   
> **方案2**  

每个节点都视为一个新节点采用插入的方式，比如先插入B节点然后再插入C节点，先找到C节点的父节点也就是A将line>1的全部加一然后把它插到最左边。插完之后大概是这个样子的
![img](/img/blogarticles/depth-tree/tree2.png)  
首先需要两张表tb1和tb2  

```
CREATE TABLE `tb1` (
  `ID` varchar(30) NOT NULL,
  `P_ID` varchar(30) DEFAULT NULL,
  `LEVEL` int(11) DEFAULT NULL,
  PRIMARY KEY (`ID`)
) ENGINE=InnoDB DEFAULT CHARSET=utf-8;

INSERT INTO `tb1` VALUES ('A', 'BEGIN', '1');
INSERT INTO `tb1` VALUES ('B', 'A', '2');
INSERT INTO `tb1` VALUES ('C', 'A', '3');
INSERT INTO `tb1` VALUES ('D', 'A', '3');
INSERT INTO `tb1` VALUES ('E', 'B', '2');
INSERT INTO `tb1` VALUES ('F', 'B', '3');
INSERT INTO `tb1` VALUES ('G', 'C', '2');
INSERT INTO `tb1` VALUES ('H', 'D', '3');
INSERT INTO `tb1` VALUES ('I', 'D', '4');
INSERT INTO `tb1` VALUES ('J', 'H', '4');
INSERT INTO `tb1` VALUES ('K', 'H', '4');
INSERT INTO `tb1` VALUES ('L', 'H', '3');
```  

```
CREATE TABLE `tb2` (
  `groupid` int(11) NOT NULL,
  `line` int(11) DEFAULT NULL,
  `sub_id` varchar(30) DEFAULT NULL,
  `level` int(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf-8;
```

主方法用于查找树跟节点、添加根节点以及添加结束节点 

```
BEGIN
DECLARE done_flag INT DEFAULT FALSE;		
DECLARE _root VARCHAR(10);
DECLARE count INT DEFAULT 0;
DECLARE max_line INT DEFAULT 0;

DECLARE cur_user CURSOR
FOR
	-- 查询所有根节点
	SELECT ID FROM tb1 WHERE P_ID NOT IN (SELECT ID FROM tb1) GROUP BY P_ID;
DECLARE CONTINUE HANDLER FOR NOT FOUND SET done_flag=TRUE;
	
OPEN cur_user;
		read_loop: LOOP
			FETCH cur_user INTO _root;
			IF done_flag THEN
				LEAVE read_loop;
			END IF;
		-- 树+1
			SET count = count + 1;
		-- 插入根节点
			INSERT INTO tb2(groupid,line,ID,level) VALUES (count,max_line + 1, _root,1);
			CALL test(count,_root);
			SELECT MAX(line) FROM tb2 INTO max_line;
		-- 插入END 节点
			SET max_line = max_line + 1;
			INSERT INTO tb2(groupid,line,ID,level) VALUES (count,max_line,'END',0);
		END LOOP;
	CLOSE cur_user;
END
``` 

再看main调用的方法test   

```
BEGIN

DECLARE done_user INT DEFAULT FALSE;
DECLARE t_subid VARCHAR(10);
DECLARE t_pid VARCHAR(10);
DECLARE t_level INT DEFAULT 0;
DECLARE t_line INT DEFAULT 0;
DECLARE count INT DEFAULT 0;

-- 定义游标取出每一级所有用户
DECLARE cur_user CURSOR
FOR
	SELECT t.ID FROM tb1 t WHERE p_id=_root;
DECLARE CONTINUE HANDLER FOR NOT FOUND SET done_user=TRUE;
-- 设置递归深度
SET @@SESSION.max_sp_recursion_depth=255;

OPEN cur_user;
		read_loop_user: LOOP
			FETCH cur_user INTO t_subid;
			IF done_user THEN
				LEAVE read_loop_user;
		  END IF;	
		-- 查询当前用户父节点
		SELECT p_id FROM tb1 WHERE ID = t_subid INTO t_pid;
		-- 获取父节点level及line
		SELECT `line`,`level` FROM tb2 WHERE sub_id = t_pid INTO t_line,t_level;
		-- 父节点后所有节点后移
		update tb2 set line=line+1 where groupid = _group and line > t_line;
		-- 插入当前节点
		insert into tb2 (groupid,line,sub_id,level) values (_group,t_line+1,t_subid,t_level+1);
		-- 递归调用
		CALL test(_group,t_subid);
		END LOOP;
	CLOSE cur_user;
END
```  

## 后记   
嗯，这个程序10W条数据在测试服上跑了12个小时，执行效率堪忧，不过这也是最简单的一种方法，其他的方案还在思考….

----
-- Neil 最后编辑于 2017.08