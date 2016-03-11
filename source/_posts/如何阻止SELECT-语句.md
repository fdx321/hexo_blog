title: '如何阻止SELECT * 语句'
date: 2015-10-31 19:12:43
tags:
- SQL
- 收藏
categories: 转载

---
###### 【本文转载自http://www.cnblogs.com/woodytu/p/4913166.html， 仅仅收藏做笔记用，如果侵权，请作者联系我删除】

我们每个人都知道是个不好的做法，但有时我们还是要这样做：我们执行SELECT * 语句。这个方法有很多弊端：

你从你的表里返回每个列，甚至后期加的列。想下如果你的查询里将来加上了VARCHAR(MAX)会发生什么……
对于指定的查询，你不能定义覆盖非聚集索引来克服执行计划里的查找（lookup）运算符，因为你会在额外的索引里重复你的数据……
现在的问题是你如何阻止SELECT *语句？当然你可以进行代码审核，你可以提供最佳模式指导，但谁最终会留意这些？基本上没有人——很遗憾这就就是令人伤心的事实……

但有一个非常简单方法来阻止SELECT *语句，在表里用技术层面来解决。<!--more-->
这个问题的解决方法非常简单：在你的表定义上增加一个产生除零错误的的计算列。这个方法超简单，但却真正有效。我们来看下面的表定义：

```sql
 1 -- Create a simple table with a computed column that generates
 2 -- a divide by zero exception.
 3 CREATE TABLE Foo
 4 (
 5     Col1 INT IDENTITY(1, 1) NOT NULL PRIMARY KEY,
 6     Col2 CHAR(100) NOT NULL,
 7     Col3 CHAR(100) NOT NULL,
 8     DevelopersPain AS (1 / 0)
 9 )
10 GO
11 
12 -- Insert some test data
13 INSERT INTO Foo VALUES ('a', 'a'), ('b', 'b'), ('c', 'c')
14 GO
```
如你所见，我这里增加了一个进行除零的计算列。这表示当是查询这个列时，你会得到一个错误信息——例如在SELECT * 语句里：
```sql
1 -- A SELECT * statement doesn't work anymore, ouch...
2 SELECT * FROM Foo
3 GO
```
![](http://images2015.cnblogs.com/blog/750348/201510/750348-20151027084005154-1492997900.png)

但另一方面如果你通过名称指定查询列，你不会反悔计算列，你的查询如愿正常执行：
```sql
1 -- This SQL statement works
2 SELECT Col1, Col2, Col3 FROM Foo
3 GO
```
![](http://images2015.cnblogs.com/blog/750348/201510/750348-20151027090459216-1260701824.png)

很不错吧，是不是？

小结
在各个交流会上我经常提到：有时我们只是变得太复杂了！这个用计算列的方法非常简单——肯定需要表架构修改。但下次设计新表的时候，要记得用这个方法。

感谢关注！

参考文章：
www.sqlpassion.at/archive/2015/10/26/how-to-prevent-select-statements/

注：此文章为WoodyTu学习MS SQL技术，收集整理相关文档撰写，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出此文链接！
