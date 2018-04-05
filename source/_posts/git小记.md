---
title: git小记
comments: true
date: 2017-10-28 22:31:19
categories:
- others
tags:
- git
---



Git是什么？

Git是目前世界上最先进的分布式版本控制系统（没有之一）。

Git有什么特点？简单来说就是：高端大气上档次！ -- 廖雪峰

git大法好， git大法妙

git大法呱呱叫

### git之ssh方式连接

- [帮助](http://git.oschina.net/oschina/git-osc/wikis/%E5%B8%AE%E5%8A%A9)

### git之https方式记住用户名和密码的方法

- [帮助](http://git.oschina.net/oschina/git-osc/issues/2586)

### 强制更新，覆盖本地修改

```.java
git fetch --all
git reset --hard origin/master
git pull
```

git fetch 只是下载远程的库的内容，不做任何的合并 git reset 把HEAD指向刚刚下载的最新的版本

### 解决git status 显示中文乱码

- 找到.gitconfig文件，添加:

```.java  
 [core]
	  quotepath = false
```

### 使用源仓库，更新自己仓库内容

```.java
#查看远程主机
git remote -v
#添加远程仓库主机（fork的源仓库,取别名upstream）
git remote add upstream git@git....com
#将源仓库内容取回本地
git fetch upstream master
#与本地仓库合并
git rebase upstream/master 或 git merge upstream/master
#推送到自己的远程仓库
git push -f origin master
```



## reset --hard找回

有时候你可能不小心降commit之后的文件reset --hard丢失了，我就这么干过。不过庆幸的是，这个也是可以找回的。那就是使用git reflog;

```sh
$ git reflog
bfb3371 HEAD@{0}: checkout: moving from hexo to master
bfb3371 HEAD@{1}: checkout: moving from master to hexo
bfb3371 HEAD@{2}: reset: moving to HEAD^
ca77d24 HEAD@{3}: checkout: moving from hexo to master
bfb3371 HEAD@{4}: checkout: moving from master to hexo
ca77d24 HEAD@{5}: commit: upgrade content
bfb3371 HEAD@{6}: checkout: moving from hexo to master
bfb3371 HEAD@{7}: checkout: moving from master to hexo
bfb3371 HEAD@{8}: commit (initial): 添加hexo文件
```

找到你误reset的commit id, 然后再次执行reset

```sh
$ git reset --hard ca77d24
```

bingo, 丢失的文件又回来啦。



> [权威官方文档](https://git-scm.com/book/zh/v2)

> [七个你无法忽视的git使用技巧](http://www.oschina.net/news/68437/seven-git-hacks-you-just-cannot-ignore)