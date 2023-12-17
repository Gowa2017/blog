---
title: git-svn来管理svn仓库
categories:
  - Git
date: 2018-08-03 12:44:57
updated: 2018-08-03 12:44:57
tags: 
  - Git
  - Svn
---
个人是比较喜欢Git的，至于说svn与git熟优熟劣，这个见仁见智，不过对于代码管理而言，svn会省很多事情，但是对于分支开发来说，git确实要好用点。公司开始就使用svn，我也只能去习惯，所以我打算用git来管理svn代码。这不，看一下关于git-svn的文档来了解一下。

<!--more-->

本页面原文：[https://git-scm.com/docs/git-svn#git-svn-svnuseSvmProps](https://git-scm.com/docs/git-svn#git-svn-svnuseSvmProps)
# git svn 命令描述
`git svn`是 git 与 svn 之间的转换通道。其在git,svn之间提供了一个双向的通道。

`git svn` 可以跟踪一个标准的 遵循 *trunk/branches/tags* 布局结构的svn 版本库（通过 --stdlayout 选项指定）。其也可以跟踪其他以 `-T/-t/-b`指定的布局。在下面的`init`命令我们会看到。

一旦 跟踪了一个版本库后，本地的Git版本库就可以 `fetch` 从svn 库拖代码下来，而svn版本库可以通过git命令 `dcommit` 命令进行更新。

# 使用参考
## **init**  
初始化一个空的Git版本库，但会有额外的 `git svn`需要的元数据。svn版本库的URL可以被当做一个命令行参数传递，或作为 `-T/-t/-b` 的参数。可选地，目标目录可以被指定为第二个命令行参数。通常，这个命令初始化当前目录。  
**-T\<trunk\_subdir>  
--trunk=\<trunk\_subdir>  
-t\<tags\_subdir>  
--tags=\<tags\_subdir>  
-b\<branches\_subdir>  
--branches=\<branches\_subdir>**  
**-s, --stdlayout**  
上面这些参数对于`init`来说都是可选的。这其中的每个参数都可指向相对于版本库内的相对路径（*--tags=project/tags*），或者一个完整地址（*--tags=https://foo.org/project/tags*）。 当svn版本库将 tags 或 branches放到多个路径时，可以指定一个或多个 `--tags, --branchs`。`--stdlayout`是把 `tags,branches,trunk`设置为相对路径（svn默认方式）的简写。其他选项的优先级高于`--stdlayout`。  当`init`执行后，在*.git/config*文件内，会有一个 `svn-remote`的节点。


```
[svn-remote "svn"]
        url = svn://guan.isum.cn/smart/nanming/code/smartpeople/android/SmartConsumer
        fetch = :refs/remotes/git-svn
```
**--no-metadata** 在上面的节点内设置 *noMetadata* 选项。一个不推荐的设置。阅读本页面的 `svn.noMetadata` 一节的内容来查看更多细节。

**--use-svm-props**
Set the useSvmProps option in the [svn-remote] config.

**--use-svnsync-props**
Set the useSvnsyncProps option in the [svn-remote] config.

**--rewrite-root=\<URL>**
Set the rewriteRoot option in the [svn-remote] config.

**--rewrite-uuid=\<UUID>**
Set the rewriteUUID option in the [svn-remote] config.

**--username=\<user>** 为传递需要svn处理的认证信息，指定用户名。对于其他传输（如 ssh+svn），必须在URL内包含用户名。

**--prefix=\<prefix>**  如果指定了*trunk/branches/tags*，此选项允许指定一个放在远程地址前的前缀。这个前缀不会自动包含一个`/`，需要我们手动加上。如果`--branches/-b`已经指定，那么这个前缀必须包含一个拖尾的`/`。强烈建议在任何时候都使用带有`/`的前缀，这样SVN跟踪的引用会被定位到`refs/remotes/$prefix/`，这与git本身的对于远程跟踪引用的布局兼容`refs/remotes/$remote/`。对于在同一个共用的资源库内跟踪多个不同的项目时使用一个前缀也很有用。默认情况下， 前缀设置为 `origin/`。
> 在git 2.0前，默认的前缀是 `""`（无前缀）。这就意味着SVN跟踪的引用是放在`refs/remotes/*`，这与现在的git自身的跟踪布局并不兼容。如果为了与以前的模式相兼容的话，传递一个空的前缀`""`。

**--ignore-refs=\<regex>** 当把这个表达式传递给 `clone, init`时，这个表达式会被保留为一个配置使用的键。查看 `fetch` 一节中关于 *--ignore-refs*的描述。

**--ignore-paths=\<regex\>** 描述同上，具体作用查看后面的章节。

**--include-paths=\<regex>** 描述同上，具体作用查看后面的章节。

**--no-minimize-url** 当跟踪多个目录时（通过 `--stdlayout, --branches, --tags选项`），`git svn`会尝试连接到svn库的根目录下。这种方式在整个库都被移动的时候对于跟踪历史信息很有用，但在读权限限制的时候这可能会出现些问题。`--no-minimize-url`可以取消这个默认的做法。默认情况下这个选项是关闭的。





##  **fetch**
从版本库拉取我们跟踪，但是本地没有的代码。这个操作会根据需要自动更新*rev_map*。（在后面的 File 一节查看 *$GIT_DIR/svn/*\*/.rev_map.**）

**--localtime** 以本地时区来存储 commint 时间，而不是以UTC时间。这会让 `git log`（即使没有 `date=local`）也会显示来与 `svn log`时间一致（svn log 使用的是本地时区时间）。

**--parent** 只从版本库获取当前 HEAD。
**--ignore-refs=\<regex>** 忽略匹配这个给定的表达式的tags或 branches。 `^refs/remotes/origin/(?!tags/wanted-tag|wanted-branch).*$` 可以用来允许指定的引用。

`config key: svn-remote.<name>.ignore-refs`

如果 忽略引用的键已经设置，同事命令行参数也有指定，那么两个地方的值都会被使用。

**--ignore-paths=\<regex>** 每次都会忽略这个选项指定的路径。

`config key: svn-remote.<name>.ignore-paths`

这个选项在命令行和配置文件内指定都有。

## **clone**
相当于 `init` 后执行  `fetch`命令

## **rebase**
从SVN 获取当前HEAD的版本，同时变换基准工作位置到对应的revision（这不会提交到 SVN）。

这个命令与 `svn update`或 `git pull`，例外的是它使用`git rebase`而不是`git merge`来保留线性历史记录，以便于使用`git svn`进行dcommitting。

此命令接受所有 `git svn fetch， git rebase`接受的选项。然而，`--fetch-all`只会从当前的 [svn-remote]节点获取，而不是所有的 [svn-remote]节点。

和 `git rebase`类似，这个操作要求工作树是干净的，同时没有任何为提交的变更。

如果有必要的话，这个操作会自动的更新 *rev_map*。

**-l, --local** 不从远程获取；只是在之前从 SVN 库内获取的最新 提交上执行 ` git rebase`。
## **dcommit**

把当前分支上的不同直接提交到 SVN 库内，接着会 rebase 或 reset（这依赖于SVN和head间是否有变更）。git中的每个commit都会在svn中创建一个 revision。

当一个可选的 git 分支名称（或一个 git 提交对象名称）指定的时候，下面的子命令工作在对应的分支上，而不是工作在当前分支上。

**--no-rebase** 在提交后，不要 `rebase or reset`。

**--commit-url \<URL>**

## **branch**
在SVN内创建一个分支。

**-m, --message** 提交一个描述信息。

**-t, --tag** 使用 *tags_subdir* 而不是 *branches_subdir*来建立一个tag（这两个dir都是在 init 命令中指定的）。

**-d\<path>,--destination=\<path>** 在`init, clone`的时候，如果指定了多于一个的 *--branches, --tags*，必须要指定要创建分支（标签）的位置。用以下命令可以查看所有引用的分支或标签：

```
git config --get-all svn-remote.<name>.branches
git config --get-all svn-remote.<name>.tags
```

其中 *\<name>* 是 SVN 版本库的名称，在`init`的时候，由 `-R`指定（默认情况下是 `svn`）。

**--username** 指定本次提交的用户名。

**--commit-url**
Use the specified URL to connect to the destination Subversion repository. This is useful in cases where the source SVN repository is read-only. This option overrides configuration property commiturl.

`git config --get-all svn-remote.<name>.commiturl`

**--parents**
Create parent folders. This parameter is equivalent to the parameter --parents on svn cp commands and is useful for non-standard repository layouts.


## **tag**
创建标签，这个等于 `branch -t`
## **log**
查看日志。对于 `svn log` 命令的特性被支持：

**-r \<n>[:\<n>],--revision=\<n>[:\<n>]** 还有些非数字的参数：*HEAD, NEXT, BASE, PREV*。

**--limit=\<n>**  

**--incremental**

新特性：

**--show-commit**

**--oneline**


## **blame**
##  **find-rev**
##  **set-tree**
##  **create-ignore**
##  **show-ignore**
##  **mkdirs**
##  **commit-diff**
##  **info**
##  **proplist**
##  **propget**
##  **propset**
##  **show-externals**
##  **gc**
##  **reset**


# 选项

* **--shared[=(false|true|umask|group|all|world|everybody)]**
* **--template=<template_directory>**
* **-r <arg>, --revision <arg>**
* **-stdin**
* **--rmdir**
* **-e, --edit**
* **-l num, --find-copies-harder**
* **-A <filename>, --authors-file=<filename>**
* **--authors-prog=<filename>**
* **-q, --quiet**
* **-m, --merge**
* **-s<strategy>, --strategy=<stragety>**
* **-p, --preserve-merges**
* **-n, --dry-run**
* **--use-log-author**
* **--add-author-from**


# 高级选项

* **-i<GIT_SVN_ID>, --id <GIT_SVN_ID>**
* **-R<remote name>, --svn-remote <remote name>**
* **--follow-parent**

# CONFIG FILE-ONLY OPTIONS
* **svn.noMetadata, svn-remote.<name>.noMetadata**


# BASIC EXAMPLES
# HANDLING OF SVN BRANCHES
# CAVEATS
# BUGS
# CONFIGURATION
# FILES