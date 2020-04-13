---
title: Git
date: '2018-10-17T11:44:00.000Z'
tags:
  - Git
  - 版本控制
comments: true
---

# Git

## 入门

### Git是什么？

Git中译为混账，是Linus先生花了一个星期写的分布式版本控制系统（VCS，Version Control System），用于Linux内核的协同开发。所谓版本控制系统，个人理解就是可以保存文本文件的历史版本信息，并且可以回溯到某个历史版本的文本文件管理系统。它的设计就是为了方便软件开发的版本迭代和协同开发。

比如，你打开电脑的记事本，一个不小心把昨天写的备忘全给删除了，而且还习惯性地按了Ctrl+S，你懵了。一般情况下是找不回来了。除非有版本备份，现在有些云笔记类软件就提供了这样的功能。其实质就是版本控制，它可以让你回溯到某个历史版本，像是吃了后悔药一般美妙。

### Git怎么玩？

#### 配置Git环境

玩什么都有第一步，已经司空见惯了呃。没错就是配置环境。Git的环境配置还不算复杂：

1. [下载Git](https://git-scm.com/download/win)并安装Git（注意：不懂英文的按默认配置就好）;
2. 向全世界宣称你的存在

   ```text
   # 注意：--global表示你电脑里的所以Git仓库都采用相同的配置
   # 如果有不用用户的Git仓库，就去掉吧。
   git config --global user.name "Your Name"
   git config --global user.email "email@example.com"
   ```

#### 建个仓库放点东西

1. 建库简单：在磁盘上新建个文件夹作为工作区，然后右击打开Git Bash，执行`git init`，你会发现多了个`.git`文件夹。
2. 放东西有点复杂
   * 第一步：在工作区新建一个文件，命名为yuren.txt。注意：文本文件的编码是UTF-8 without BOM（使用Notepad++可以查看和修改）
   * 第二步：`git add .`添加文件到暂存区
   * 第三步：`git commit -m "add yuren.txt"`添加文件到本地仓库

历尽千辛万苦终于把文件交给Git来管理了。

#### 时光机穿梭

假定你在上个步骤中放入的文件是：yuren.txt。里面的内容是（标记为v1）：

> \[微信红包\]恭喜发财，大吉大利！ 是不是又冲进来了，这么激动干嘛，发几个文字而已~

后来，你想了想，改成了（标记为v2）：

> \[微信红包\]愚人节Happy！ 是不是又冲进来了，这么激动干嘛，发几个文字而已~

根据【建个仓库放点东西】里说的步骤更新本地仓库中的信息：

```bash
git add .
git commit -m "update yuren.txt"
```

不知道哪根筋不对，你又想要v1版本的yuren.txt了，但是又忘了上一次写的内容，怎么办？

没关系，可以通过`git log`查看每次的提交信息。

```bash
$ git log
commit 1ba454f3ef3e48b88b4c24f72dc8055407cd9019
Author: Your Name <email@example.com>
Date:   Fri Apr 1 16:19:41 2016 +0800

    update yuren.txt

commit ce58ee1a57d21c9d752e80b820b7f2968249ac2e
Author: Your Name <email@example.com>
Date:   Fri Apr 1 16:17:35 2016 +0800

    add yuren.text
```

回退到上一次commit，只需要执行

```bash
$ git reset --hard HEAD^
HEAD is now at ce58ee1 add yuren.text
```

这里，HEAD^代表上一个版本，HEAD^^代表上上个版本，HEAD~100代表上100个版本。当然可以使用具体的commit id来回退，比如上面的等价于`$ git reset --hard ce58ee1`

重新打开yuren.txt来看，v1版本的信息果然回来了。

人的心理真是捉摸不透，刚找回v1版本的信息了，就又怀念v2版本的信息了。然而，现在使用`git log`都救不了你了。

```bash
$ git log
commit ce58ee1a57d21c9d752e80b820b7f2968249ac2e
Author: Lshare <linlshare@gmail.com>
Date:   Fri Apr 1 16:17:35 2016 +0800

    add yuren.text
```

v2版本的信息似乎丢失了。怎么办？

* 方法一：如果你还知道v2版本的commit id的话可以`$ git reset --hard <commit id>`来解决；
* 方法二：假如你不知道也没关系，使用`git reflog`查看你的每次命令，接下来就可以用方法一解决了。

  ```text
    $ git reflog
    ce58ee1 HEAD@{0}: reset: moving to HEAD^
    1ba454f HEAD@{1}: commit: update yuren.txt
    ce58ee1 HEAD@{2}: commit (initial): add yuren.text
  ```

## 进阶

### 本地记住密码

```bash
echo https://账号:密码@github.com >> ~/.git-credentials
# 存储到内存
git config --global credential.helper cache
# 存储到本地
git config --global credential.helper store
```

### 提交当前分支的更改到远程的同名分支

```text
git push origin HEAD
```

### 删除远程分支

```bash
git push origin :dev-sth
# 相当于
git push origin --delete dev-sth
```

### 重命名分支

```bash
# 本地分支重命名
git branch -m dev-old dev-new
# 删除远程分支
git push origin :dev-old
# 推送到远程
git push origin dev-new:dev-new
```

### 撤销 git commit 但未 git push 的修改

```bash
# 找到想要撤销的 commit id
git log
# 撤销，但不对代码修改进行撤销，还可以再次 commit
git reset <commit id> 
# 撤销并忽略该次 commit 的代码修改
git reset --hard <commit id>
```

### 移除远程分支上的脏文件（clean remote files）

```text
git rm -rf <file1> <file2>
```

### 添加子模块（submodule）

```text
git submodule add git://github.com/chneukirchen/rack.git rack
```

### 重命名已有的分支

If you want to rename a branch while pointed to any branch, do:

```text
git branch -m <oldname> <newname>
```

If you want to rename the current branch, you can do:

```text
git branch -m <newname>
```

### 递归拉取代码（包含子模块）

```bash
git clone --recursive [Git仓库地址]
```

## 问题

### Server does not allow request for unadvertised object

日志：

```text
Cloning into '/home/ec2-user/pack_workspace/dts-plugin-abot_default_jenkinsDefault/dts-plugin-abot/dts/common/build'...
Cloning into '/home/ec2-user/pack_workspace/dts-plugin-abot_default_jenkinsDefault/dts-plugin-abot/dts/common/protocol'...
error: Server does not allow request for unadvertised object 9cfd865e2e275dd6cf375efd734bc7d6b17da49f
Fetched in submodule path 'dts/dts-plugin', but it did not contain 9cfd865e2e275dd6cf375efd734bc7d6b17da49f. Direct fetching of that commit failed.
Failed to recurse into submodule path 'dts'
```

原因：

子模块分支游离，需要

```text
git pull
git checkout <your branch>
```

然后提交下即可。

记得确保解决完该问题后，合并到你的 master 分支上。

