title: Hexo文章置顶
date: 2016-07-09 17:08:38
tags:
---
博客里《读过的好的文章》记录我学习后整理的一些文章，希望在博客中置顶，方便自己查看。
在hexo github 的issue里找到了解决办法,[解决Hexo置顶问题](http://www.netcan666.com/2015/11/22/%E8%A7%A3%E5%86%B3Hexo%E7%BD%AE%E9%A1%B6%E9%97%AE%E9%A2%98/)，只需两步：
1. 用文章中的js代码替换node_modules/hexo-generator-index/lib/generator.js
2. 在需要置顶的文章的front-matter中添加top值，值越大越置顶。

``` java
title: 读过的好的文章
date: 2015-07-13 21:00:23
tags:
- 博客
- 收藏
categories: 收藏
top: 1000
```
