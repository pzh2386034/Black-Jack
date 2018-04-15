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

## 初始化markdown-mode时绑定快捷键

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
                                             

## 尝试修正错误

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

## 换个快捷键

* `non-prefix key`大概的意思就是`M-n`不是一个快捷键的前缀，而就是一个快捷键；

* 通过`C-h k`查询了下`M-n`绑定的函数，是`markdown-next-link`,也就是说该快捷键只有在`markdown mode`被加载时才会生效，无法使用`global-set-key`去绑定

* 尝试使用`M-r 1`(因为`M-r`似乎无关紧要)

``` emacs-lisp
(defun pan/init-markdown-mode()
  (use-package markdown-mode
    :defer t
    :init
    :bind (("M-r 1" . markdown-insert-header)
           )
    ))
```

测试是不报错了，但是并没有生效。。

## final slove ###

最终综合前面两种方法

* 选择`M-r`为快捷键前缀(该快捷键是emacs默认的，启动即加载，方便去加载)

* 先去绑定emacs绑定的默认函数，再绑定我们自定义的函数

* 代码如下

``` emacs-lisp
(global-set-key (kbd "M-r") 'nil)
(defun pan/init-markdown-mode ()
  (use-package markdown-mode
    :init
    :config
    ;; (use-package moccur-edit)
;;    (unbind-key "M-n" markdown-next-link)
   :bind
   (
    ("M-r -" . org-ctrl-c-minus)
    ("M-r h" . markdown-insert-header)
    ("M-r 1" . markdown-insert-header-atx-1)
    ("M-r 2" . markdown-insert-header-atx-2)
    ("M-r 3" . markdown-insert-header-atx-3)
    ("M-r l" . markdown-insert-link-button)
    ("M-r u" . markdown-insert-uri)
   ;; ("M-r t" . markdown-toc-generate-toc)
    ("M-r t" . table-insert)
    ("M-r c" . markdown-insert-gfm-code-block)
    ("M-r q" . markdown-insert-blockquote)
    ("M-r q" . markdown-insert-blockquote)
    ("M-r i" . markdown-insert-italic)
    ("M-r b" . markdown-insert-bold)
   )
))
```

## 遗留问题 ###

* 如何去绑定package自定义延时加载的快捷键, 如下代码会报错

``` emacs-lisp
(defun pan/init-markdown-mode ()
  (use-package markdown-mode
    :init
    :config
        (unbind-key "M-n" markdown-next-link)
    :bind
       (
           ("M-r -" . org-ctrl-c-minus)
       ) 
))
```

### 遗留问题解决 ###

* 在`:bin`区域中定义快捷键的好处是，可以在`describe-personal-keybindings`中查到自己定义的快捷键

```emacs-lisp
(defun pan/post-init-markdown-mode ()
  (with-eval-after-load 'markdown-mode
    (use-package markdown-mode
      :init
      (add-hook 'markdown-mode-hook
                (lambda ()
                  (local-unset-key (kbd "M-n"))
                  ;; (define-key markdown-mode-map (kbd "M-n 1") 'markdown-insert-header)
                )
      )
      :bind (
             ("M-n h" . markdown-insert-header)
      )
    )
  )
)
```

### 参考链接 ###

[emacs china](https://emacs-china.org/t/topic/5563/5)

[stackoverflow](https://stackoverflow.com/questions/19324644/emacs-unbind-a-modes-keybinding)

