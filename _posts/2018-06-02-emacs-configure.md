---
title: emacs-configure
date: 2018-06-02 22:06
categories:
- spacemacs
tags:
- spacemacs
---

>本文主要记录spacemacs配置c-c++编辑器,Python编辑器,web前端编辑器；以备以后翻阅

## c-c++配置 ##

使用`c-c++`layer及`company mode`是必须的，不需要再说了

### company-c-headers ###

在安装`c-c++`layer时，会自动安装该package，我们只需要配置下头文件目录，如果大型工程，涉及上百万行代码的，使用semantic可能会比较卡，这时候其实只要使用`company-c-headers`package，就已经能实现较为不错的函数自动补全功能

``` emacs-lisp
(with-eval-after-load 'company-c-headers
  (add-to-list 'company-c-headers-path-system "/usr/include/c++/4.2.1")
)
(add-hook 'c++-mode-hook
          (lambda ()
            (set (make-local-variable 'company-backends) '(company-semantic company-c-headers  company-dabbrev-code  company-gtags company-c-headers company-keywords company-files company-dabbrev))
        )
    )
```

### ggtags ###

当代码量比较大时，推荐使用ggtags进行静态代码补全即可，需要安装一个`gnu global`(brew install global)，再配置一下更方便的快捷键就行了

``` emacs-lisp
(setq
  helm-gtags-ignore-case t
  helm-gtags-auto-update t
  helm-gtags-use-input-at-cursor t
  helm-gtags-pulse-at-cursor t
  helm-gtags-prefix-key "\C-cg"
  helm-gtags-suggested-key-mapping t
  )
(set 'helm-gtags-mode t)
(with-eval-after-load 'helm-gtags
  (define-key helm-gtags-mode-map (kbd "s-g s") 'helm-gtags-select)
  (define-key helm-gtags-mode-map (kbd "s-g r") 'helm-gtags-find-rtag)
  (define-key helm-gtags-mode-map (kbd "s-g f") 'helm-gtags-find-symbol)
  (define-key helm-gtags-mode-map (kbd "s-<") 'helm-gtags-previous-history)
  (define-key helm-gtags-mode-map (kbd "s->") 'helm-gtags-next-history)
  (define-key helm-gtags-mode-map (kbd "s-,") 'helm-gtags-pop-stack)
  (define-key helm-gtags-mode-map (kbd "s-g c") 'helm-gtags-create-tags)
  (define-key helm-gtags-mode-map (kbd "s-g D") 'helm-gtags-dwim-other-window)
  (define-key helm-gtags-mode-map (kbd "s-g d") 'helm-gtags-dwim)
  (define-key helm-gtags-mode-map (kbd "s-g t") 'helm-gtags-tags-in-this-function)
  (define-key helm-gtags-mode-map (kbd "s-g k") 'helm-gtags-show-stack)
  (define-key helm-gtags-mode-map (kbd "s-g a") 'helm-gtags-clear-stack)
  )
```

### semantic ###

semantic是emacs重要的动态代码解析工具，对于第一次打开一个工程的代码文件，他会自动扫描头文件库及buffer中的关键字，生成一个semanticdb数据库；动态自动补全的原材料就从中来；

它是(cedet)[http://cedet.sourceforge.net]主要组成部分，也可以安装官方的介绍直接安装`cedet`，省心

它包含几种常用的mode

  * global-semanticdb-minor-mode: 支持semanticdb，必须
  * global-semantic-mru-bookmark-mode: 支持动态生成标签，可以通过global-semantic-switch-tags来支持动态标签间的跳转
  * global-cedet-m3-minor-mode: 激活cedet菜单，通过右键使用；没啥用
  * global-semantic-highlight-func-mode：高亮当前标签第一行，例如函数名，类名；没啥用
  * global-semantic-stickyfunc-mode：将当前标签名放在buffer顶部
  * global-semantic-decoration-mode：使用semantic-decoration-styles中定义的风格，作为tags的分割；没啥用
  * global-semantic-idle-local-symbol-highlight-mode：高亮光标所在tag；有更好用的
  * global-semantic-idle-scheduler-mode：在空闲时间自动分析源文件；必须
  * global-semantic-idle-completions-mode：触发自动补全；必须
  * global-semantic-idle-summary-mode：显示tag的信息；必须
  * global-semantic-show-unmatched-syntax-mode：显示哪些元素无法被当前解析器解释
  * global-semantic-show-parse-states-mode：显示当前解析源文件进度
  * global-semantic-highlight-edits-mode：高亮显示当前buffer哪些还没有没解析器增量处理过
  
  
semantic优化

  * 如果使用gcc，则`(require 'semantic/bocine/gcc)`可以帮我们自动搜索系统头文件目录
  * 使用ede工程，限制搜索范围
  * `(semantic-add-system-include "~/exp/include/boost_1_37" 'c++-mode)`显示的限定tags搜索范围
  * 对于关键目录，首先显示的产生tags db；例如使用`semanticdb-create-ebrowse-database`或者`semanticdb-create-cscope-database`
  * 对于特定的模式，去除某些搜索目录

  ``` emacs-lisp
  (setq-mode-local c-mode semanticdb-find-default-throttle
                 '(project unloaded system recursive))
  ```
  * 通过`semantic-idle-scheduler-idle-time`设置进入idle time的时间，默认1s

目前我的semantic使用还是非常基本的

``` emacs-lisp
(require 'cc-mode)
(require 'semantic)
(global-semanticdb-minor-mode 1)
(global-semantic-idle-scheduler-mode 1)
(global-semantic-highlight-func-mode 1) ;; active highlighting of first line for current tag
(global-semantic-stickyfunc-mode 1) ;; activates mode when name of current tag will be shown in top line of buffer
(global-semantic-idle-local-symbol-highlight-mode 1) ;; activates highlighting of local names that are the same as name of tag under cursor;
(global-semantic-idle-scheduler-mode 1) ;; activates automatic parsing of source code in the idle time;
(require 'semantic/ia)
(require 'semantic/bovine/gcc)
```

## EDE ##

ede可以允许我们以工程的形式管理代码，并通过`ede-cpp-root-project`, 帮助semantic获取额外信息，更好的分析源文件

``` emacs-lisp
(ede-cpp-root-project "Test"
                :name "Test Project"
                :file "~/work/project/CMakeLists.txt"
                :include-path '("/"
                                "/Common"
                                "/Interfaces"
                                "/Libs"
                               )
                :system-include-path '("~/exp/include")
                :spp-table '(("isUnix" . "")
                             ("BOOST_TEST_DYN_LINK" . "")))
```

  * :file  :可以使用工程根目录的任何文件，仅仅提供根目录位置
  * :system-include-path   :绝对路径
  * :include-path    :指明局部头文件搜索路径， "\"代码代码库根目录
  * :spp-table     :定义键值对，预处理时会使用
