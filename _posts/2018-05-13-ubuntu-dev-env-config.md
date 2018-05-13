---
title: ubuntu-dev-env-config 
date: 2018-05-13 16:05
categories:
- env
tags:
- gtk-doc
---

    >本篇将会汇集我ubuntu环境的配置，以便以后重装系统后能迅速恢复工作环境；ubuntu系统还是有写不稳定的地方，有些问题解决不了只能放大招
    
    
# glib安装 #

glib作为gone开发工具链成员，在嵌入式linux的项目中应用非常广泛；公司的单板管理软件几乎都以它的API库为基础开发；

今天安装的是[glib-2.56.0](http://ftp.gnome.org/pub/gnome/sources/glib/2.56/),为了能方便以后使用，当然希望能搞出API手册咯，以下为安装过程

``` shell
tar -xvf glib-2.56.0.tar.xz
cd glib-2.56.0
sudo ./configure --enable-libmount=no --enable-gtk-doc --enable-gtk-doc-html --enable-gtk-doc-pdf --enable-man
sudo make
sudo make install
```

但是`--enable-gtk-doc`参数要求环境必须要`gtk-doc>1.2`

### gtk-doc安装 ###

  * 首先要安装[itstool-2.0.3](http://itstool.org/download),根据官方提供的方法应该没什么问题；注意2.0.4的版本配合`gtk-1.28`有问题，安装过程会core-down
  
  ``` shell
sudo bzip2 -d itstool-2.0.3.tar.bz2
sudo tar -xvf itstool-2.0.3.tar
cd itstool-2.0.3/
sudo ./configure --prefix=/usr
sudo make
sudo make install
  ```

  * 再安装`dblatex`

``` shell
sudo apt-get install dblatex
```

  * 下载[gtk-doc-1.28](http://www.linuxfromscratch.org/blfs/view/svn/general/gtk-doc.html)

``` shell
sudo ./configure --prefix=/usr
make
make install
```

![configure-ok]({{"/assets/images/tools/gtk-doc-config.png" | absolute_url}})

![gtk-install-ok]({{"/assets/images/tools/gtk-install-ok.png" | absolute_url}})

