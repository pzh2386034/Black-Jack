---
title: emacs-error
date: 2018-04-19 01:04
categories:
- spacemacs
tags:
- spacemacs
---

最近修改emacs配置比较频繁,git提交的时候可能有些改动也没看仔细，`clean-aindent-mode`包出了问题，导致后面安装一些别的有依赖的包都出了问题，如下图

    >File mode specification error: (void-function clean-aindent-mode)

![err]({{"/assets/images/spacesmacs/clean-aindent.png" | absolute_url}})

使用`emacs --debug-info`启动，可以看见如下错误打印

![err]({{"/assets/images/spacesmacs/clean-aindent3.png" | absolute_url}})

不安装有依赖的包，启动则报错

    >symbol's function definition is void: global-company-mode

## 第一次尝试修复 ##

感觉是`clean-aindent-mode`这个包还没生效的时候调用了函数`clean-aindent-mode`函数，我尝试在安装latex之前先安装一遍`clean-aindent-mode`, 代码如下

![test1]({{"/assets/images/spacesmacs/clean-aindent2.png" | absolute_url}})

没起作用，google搜了一波，也没有什么信息；于是版本回退，重下`spacesmacs`等等，一些列努力均失败；

## 第二次尝试 ##

扫了一遍elisp语法，有灵感，应该是手动加载的方法不对，尝试使用如下方法加载
  
  * 先手动下载`clean-aindent-mode`包，见[官网](https://github.com/pmarinov/clean-aindent-mode)
  * 将`clean-aindent-mode.el`文件放到`~/.spacemacs.d/private/pan`目录
  * 使用如下代码手动加载

``` emacs-lisp
(add-to-list 'load-path "~/.spacemacs.d/private/pan")
(require 'clean-aindent-mode)
```
保存，重启，顺利安装包,顺利启动

## 总结 ##

应该说这是很基本的知识了，但是还是花了好长时间才搞定;于是顺便补齐下基本知识
  * load:导入文件的基本函数,会尝试加后缀导入
  * load-file: 导入特定文件，如果知道文件完整名称，优先使用该函数
  * require:加载还未被加载的包；首先会查看变了features中是否存在所要加载的符号，如果不存在会自动使用`load`加载
  * autoload:仅在函数调用时加载文件
  
## 参考链接 ##

[Elisp文档](http://ergoemacs.org/emacs/elisp_library_system.html)

[emacs china](https://emacs-china.org/t/topic/5615)
