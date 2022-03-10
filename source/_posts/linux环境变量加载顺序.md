---
title: linux环境变量加载顺序
date: 2021-10-13 16:49:14
tags: linux
---
# 01、环境变量文件描述

**/etc/profile**: 此文件为系统的每个用户设置环境信息,当用户第一次登录时,该文件被执行,并从/etc/profile.d目录的配置文件中搜集shell的设置.
**/etc/bashrc**: 为每一个运行bash shell的用户执行此文件.当bash shell被打开时,该文件被读取.

//用户级别的环境变量，用户可以覆盖全局变量
**~/.bash_profile**: 每个用户都可使用该文件输入专用于自己使用的shell信息,当用户登录时,该文件仅仅执行一次!默认情况下,他设置一些环境变量,执行用户的.bashrc文件.
**~/.bashrc**: 该文件包含专用于你的bash shell的bash信息,当登录时以及每次打开新的shell时,该文件被读取.
**~/.bash_logout**: 当每次退出系统(退出bash shell)时,执行该文件.

/etc/profile中设定的变量(全局)的可以作用于任何用户,
而~/.bashrc等中设定的变量(局部)只能继承/etc/profile中的变量,他们是"父子"关系.

~/.bash_profile 是交互式、login 方式进入 bash 运行的
~/.bashrc 是交互式 non-login 方式进入 bash 运行的
通常二者设置大致相同，所以通常前者会调用后者

## 一、系统环境变量：

**/etc/profile** ：这个文件预设了几个重要的变量，例如PATH, USER, LOGNAME, MAIL, INPUTRC, HOSTNAME, HISTSIZE, umask等等。

为系统的每个用户设置环境信息。当用户第一次登陆时，该文件执行，并从/etc/profile.d目录中的配置文件搜索shell的设置（可以用于设定针对全系统所有用户的环境变量，环境变量周期是永久的）

**/etc/bashrc** ：这个文件主要预设umask以及PS1。这个PS1就是我们在敲命令时，前面那串字符了，例如 [root@localhost ~]#,当bash shell被打开时,该文件被读取

这个文件是针对所有用户的bash初始化文件，在此设定中的环境信息将应用与所有用户的shell中，此文件会在用户每次打开shell时执行一次。（即每次新开一个终端，都会执行/etc/bashrc）**

## 二、用户环境变量：

**.bash_profile** ：定义了用户的个人化路径与环境变量的文件名称。每个用户都可使用该文件输入专用于自己使用的shell信息,当用户登录时,该文件仅仅执行一次。（在这个文件中有执行.bashrc的脚本）

**.bashrc** ：该文件包含专用于你的shell的bash信息,当登录时以及每次打开新的shell时,该该文件被读取。例如你可以将用户自定义的alias或者自定义变量写到这个文件中。

**.bash_history** ：记录命令历史用的。

**.bash_logout** ：当退出shell时，会执行该文件。可以把一些清理的工作放到这个文件中。

linux加载配置项时通过下面方式
首先 加载/etc/profile配置

然后 加载/ect/profile.d/下面的所有脚本

然后 加载当前用户 .bash_profile

然后 加载.bashrc

最后 加载 [/etc/bashrc]

/etc/profile → /etc/profile.d/*.sh → ~/.bash_profile → ~/.bashrc → [/etc/bashrc]

