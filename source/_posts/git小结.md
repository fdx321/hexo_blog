title: git小结
date: 2016-06-13 09:30:20
tags: Git
---
之前也一直用git，不过都是自己玩，最近由于公司用git，所以又重新学习了一下。过了一遍[廖学峰的git教程](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/)和[Atlassian Git Tutorial](https://www.atlassian.com/git/tutorials/what-is-version-control/). 现对一些关键点做一些总结，方便自己查阅。

```python
# 在当前目录新建repository
$ git init
# 在指定目录新建repository
$ git init <dir>
```
```python
# 克隆到当前目录
git clone <repo的链接>
# 克隆到指定目录
git clone <repo的链接> <dir>
```
<!--more-->
```python
# 配置用户名，邮件，不加global则只配置当前repo
$ git config --global user.name <name>
$ git config --global user.email <email>
```
```python
# 这个命令可以修改上一次的commit message
$ git commit --amend
# 查看指定条数的log
$ git log -n <limit>
```
```python
# 切换分支
$ git checkout <brach name>
# 切换到指定历史版本
$ git checkout <commit>
# 切换指定文件到指定历史版本
$ git checkout <commit> <file>
```
##### git revert VS git reset
最大的区别就是revert会保留原来的commit记录并在后面新建一条commit记录
reset则直接恢复到之前的版本并删除commit记录。
```python
$ git revert <commit>
$ git reset <commit>
```
![image1](/images/git小结_1.png)

##### 分支操作
```python
# 列出所有分支名
$ git branch
# 新建分支
$ git branch <branch>
# 安全删除分支，如果有unmerged changes则不删除
$ git branch -d <branch>
# 暴力删除分支
$ git branch -D <branch>
# 重命名当前分支
$ git branch -m <branch>
```
##### git rebase VS git merge
```python
# rebase当前分支到指定分支
$ git rebase <base>
# 合并指定分支到当前分支
$ git merge <branch>
```
![image1](/images/git小结_2.png)
![image1](/images/git小结_3.png)

##### git fetch VS git pull
最大的区别就是git fetch不会自动合并

##### 工作流
[comparing-workflows](https://www.atlassian.com/git/tutorials/comparing-workflows)这篇文章讲的很清楚，网上也有很多中文翻译。
* Centralized Workflow
![image1](/images/git小结_4.png)
* Feature Branch Workflow
![image1](/images/git小结_5.png)
* Gitflow Workflow
![image1](/images/git小结_6.png)
* Forking Workflow
![image1](/images/git小结_7.png)
<style>
img[alt="image1"] {
  width:300px;
  display: block;
}
</style>