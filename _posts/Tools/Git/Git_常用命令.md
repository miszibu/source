---
title: Git_常用命令
date: 2018-04-21 10:41:27
tags: [Git]

---



本文记录日常工作中常用的GIt命令，做一个备忘录

<!--more-->

### 0.创建公钥密钥对

```shel
ssh-keygen -t rsa -C "varzibu@163.com" -f ~/.ssh/id_rsa
// 生成一个公钥密钥对
// 将本机的公钥部署到所需要的git服务器上
ssh-add -l // 查看私钥列表
ssh-agent bash //若Could not open a connection to your authentication agent.则输入
ssh-add ~/.ssh/id_rsa // 将私钥加到git中
```

### 1.新建代码库

```bash
git init //在当前目录新建一个Git代码库 
git init [project-name] //新建一个目录，将其初始化为Git代码库
git clone [url]  //下载一个项目和它的整个代码历史
```

### 2.配置仓库信息

````bash
git config --list //显示当前的Git配置
git config -e --global //全局编辑Git配置文件
````

### 3.增加/删除文件

```bash
git add [file1] [file2] //添加文件到暂存区
git add [dir]		   //添加目录到暂存区
git add .			   //添加当前目录下所有文件到暂存区

git rm [file1] [file2]  //删除工作区文件
git rm --cached [file]  //停止追踪指定文件
git rm -f [file]  	    //强制删除工作区文件

git mv [file-origin] [file-rename] //改名文件，并且将这个改名放入暂存区 
```

### 4.代码提交

```bash
git commit -m [message] //将暂存区提交到仓库区
git commit [file1] [file2] -m [message] //提交暂存区的指定文件到仓库区
git commit -a //提交本地所有暂存
git commit -v //提交时显示所有的diff信息
```

### 5.分支

```bash
git branch //列出所有本地分支
git branch -r //列出所有远程分支
git branch -a //列出所有本地分支和远程分支
git branch -vv //查看branch详情
git branch [branchName] //新建本地分支
git checkout [branchName] //切换分支
git checkout -b [branchname] //新建分支 并切换到该分支
git branch -d [branchName] //删除分支 -d只能删除已合并的分支 -D可以强制删除分支
git merge [branchName] //合并分支到当前分支
git push origin [branchName] //创建远程分支 既push本地分支
git push origin :[branchName] //删除远端分支
git push origin --delete [branchName]//删除远端分支
git branch --set-upstream-to=origin/<branchName> localBranchName //将本地分支与远端分支挂钩 
git checkout -b [localBranchName] [remoteBranchName] // 创建本地分支并绑定远端分支
git remote show origin  // 查看远端分支和本地分支绑定情况
git remote prune origin // 清除远端分支中无用的分支
```

### 6.标签

```bash
git tag -d [tag]　　　　       　　　　//本地删除tag
git push origin :refs/tags/vx.x.x　 //本地tag删除了，再执行该句，删除线上tag
git tag　　				   //查看tag
git tag [tag] c809ddbf83939a89659e51dc2a5fe183af384233　//在某个commit 上打tag
git tag [tag]				 //在当前commit版本 打tag
git push origin [tag]		  //本地tag推送到线上	
```

### 7.查看信息

```bash
git status //显示有变更的文件
git log    //显示当前分支的版本历史
```

### 8.远程同步

```bash 
git fetch [remote]  //下载远程仓库的所有变动
git remote -v       //显示所有远程仓库
git remote # 显示某个远程仓库的信息
$ git remote show [remote]

# 增加一个新的远程仓库，并命名
$ git remote add [shortname] [url]

# 取回远程仓库的变化，并与本地分支合并
$ git pull [remote] [branch]

# 上传本地指定分支到远程仓库
$ git push [remote] [branch]

# 强行推送当前分支到远程仓库，即使有冲突
$ git push [remote] --force

# 推送所有分支到远程仓库
$ git push [remote] --all
```

### 9.撤销

```bash
# 恢复暂存区的指定文件到工作区
$ git checkout [file]

# 恢复某个commit的指定文件到暂存区和工作区
$ git checkout [commit] [file]

# 恢复暂存区的所有文件到工作区
$ git checkout .

# 重置暂存区的指定文件，与上一次commit保持一致，但工作区不变
$ git reset [file]

# 重置暂存区与工作区，与上一次commit保持一致
$ git reset --hard

# 重置当前分支的指针为指定commit，同时重置暂存区，但工作区不变
$ git reset [commit]

# 重置当前分支的HEAD为指定commit，同时重置暂存区和工作区，与指定commit一致
$ git reset --hard [commit]

# 重置当前HEAD为指定commit，但保持暂存区和工作区不变
$ git reset --keep [commit]

# 新建一个commit，用来撤销指定commit
# 后者的所有变化都将被前者抵消，并且应用到当前分支
$ git revert [commit]

# 暂时将未提交的变化移除，稍后再移入
$ git stash
$ git stash pop
```

### 10.放弃本地修改

~~~Shell
git fetch origin 			
git reset --hard origin/master
#获取远端分支 然后将本地分支重置到远端版本

git checkout . #本地所有修改的。没有的提交的，都返回到原来的状态
git stash #把所有没有提交的修改暂存到stash里面。可用git stash pop回复。
git reset --hard HASH #返回到某个节点，不保留修改。
git reset --soft HASH #返回到某个节点。保留修改

git clean -df #返回到某个节点
git clean 参数
    -n 显示 将要 删除的 文件 和  目录
    -f 删除 文件
    -df 删除 文件 和 目录
~~~

### 其他

```shell
# 使下一次输入的帐密 被记录 
git config --global credential.helper store

```



### 修改历史commit message

```sh
# 直接修改上一条commit的message
git commit --amend

# 修改历史 commit message
git log # 查看历史提交情况
git rebase -i HEAD~3 #最后数字为距当前提交最远处提交的举例，比如想要修改的提交在10次前，该参数就可以设置为比10大
pick 56b2308 feat(pages): home DONE
pick 82f65eb fix(pages movie): slides bug fixed
edit 08b2087 feat(pages home & movie): add FABs animation # 找到想要修改的那条记录，将pick改为edit

# 若想要修改多条提交，则需要循环执行以下两条命令，以迭代所有提交信息
# 修改commit message
git commit --amend
# rebase 到修改后的 commit point,若无需要更新的 commit,则 rebase 到最新的 commit point
git rebase --continue
```



### 相关引用

[[GIT 常用命令](http://www.cnblogs.com/chenwolong/p/GIT.html)](https://www.cnblogs.com/chenwolong/p/GIT.html)