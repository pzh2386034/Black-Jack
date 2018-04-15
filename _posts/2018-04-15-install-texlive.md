---
title: install texlive 2017
date: 2018-04-15 15:04
categories:
- tools
tags:
- texlive
---

好久没有用`latex`编辑文档了，硕士第一次接触这个工具，惊叹于它能排版出完美的数学文档；毕业后，数学公式都快不认识了，排版工具当然也是长时间没有使用，再加在linux环境下，`latex`的编辑工具也一直没有理顺，中间有一段时间用过，特别拗，效率不高就搁置了；这次重新整合到`spacemacs`中，重新用起来。

## 安装 ##

### ubuntu自动安装 ###

  * `sudo apt-get install texlive-full`
  
  在ubuntu14.04LTS平台下，安装完成后显示版本信息是texlive2013, 因此卸载
  
  * 彻底卸载方法
  
  
``` shell
sudo apt-get purge texlive*
sudo apt-get remove tex-common --purge
rm -rf /usr/local/texlive/2012 
rm -rf /usr/local/share/texmf
rm -rf /var/lib/texmf
rm -rf /etc/texmf
```

### 手动安装 ###

  * 下载安装包[texlive_2017.iso](http://mirrors.ustc.edu.cn/CTAN/systems/texlive/Images/texlive2017.iso)
  * 挂载镜像文件并执行安装脚本
  
``` shell
sudo mount -o loop texlive2017.iso /mnt
cd /mnt
sudo ./install-tl
# 输入I进入自动化安装
```

  * 安装成功
![install success]({{"/assets/images/tools/latex.png" | absolute_url}})

  * 配置启动脚本(/etc/bash.bashrc)

![bash.bashrc]({{"/assets/images/tools/bashrc.png" | absolute_url}})

  * 配置启动脚本(/etc/manpath.config)

![manpath.config]({{"/assets/images/tools/manpath.png" | absolute_url}})

  * 重启linux

## 测试 ##

### 查询版本信息 ###

![latex version]({{"/assets/images/tools/latex-version.png" | absolute_url}})

### 使用帮助文档 ###

``` shell
man bibtex
```
![bibtex]({{"/assets/images/tools/bibtex.png" | absolute_url}})

### 参考链接 ###

[ubuntu home](https://askubuntu.com/questions/60765/how-do-i-add-a-directory-to-manpath-or-infopath)


