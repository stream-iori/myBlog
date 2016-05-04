---
layout: post
title: "OS X 下设置Emacs Shell-mode 输出颜色"
date: 2014-05-02 10:53
comments: true
categories:
- Emacs
published: true

---

一旦染上Emacs, 你就会像中毒一样高度定制他. Emacs简直就是控制癖们的毕生火坑.
鄙人已经被Shell模式下的颜色折腾了大半年,只不过最近在写App.所以没怎么用到Shell就没尝试修复他.最近休息花了点时间,准备把颜色折腾出来.

### 为什么Linux下Shell-Mode会有颜色

如果你的Emacs跑在Linux, 你会发现一切都很正常.ls一下会有颜色.当然前提是你设置了ls alias `ls --color=auto`.这里流程是
* Emacs进入Shell-mode
* 会尝试加载~/.bashrc 并执行里面的内容
* 这里如果要对shell-mode进行一些设置可以判断 $TERM 变量看看是不是dump

同样的步骤同样的设置为嘛在OS X下就没用. 摔
经过各方面资料搜索才发现原来是BSD核心的问题.

由于OSX使用的Unix核心,所以很多基本的命令参数都是不一样的.比如`ls`.
在GNU Linx下你可以通过参数 `ls --color`使用颜色渲染.但是在BSD核心下你只能使用`ls --G`这个参数.OK,那改一下alias不就可以了.进入~/.bashrc增加下面的alias以及颜色配置

`alias ls='ls -G'`
`export CLICOLOR=1`
`export LSCOLORS=ExFxCxDxBxegedabagacad`

兴致冲冲开启我心爱的Emacs, M-x shell 执行`ls`, 吓~~~~木有变化,依旧是黑底白字,摔

当然也不是没有收获,进入`Ansi-term`模式发现颜色变了.突然感觉shell-mode可能不能认识BSD核心的命令.估计要替换BSD的`ls 才行`

### 设置OS X的命令使用GNU Linux式

Mac 下安装GNU命令有两种方法,port或者Homebrew.我使用brew.你们自便.
* 执行`brew install coreutils`
不出意外你们会得到UNSAFE的警告.这是coreutils upstream的问题,brew issue里有这个问题.很容易绕过去.

* 设置命令别名到.bashrc里
`alias ls='gls --color=auto'`
`alias dir='gdir --color=auto'`
`alias grep='grep --color=auto'`

再次进入shell-mode. 终于看到颜色上的变化了.欣慰~~~~

当然还可以通过`gdircolors`继续美化你的shell输出.有兴趣的可以参考[dircolors-solarized](https://github.com/seebi/dircolors-solarized)

#### PS
如果你tramp到远程服务器,发现ls还是没有颜色,千万不要再摔了,如果执行命令加`ls --color`有效果那就给远程的服务器设置一下别名吧.


















