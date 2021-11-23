---
title: vim-plug 插件
date: 2021-11-23 09:48:16
tags:
---


# 简介

vim-plug 是 vim 的一个插件管理器，功能非常多。具体描述看它的 [GITHUB](https://github.com/junegunn/vim-plug) 主页。

# 安装

## 脚本下载

Unix 环境

```sh
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

Windows 环境

```sh
iwr -useb https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim |`
    ni $HOME/vimfiles/autoload/plug.vim -Force
```

## 配置 .vimrc

执行 `vim ~/.vimrc` ，输入以下内容：

```text
call plug#begin('~/.vim/plugged')

" Make sure you use single quotes

" Shorthand notation; fetches https://github.com/junegunn/vim-easy-align
Plug 'junegunn/vim-easy-align'

" Any valid git URL is allowed
Plug 'https://github.com/junegunn/vim-github-dashboard.git'

" Multiple Plug commands can be written in a single line using | separators
Plug 'SirVer/ultisnips' | Plug 'honza/vim-snippets'

" On-demand loading
Plug 'scrooloose/nerdtree', { 'on':  'NERDTreeToggle' }
Plug 'tpope/vim-fireplace', { 'for': 'clojure' }

" Using a non-default branch
Plug 'rdnetto/YCM-Generator', { 'branch': 'stable' }

" Using a tagged release; wildcard allowed (requires git 1.9.2 or above)
Plug 'fatih/vim-go', { 'tag': '*' }

" Plugin options
Plug 'nsf/gocode', { 'tag': 'v.20150303', 'rtp': 'vim' }

" Plugin outside ~/.vim/plugged with post-update hook
Plug 'junegunn/fzf', { 'dir': '~/.fzf', 'do': './install --all' }

" Unmanaged plugin (manually installed and updated)
Plug '~/my-prototype-plugin'

" Initialize plugin system
call plug#end()
```

然后关闭，重新进入 vim 界面，输入`:PlugInstall` 安装 vim-plug 包管理器。

按照时如果出现无法访问 github 地址，此时可以设置代理，可以自己使用 proxyclient 软件启动代理服务。使用 `export HTTPS_PROXY=socks5://127.0.0.1:5000` 设置代理服务。

# GDB 插件

请先安装 gdb 工具。

我使用 [NeoDebug](https://github.com/cpiger/NeoDebug) 这个插件。编辑 `.vimrc` 文件，加上如下代码：

```text
Plug 'cpiger/NeoDebug'
```

退出 vim 后再进入 vim，执行 `:PlugInstall` 安装。

# ctrlp模糊查找

使用 [ctrlp.vim](https://github.com/ctrlpvim/ctrlp.vim)。编辑 `.vimrc` 文件，加上如下代码：

```text
Plug 'ctrlpvim/ctrlp.vim'
```

退出 vim 后再进入 vim，执行 `:PlugInstall` 安装。

# nerdtree目录导航

使用 [nerdtree](https://github.com/preservim/nerdtree)。编辑 `.vimrc` 文件，加上如下代码：

```text
Plug 'ctrlpvim/ctrlp.vim'
```

退出 vim 后再进入 vim，执行 `:PlugInstall` 安装。

# taglist

使用 [taglist](https://www.vim.org/scripts/script.php?script_id=273) 可以在小窗线上变量方法名，快速定位。

先安装 [ctags](http://ctags.sf.net) 工具，Ubuntu 下执行 `sudo apt install ctags` 安装。

然后编辑 `.vimrc` 文件，加上如下代码：

```text
Plug 'yegappan/taglist'
```

退出 vim 后再进入 vim，执行 `:PlugInstall` 安装。