---
title: spacemacs更改快捷键
date: 2018-04-12 23:31:30
categories:
 - spacemacs
tags:
 - spacemacs
---

用了大半年的spacemacs,确实感受到了什么叫操作系统级别的编译器,头疼不会Lisp语言,要想按自己的想法配置下，基本都是照葫芦画瓢；今天看了下markdown-mode,感觉有很多快捷还是挺常用的，但是原始设计的按键弧度太长，而却`leader-key`是`SPC`,意味着必须要退出`insert-mode`才能使用......

## 尝试自定义快捷键

首先想到的当然是尝试在自定义的`layer`中，再加载一遍`markdown-mode`;使用了几种不同的Lisp代码，尝试加载自定义的快捷键

### 初始化markdown-mode时绑定快捷键

``` emacs-lisp
(defun pan/init-markdown-mode()
  (use-package markdown-mode
    :defer t
    :init
    :bind (("M-n 1" . markdown-insert-header)
           )
    ))
```

结果如下:

![err]({{"/assets/images/spacesmacs/key-bind-err.png" | absolute_url}})


### 尝试修正错误

* 根据错误提示应该是`M-n 1`快捷键已经绑定了其它函数了，不能再次绑定了，于是想到了先情空该快捷键

``` emacs-lisp
(global-set-key (kbd "M-n 1") 'nil)
```

* 于是修改代码如下

``` emacs-lisp
(global-set-key (kbd "M-n 1") 'nil)
(defun pan/init-markdown-mode()
  (use-package markdown-mode
    :defer t
    :init
    :bind (("M-n 1" . markdown-insert-header)
           )
    ))
```

问题依旧，错误提示也是一样的。。

### 换个快捷键

* `non-prefix key`大概的意思就是`M-n`不是一个快捷键的前缀，而就是一个快捷键；

* 尝试使用`M-m 1`

``` emacs-lisp
(defun pan/init-markdown-mode()
  (use-package markdown-mode
    :defer t
    :init
    :bind (("M-m 1" . markdown-insert-header)
           )
    ))
```

测试是不报错了，但是并没有生效。。暂时就先放弃吧，等着使用进一步熟悉后，再来解决这个问题。


