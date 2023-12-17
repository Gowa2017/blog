---
title: vim-利用Vundle管理插件
categories:
  - Vim
date: 2017-12-17 15:58:23
updated: 2017-12-17 15:58:23
tags: 
  - Vim
---
Vundle是Vim bundle的一个简称，一个Vim插件管理器。开源项目地址位于[Vundle.vim.git](https://github.com/VundleVim/Vundle.vim)
只需要在`.vimrc`文件内加上几行，你就可以使用它来下载、卸载、管理各种位于github、官方插件仓库等地方的插件。
<!-- more -->



下面是一张截图：
![Vundle界面展示](https://camo.githubusercontent.com/bc559468e6623d18947ced1ef353f68f6116e45a/687474703a2f2f692e696d6775722e636f6d2f527565683743632e706e67)

# 安装
1. 安装需要`git`的支持，采用克隆的方式先克隆到本地。

	git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim

2. 在配置文件内加上几行。
```
	set nocompatible              " be iMproved, required
	filetype off                  " required
	
	" set the runtime path to include Vundle and initialize
	set rtp+=~/.vim/bundle/Vundle.vim
	call vundle#begin()
	" alternatively, pass a path where Vundle should install plugins
	"call vundle#begin('~/some/path/here')
	
	" let Vundle manage Vundle, required
	Plugin 'VundleVim/Vundle.vim'
	
	" The following are examples of different formats supported.
	" Keep Plugin commands between vundle#begin/end.
	" plugin on GitHub repo
	Plugin 'tpope/vim-fugitive'
	" plugin from http://vim-scripts.org/vim/scripts.html
	" Plugin 'L9'
	" Git plugin not hosted on GitHub
	Plugin 'git://git.wincent.com/command-t.git'
	" git repos on your local machine (i.e. when working on your own plugin)
	Plugin 'file:///home/gmarik/path/to/plugin'
	" The sparkup vim script is in a subdirectory of this repo called vim.
	" Pass the path to set the runtimepath properly.
	Plugin 'rstacruz/sparkup', {'rtp': 'vim/'}
	" Install L9 and avoid a Naming conflict if you've already installed a
	" different version somewhere else.
	" Plugin 'ascenator/L9', {'name': 'newL9'}
	
	" All of your Plugins must be added before the following line
	call vundle#end()            " required
	filetype plugin indent on    " required
	" To ignore plugin indent changes, instead use:
	"filetype plugin on
	"
	" Brief help
	" :PluginList       - lists configured plugins
	" :PluginInstall    - installs plugins; append `!` to update or just :PluginUpdate
	" :PluginSearch foo - searches for foo; append `!` to refresh local cache
	" :PluginClean      - confirms removal of unused plugins; append `!` to auto-approve removal
	"
	" see :h vundle for more details or wiki for FAQ
	" Put your non-Plugin stuff after this line
```
3. 运行`:PluginInstall`命令就会将指定的插件进行安装。
