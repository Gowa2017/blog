---
title: ultisnips的readme文档中文
categories:
  - Vim
date: 2018-03-14 22:19:57
updated: 2018-03-14 22:19:57
tags: 
  - Vim
---
UltiSnips 是VIM中代码片段（snippets）的最终解决办法。它拥有很多的特性，而且非常快。

![GIF Demo](https://raw.github.com/SirVer/ultisnips/master/doc/demo.gif)

<!--more-->
UltiSnips
=========

展示中是在编辑一个python文件。 首先，展开了 `#!` 片段，然后是 `class` 片段。下拉菜单来自
[YouCompleteMe](https://github.com/Valloric/YouCompleteMe), UltiSnips 也集成了 [neocomplete](https://github.com/Shougo/neocomplete.vim)。我在占位符间跳转，添加文本，片段会自动更新其他地方: 当我添加 `Animal`作为一个基类时， `__init__` 进行了更新来调用基类的构造函数。 当我为构造函数添加参数时，他们会被自动的赋给实例变量。接着，我插入了我个人自己的用来打印调试的片段 `print`。要注意的是，当我离开插入模式，插入了其他的片段，然后再回来为`__init__`添加额外参数的时候，这个类的片段依然是或者的，并且会自动增加了一个实例变量。



 UltiSnips 的官方站点是 <https://github.com/sirver/ultisnips>.
 
 欢迎 pull 请求和 issues。

UltiSnips was started in Jun 2009 by @SirVer. In Dec 2015, maintenance was
handed over to [@seletskiy](https://github.com/seletskiy).

What can you do with UltiSnips?
-------------------------------

1. Advanced snippets:

    * [Snippets Aliases](doc/examples/snippets-aliasing/README.md)
    * [Dynamic Tabstops/Tabstop Generation](doc/examples/tabstop-generation/README.md)

Quick Start
-----------

假设使用的是[Vundle](https://github.com/gmarik/Vundle.vim)来管理VIM插件。 在 `.vimrc`中加入下面的代码：

    " Track the engine.
    Plugin 'SirVer/ultisnips'

    " Snippets are separated from the engine. Add this if you want them:
    Plugin 'honza/vim-snippets'

    " Trigger configuration. Do not use <tab> if you use https://github.com/Valloric/YouCompleteMe.
    let g:UltiSnipsExpandTrigger="<tab>"
    let g:UltiSnipsJumpForwardTrigger="<c-b>"
    let g:UltiSnipsJumpBackwardTrigger="<c-z>"

    " If you want :UltiSnipsEdit to split your window.
    let g:UltiSnipsEditSplit="vertical"

UltiSnips有非常详细的文档
[documentation](https://github.com/SirVer/ultisnips/blob/master/doc/UltiSnips.txt)。这里有很多选项和特性我建议你最少看一下。


Screencasts
-----------

From a gentle introduction to really advanced in a few minutes: The blog posts
of the screencasts contain more advanced examples of the things discussed in the
videos.

- [Episode 1: What are snippets and do I need them?](http://www.sirver.net/blog/2011/12/30/first-episode-of-ultisnips-screencast/)
- [Episode 2: Creating Basic Snippets](http://www.sirver.net/blog/2012/01/08/second-episode-of-ultisnips-screencast/)
- [Episode 3: What's new in version 2.0](http://www.sirver.net/blog/2012/02/05/third-episode-of-ultisnips-screencast/)
- [Episode 4: Python Interpolation](http://www.sirver.net/blog/2012/03/31/fourth-episode-of-ultisnips-screencast/)

Also the excellent [Vimcasts](http://vimcasts.org) dedicated three episodes to
UltiSnips:

- [Meet UltiSnips](http://vimcasts.org/episodes/meet-ultisnips/)
- [Using Python interpolation in UltiSnips snippets](http://vimcasts.org/episodes/ultisnips-python-interpolation/)
- [Using selected text in UltiSnips snippets](http://vimcasts.org/episodes/ultisnips-visual-placeholder/)
