---
title: git命令之-rebase
categories:
  - Git
date: 2018-09-18 23:36:36
updated: 2018-09-18 23:36:36
tags: 
  - Git
---
很遗憾，用了这么久的 git ，对于其分支模型实在是没有仔细了解的。因为公司用的是 svn，我只是自己在本地用 git 进行了代码管理。比较无奈的是，公司的 svn 版本库结构很坑，无法用 git-svn 来实现比较友善的管理，只徒呼奈何了。

<!--more-->
有一个疑问就是，我能否在一个叫做 dev 分支上进行开发，然后把功能完善后，合并到 master 分支。

更实际一点，为了避免需要解决很复杂的代码冲突问题。我在主分支 master 上同步 svn 上的代码，每次想要合并把 dev 代码合并过去的时候，都先用 svn 把别人更新的代码更新下来，再用 merge 命令进行合并。我想用实例来操作是最好的。

# Rebase 命令

基本命令：

```
       git rebase [-i | --interactive] [options] [--exec <cmd>] [--onto <newbase>]
               [<upstream> [<branch>]]
       git rebase [-i | --interactive] [options] [--exec <cmd>] [--onto <newbase>]
               --root [<branch>]
       git rebase --continue | --skip | --abort | --quit | --edit-todo | --show-current-patch
```

这里，我们来解释一下两个名字：

- **upstream** 上游。这指的是我们想要将其变更拉取过来的分支。
- **branch** 分支。表示的是我们想要将变更应用到的分支。


如果指定了 *<branch>* 。`git rebase` 命令会首先做一个 `git checkout <branch>`（切换到我们指定的分支）。否则的话就停留当前分支上进行动作。

如果 *<upstream>* 没有指定（表示需要拉取其变更的分支），那么在 *branch.<name>.remote* 和 *branch.<name>merge* 选项中配置的 *upstream* 会被使用，如果 *--fork--point* 选项被假设。 如果当前不在任何分支上或当前分支没有配置一个 *upstream*，那么 rebase 会失败。

所有当前分支中的提交造成的变更，且不在 *upstream* 中的变更，会被存储到一个临时的区域。

当前分支会被重置到 *upstream*，若指定了 *--onto* 选项，则会重置到 *newbase* 。这和 `git reset --hard <upstream>`有相同的影响。*ORGI_HEAD* 被设置为指向重置前的分支顶。

那么，之前存储在临时区域的提交就会在当前分支上按序重放。

## 例子
如果我们有以下分支：

```
                     A---B---C topic
                    /
               D---E---F---G master
```

执行下面两个命令都会有同样的结果：

```
           git rebase master
           git rebase master topic
```

```
                             A'--B'--C' topic
                            /
               D---E---F---G master

```

事实上  `git rebase master topic` 与  

```
git checkout topic
git rebase master
```

相同。

## 重复变更内容的处理

如果上游分支中存在了我们当前分支中已经有了的变更，那么当前分支上的这个变更会被忽略。就如下面的例子( A， A' 代表了两个相同的变更，但只是 commit 信息不一样。)：

```
                     A---B---C topic
                    /
               D---E---A'---F master
```

如果我们当前位于 topic 分支，我们执行 

```
git rebase master
```

的结果将会是下面这样的：

```
                              B'---C' topic
                             /
               D---E---A'---F master

```

## 基于多分支迁移
我们来看一下，基于一个分支与另一个分支的差异来迁移变更到 topic 分支。

这里，我们假设我们的 topic 分支是从 next 分支衍生的。

```
               o---o---o---o---o  master
                    \
                     o---o---o---o---o  next
                                      \
                                       o---o---o  topic
```

现在我们想让 topic 分支变成是基于 master 衍生的。我们可以执行下面这个命令：

```
git rebase --onto master next topic
```

最后的结果就是：

```

               o---o---o---o---o  master
                   |            \
                   |             o'--o'--o'  topic
                    \
                     o---o---o---o---o  next

```

上面这个操作，也就真实的展示了一个比较让人难以理解的概念 **变基**。

**变基**：改变一个分支的基准位置(commit)。

## 对变基的直观解释。
# 实例

我们先初始化一个库。

```
git init temp 
cd test
echo README >> README.md
git add .
git commit -m 'add readme file'

echo file1 >> file1
git add .
git commit -m 'add file1'

```

再另外一个分支更新文件：

```
git co -b dev
echo file2 >> file2
git add .
git commit -m 'add file2'
```

切换回主分支，再添加个文件：

```
git co master
echo file3 >> file3
git add .
git commit -m 'add file3'
```

现在我打算把 dev 分支上的改动合并到 master 分支。一般来说，我们可以采用 merge 命令。


如果 *upstream branch* 已经包含了一个你已经做了的改变，那么这个变化会被跳过。在下面的操作中（ A' 与 A 做了相同的变化，但是 commit 信息不一样）

```
                     A---B---C topic
                    /
               D---E---A'---F master
```

其结果是：

```
                              B'---C' topic
                             /
               D---E---A'---F master
```

## --onto

下面来看一下怎么样将一个 topic 分支移植到
# merge

```
git co master
git merge dev

git log --pretty=oneline 

b67203a314ab47dde68016f4a2fc04b6ee056e28 (HEAD -> master) Merge branch 'dev'
1d38148fa8a973e41f41e02b9554b06137896956 add file3
bbad65a53a9f545c11f8db8678a63b47201dc433 (dev) add file2
f0c5b7b7674cc7956f3d89e06ce041ccaa4cd545 add file1
acd1ecdd9f88a1c936ca15946bc48c1104ef55f2 add readme file
```

我们的提交历史变更成这样。我们再看看 rebase 的区别。

# rebase
先把我们的记录恢复到之前的 master 状态。

```
git reset --hard head^
git rebase dev
git log --pretty=oneline 
e90bb2e9d5996d16c45682596dd4014c4981dd41 (HEAD -> master) add file3
bbad65a53a9f545c11f8db8678a63b47201dc433 (dev) add file2
f0c5b7b7674cc7956f3d89e06ce041ccaa4cd545 add file1
acd1ecdd9f88a1c936ca15946bc48c1104ef55f2 add readme file
```

操作结果有所不同。 merge 显示的历史顺序和我们进行的合并操作有关，而 rebase 显示顺序和我们实际动作发生的过程相关。同时，rebase 少了一个 join 操作。

我们来换个办法，在 dev 分支上 rebase master 然后，再合并。可是这个时候我懵逼了，我们如何取消了已经 rebase 的操作呢？ 没有对应的 undo 操作。只能通过找到 ref-log 来操作。

```
git reflog

e90bb2e (HEAD -> master) HEAD@{0}: rebase finished: returning to refs/heads/master
e90bb2e (HEAD -> master) HEAD@{1}: rebase: add file3
bbad65a (dev) HEAD@{2}: rebase: checkout dev
1d38148 HEAD@{3}: reset: moving to head^
b67203a HEAD@{4}: merge dev: Merge made by the 'recursive' strategy.
1d38148 HEAD@{5}: checkout: moving from master to master
1d38148 HEAD@{6}: checkout: moving from dev to master
bbad65a (dev) HEAD@{7}: checkout: moving from master to dev
1d38148 HEAD@{8}: commit: add file3
f0c5b7b HEAD@{9}: checkout: moving from dev to master
bbad65a (dev) HEAD@{10}: commit: add file2
f0c5b7b HEAD@{11}: checkout: moving from master to dev
f0c5b7b HEAD@{12}: commit: add file1
acd1ecd HEAD@{13}: commit (initial): add readme file
```

```
git reset --hard 1d38148
git co dev
git rebase master
git log --pretty=oneline 
0b5d862609228abb1255775ab9c2124f01fd7ed5 (HEAD -> dev) add file2
1d38148fa8a973e41f41e02b9554b06137896956 (master) add file3
f0c5b7b7674cc7956f3d89e06ce041ccaa4cd545 add file1
acd1ecdd9f88a1c936ca15946bc48c1104ef55f2 add readme file
```
似乎可以发现一个问题。当我们在 dev 分支上 rebase master 的时候，实际上是找到两者共同的祖先，然后先进行 master 的重放，再把 dev 自己的修改放在后面去。

仔细观察一下我们的日志：

```
0b5d862 (HEAD -> dev) HEAD@{4}: merge dev: Fast-forward
1d38148 (master) HEAD@{5}: checkout: moving from dev to master
0b5d862 (HEAD -> dev) HEAD@{6}: rebase finished: returning to refs/heads/dev
0b5d862 (HEAD -> dev) HEAD@{7}: rebase: add file2
1d38148 (master) HEAD@{8}: rebase: checkout master
bbad65a HEAD@{9}: checkout: moving from master to dev
```

1. 先切换到 dev 分支
2. rebase： 检出 master 上的更新
3. 应用 dev 分支上的更新 **add file2**
4. 更新 HEAD 指向最新 dev 分支

# 我的工作流程

* 日常的工作在 dev 分支。
* 需要提交的时候，git svn rebase 更新 svn 代码下来
* 把 master 的变更应用到 dev
* 合并到主分支
* git svn dcommit



