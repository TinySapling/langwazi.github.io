---
title: git与我
date: 2017-11-01 02:32:22
tags:
  - git
  - 工具
---

![搞事情](http://upload-images.jianshu.io/upload_images/8403018-50feb50d3914f8c7.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*使用git日常*
参考自：[git教程](http://www.yiibai.com/git/)

 <!-- more -->

git常用命令
--
1. 初始化工程
  `git init`
  在本地已存在的工程目录下打开git base，输入`git init`会在该目录下创建.git隐藏目录（相当于本地仓库）。
   `git clone 远程仓库地址 `
   该命令是将远程仓库克隆到本地，即直接将远程仓库上的内容下载到本地电脑。

2. 添加与提交
   `git add -A`
    `git add -u`
    `git add .`
    `git add fileName`
   这一组命令是添加文件到暂缓区，暂缓区的文件可以被commit,最后被push到远程仓库。`git add -A`添加所有变化；`git add -u`添加被修改(modified)和被删除(deleted)文件，不包括新文件(new)；`git add .`添加新文件(new)和被修改(modified)文件，不包括被删除(deleted)文件；`git add fileName`添加制定文件到暂缓区。
    `git commit `
    `git commit -m 标题`
   提交修改到本地仓库.

3. 添加远程仓库

   如果是通过 `git init`初始化的git仓库(clone方式跳过)，那么在提交到远程仓库前需要绑定远程仓库,步骤如下:
     在远程创建一个没有任何文件的空仓库(不要read等默认文件),**要求空仓库是为了避免在在后面push到远程仓库时报两边状态不一致的错误,否则还要在push前处理两端同步的问题.**
     输入命令`git remote add origin 远程仓库地址`添加远程仓库,原始命令为`git remote add 仓库名 远程仓库地址`,origin表示原始的,一般第一个仓库默认为origin.

4. pull与push
  通过`git push <仓库名> <本地分支名>  <远程分支名>`将本地仓库中已经commit的代码推到远程仓库中,没有做修改的情况下仓库为origin,本地分支为master,即`git push origin master`,省略远程分支名则默认与本地分支名相同,如果该远程仓库不存在会自动创建    [更多push命令](http://www.cnblogs.com/qianqiannian/p/6008140.html).
  通常在push前需要执行`git pull <远程主机名> <远程分支名>:<本地分支名>`例如origin主机的next分支与本地的master分支合并命令如下`git pull origin next:master`若省略本地分支名表示直接入当前本地分支合并,`git pull origin next`.pull后需要解决冲突，然后在commit，最后再一次 push.

5. 其他命令
  `git branch`查看分支
  `git branch dev2`新建分支dev2
  `git branch -d dev2`删除分支dev2
  `git checkout dev2`切换到制定分支 dev2
  `git merge 分支A`将分支A合并到当前分支
  `git status list`将改动缓存到临时区域，并还原到上次commit后的状态，此时你可以直接切换分支
  `git status pop`将缓存区域的改动重新应用到该分支并且移除临时缓存


一个本地仓库同时添加多个远程仓库
---
```
#添加远程1
git remote add <远程主机名1> <仓库地址>
#添加远程2
git remote add <远程主机名2> <仓库地址>
#提交到远程1分支
git push <远程主机名1> <本地分支名> <远程分支名>
#提交到远程2分支
git push <远程主机名2> <本地分支名> <远程分支名>
#pull同理
git pull <远程主机名> <远程分支名>:<本地分支名>
```
 github（其他仓库）账号切换后提交的账户未改变
 ---
场景：
本地studio中已经登录了github账号A，现在我切换到账号B创建了一个全新的仓库并正常操作完成push代码后，github显示的提交用户为账号A。

原因：
在当前git配置中存在一个user.name 和user.email在push操作时会自动嵌入到当次操作。
通过Git Bash命令`git config --list`查看当前的git相关配置。此处修改的值为
user.name 和user.email

解决方法大致有以下三种：
- 修改全局 （所有project都应用该配置）name和email，通过以下命令：
```
git config  --global user.name 你的目标用户名；
git config  --global user.email 你的目标邮箱名;
```
- 修改当前project的name和email，在当前project目录下通过命令：
```
git config user.name 你的目标用户名;
git config user.email 你的目标邮箱名;
```
- 修改位于在project下.git目录下的config添加如下节点

![增加配置](http://upload-images.jianshu.io/upload_images/8403018-7fce257cae6b4de4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)