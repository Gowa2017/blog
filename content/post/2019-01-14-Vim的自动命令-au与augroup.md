---
title: Vim的自动命令-au与augroup
categories:
  - Vim
date: 2019-01-14 21:50:02
updated: 2019-01-14 21:50:02
tags: 
  - Vim
---
还是在把 Vim 配置为 Go 的开发环境的时候看到使用了 augroup/autocmd 命令的。但是当时没有深究，后来在很多地方看到了，所以就想要看一下其到底有什么用的。
<!--more-->

# 简介

可以在读、写文件，进入或离开缓冲区，或者退出 Vim 的时候自动执行一些命令。命令的基本格式是：

```vim
:au[tocmd] [group] {event} {pat} [nested] {cmd}
```
其基本意思就是：把 *{cmd}* 命令添加到 Vim 在一个文件匹配 *{pat}* 并遇到 *{event}* 事件时会自动执行的命令列表中。 Vim总是在已存在的自动命令后添加 *{cmd}*。
# {event}

可以在 Vim 中用命令 :help event 来查看事件。


# augroup {name}

为后面的 autocmd 命令定义一个自动命令组。`augroup END|end` 选择默认的自动命令组。为了避免混淆， `{name}` 应该与已经存在 `{event}` 相区别开来。


`augroup! {name}` 会删除一个组。在尚有 autocmd 命令使用要这个组时，不要这样干，不然会出错。

在组内定义自动命令的基本过程是:

1. 用命令 `:autogroup {name}` 选择组
2. 用命令 `:au!` 删除所有老的命令
3. 定义新的命令 `:au`
4. 回到默认组 `:augroup END`

例子：

```vim
:augroup uncompress
:   au!
:   au BufEnter *.gz %!gunzip
:augroup END
```

这会防止自动命令被执行两次。（如 source .vimrc 两次或多次）。


# augroup go

```vim
augroup go
  autocmd!
  " 不在设置全局绑定
  autocmd FileType go nmap <C-g> :GoDeclsDir<cr>
  autocmd FileType go imap <C-g> <esc>:<C-u>GoDeclsDir<cr>

  " Show by default 4 spaces for a tab
  autocmd BufNewFile,BufRead *.go setlocal noexpandtab tabstop=4 shiftwidth=4

  " :GoBuild and :GoTestCompile
  autocmd FileType go nmap <leader>b :<C-u>call <SID>build_go_files()<CR>

  " :GoTest
  autocmd FileType go nmap <leader>t  <Plug>(go-test)

  " :GoRun
  autocmd FileType go nmap <leader>r  <Plug>(go-run)

  " :GoDoc
  autocmd FileType go nmap <Leader>d <Plug>(go-doc)

  " :GoCoverageToggle
  autocmd FileType go nmap <Leader>c <Plug>(go-coverage-toggle)

  " :GoInfo
  autocmd FileType go nmap <Leader>i <Plug>(go-info)

  " :GoMetaLinter
  autocmd FileType go nmap <Leader>l <Plug>(go-metalinter)

  " :GoDef but opens in a vertical split
  autocmd FileType go nmap <Leader>v <Plug>(go-def-vertical)
  " :GoDef but opens in a horizontal split
  autocmd FileType go nmap <Leader>s <Plug>(go-def-split)

  " :GoAlternate  commands :A, :AV, :AS and :AT
  autocmd Filetype go command! -bang A call go#alternate#Switch(<bang>0, 'edit')
  autocmd Filetype go command! -bang AV call go#alternate#Switch(<bang>0, 'vsplit')
  autocmd Filetype go command! -bang AS call go#alternate#Switch(<bang>0, 'split')
  autocmd Filetype go command! -bang AT call go#alternate#Switch(<bang>0, 'tabe')
augroup END
```

上面就是一个不错的例子了


