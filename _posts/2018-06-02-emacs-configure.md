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

| global-semanticdb-minor-mode                     | 支持semanticdb                                                            | 必须              |
| global-semantic-mru-bookmark-mode                | 支持动态生成标签，可以通过global-semantic-switch-tag 来支持动态标签间跳转 | -                 |
| global-cedet-m3-minor-mode                       | 激活cedet菜单，通过右键使用                                               | -                 |
| global-semantic-highlight-func-mode              | 高亮当前标签第一行，例如函数名，类名                                      | -                 |
| global-semantic-stickyfunc-mode                  | 将当前标签名放在buffer顶部                                                | -                 |
| global-semantic-decoration-mode                  | 使用semantic-decoration-styles中定义的风格作为tags的分割                  | -                 |
| global-semantic-idle-local-symbol-highlight-mode | 高亮光标所在tags                                                          | - |
| global-semantic-idle-scheduler-mode              | 空闲时间自动分析源文件                                                    | 必须              |
| global-semantic-idle-completions-mode            | 触发自动补全                                                              | 必须              |
| global-semantic-idle-summary-mode                | 显示tags信息                                                              | 必须              |
| global-semantic-show-unmatched-syntax-mode       | 显示哪些元素无法被当前解析器解析                                          | -                 |
| global-semantic-show-parse-states-mode           | 显示当前解析源文件进度                                                    | -                 |
| global-semantic-highlight-edits-mode             | 高亮显示当前buffer哪些还没有被解析起增量处理过                            | -                 |

  
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
  * 很多函数库将所有的宏定义存放在若干个文件中，可以使用`semantic-lex-c-preprocessor-symbol-file`来分析文件，并使用其中的宏定义，例如引入Qt4函数库

  ``` emacs-lisp
  (setq qt4-base-dir "/usr/include/qt4")
(semantic-add-system-include qt4-base-dir 'c++-mode)
(add-to-list 'auto-mode-alist (cons qt4-base-dir 'c++-mode))
(add-to-list 'semantic-lex-c-preprocessor-symbol-file (concat qt4-base-dir "/Qt/qconfig.h"))
(add-to-list 'semantic-lex-c-preprocessor-symbol-file (concat qt4-base-dir "/Qt/qconfig-dist.h"))
(add-to-list 'semantic-lex-c-preprocessor-symbol-file (concat qt4-base-dir "/Qt/qglobal.h"))
  ```
  
  获取标签信息
  
    * `semantic-ia-show-doc`: 显示光标下函数或变量的基本信息; 变量显示声明的信息，函数则显示定义方式
    * `semantic-ia-show-summary`: 和上几乎一致
    * `semantic-ia-describe-class`: 查询类信息
    
  代码导航
  
    * `semantic-ia-fast-jump`: 跳转到申明处
    * `semantic-mrub-switch-tag`: return back, 仅在`semantic-mrub-bookmark-mode`minor mode模式下使用
    * `semantic-complete-jump(-local)`: 跳转到本文件(本项目)
    * `semantic-analyze-proto-impl-toggle`: 在函数申明和实现间跳转
    * `semantic-decoration-include-visit`: 跳转到头文件
    * `semantic-next(previous)-tag`: 字面翻译即可
    * `senator-go-to-up-reference`: 跳到父标签，需要实测效果
    * `semantic-symref`: 查找标签引用处
    * `semantic-symref-symbol`: 查找手动输入的标签名
    * `senator-kill-tag`, `senator-yank-tag`, `senator-copy-tag`

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



### EDE ###

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
  
  
## python ##

### anaconda mode ###

`python layer`安装就不说了，主要记录下(anaconda mode)[https://github.com/proofit404/anaconda-mode]，代码提示就靠他了

如果安装`anaconda-mode`代码提示有问题，注意以下几点

  * `cli`和`emacs`识别到的系统默认python版本可能不一致，`emacs`貌似是`python3`优先的，具体原因还没找到
  * 如果代码提示的时候发现一些系统自带的库可以提示，但是自己通过`pip`或者其他途径安装的包无法提示，可以参考下(emacs-china)[https://emacs-china.org/t/topic/6104/15]
  * 使用`virtual environment`有好处，但是我还是不太会用
  
  ``` emacs-lisp
  (add-hook 'python-mode-hook
          (lambda ()
            (set (make-local-variable 'company-backends) '(company-anaconda company-dabbrev))
            (local-set-key (kbd "c-r") 'anaconda-mode-find-references)
            (local-set-key (kbd "c-d")  'anaconda-mode-find-definitions)
            (local-set-key (kbd "c-u") 'anaconda-mode-find-assignments)
            (local-set-key (kbd "c-y") 'anaconda-mode-show-doc)
            (local-set-key (kbd "s-i") 'python-start-or-switch-repl)
            (local-set-key (kbd "s-n") 'python-shell-send-buffer)
            (local-set-key (kbd "s-N") 'python-shell-send-buffer-switch)
            (local-set-key (kbd "s-m") 'python-shell-send-defun)
            (local-set-key (kbd "s-M") 'python-shell-send-defun-switch)
            (local-set-key (kbd "s-r") 'python-shell-send-region)
            (local-set-key (kbd "s-R") 'python-shell-send-region-switch)
            (company-quickhelp-mode)
            ))

(add-hook 'python-mode-hook 'anaconda-mode)
(add-hook 'python-mode-hook 'eldoc-mode)

  ```
![python]({{"/assets/images/tools/python-note.png" | absolute_url}})


## go ##

  * 通过`brew install go`安装go编译环境
  * 安装[go layer](https://github.com/syl20bnr/spacemacs/tree/master/layers/%2Blang/go),官方有详细文档
  
  但是在安装一些go工具包(例如guru,gorename,goimports等)时，可能会遇到一点问题，即使我用了科学上网，依旧无法成功安装，但我我们可以把包下载了，自己手动安装，步骤如下
  
    * 确定你的`gopath`路径，例如我的是`/Users/pan/go`,将如下语句加入`~/.bashrc_profile`

    ``` shell
    export PATH=$PATH:/usr/local/opt/go/libexec/bin
    export GOPATH=/Users/pan/go
    PATH="$GOPATH/bin:$PATH"
    ```
    
      * git clone https://github.com/golang/tools.git tools
      * mkdir -p /Users/pan/go/golang.org/x/tools
      * cp -r tools/* /Users/pan/go/golang.org/x/tools
      * go install golang.org/x/tools/cmd/guru
      * go install  golang.org/x/tools/cmd/gorename
      * go install golang.org/x/tools/cmd/goimports

