---
title: http+php+cgi搭建前端(一)
date: 2018-04-21 01:04
categories:
- spacemacs
tags:
- spacemacs
---

项目组使用apache http+php+cgi+rpc搭建了一套数据访问、查询、设置的管理软件；在一整个机框中，作为管理板，负责管理最多32个计算节点，4个交换。现学现卖，自己也搞个http+php+cgi+rpc，做个什么数据查询，访问，存储框架估计也挺有用；因此打算详细记录下搭建过程，并附上一个[github仓库](https://github.com/pzh2386034/private_web), 本篇设计的脚本在仓库的以下路径`thirdpart/httpd/build.sh`.

# 安装依赖软件，按顺利安装 #

测试机环境：

    >ubuntu 14.04TLS

软件列表
  * [apr-1.6.3](http://mirrors.tuna.tsinghua.edu.cn/apache//apr/apr-1.6.3.tar.gz)
  * [apr-iconv-1.2.2](http://mirror.bit.edu.cn/apache//apr/apr-iconv-1.2.2.tar.gz)
  * [apr-util-1.6.1](http://mirror.bit.edu.cn/apache//apr/apr-util-1.6.1.tar.gz)
  * [pcre-8.41](http://ftp.exim.llorien.org/pcre/pcre-8.41.tar.gz)
  * [httpd-2.4.29](http://mirrors.tuna.tsinghua.edu.cn/apache//httpd/httpd-2.4.29.tar.gz)
  * [libxml2-2.9.6](http://xmlsoft.org/sources/libxml2-sources-2.9.6.tar.gz)
  * php-5.6.32
  
  前置环境变量
  
  ``` shell
X86_INSTALL="/home/pan/apache/x86"
ARM_INSTALL="/home/pan/apache/arm"
  ```

## 安装apr套件 ##

(apache portable Run-time libraries)主要为上层应用程序提供一个可跨越多操作系统平台使用的底层支持接口库,包含3个开发包(apr, apr-iconv, apr-util)

### apr ###

``` shell
    ./configure --prefix=${X86_INSTALL}/${APR} || exit 1
    make || exit 1
    make install || exit 1
```

### apr-iconv ###

``` shell
    ./configure --prefix=${X86_INSTALL}/${APR_ICONV} --with-apr=${X86_INSTALL}/${APR} || exit 1
    make || exit 1
    make install || exit 1
```

### apr-util ###

``` shell
    ./configure --prefix=${X86_INSTALL}/${APR_UTIL} --with-apr=${X86_INSTALL}/${APR} || exit 1
    make || exit 1
    make install || exit 1
```

## 安装pcre ##

(Perl Compatible Regular Expressions),perl库，包含perl兼容的正则表达式
``` shell
    ./configure --prefix=${X86_INSTALL}/${PCRE} --with-apr=${X86_INSTALL}/${APR} || exit 1
    make || exit 1
    make install || exit 1
```

## 安装httpd ##

``` shell
    ./configure --prefix=${X86_INSTALL}/${HTTPD} --with-pcre=${X86_INSTALL}/${PCRE} --with-apr=${X86_INSTALL}/${APR} --with-apr-util=${X86_INSTALL}/${APR_UTIL} || exit 1
    make || exit 1
    make install || exit 1
```

## 安装libxml ##

``` shell
    ./configure --prefix=${X86_INSTALL}/libxml2 || exit 1
    make || exit 1
    make install || exit 1
```

## 安装php5 ##

``` shell
    ./configure  --prefix=${X86_INSTALL}/${PHP5} --with-apxs2=${X86_INSTALL}/${HTTPD}/bin/apxs --with-mysql=/usr --includedir=${X86_INSTALL}/${HTTPD}/include --with-config-file-path=${X86_INSTALL}/${HTTPD}/conf/extra --without-pear --disable-inline-optimization --enable-zend-multibyte=no --disable-ipv6 --without-sqlite3 --enable-dba=no --without-pdo-sqlit --enable-opcache=no --disable-pdo --disable-FEATURE --disable-rpath --enable-ftp=no --with-xmlrpc --with-libxml-dir=${X86_INSTALL}/libxml2 --enable-libxml --enable-maintainer-zts --enable-dom --enable-simplexml --enable-xml --enable-xmlreader --enable-xmlwriter --enable-so || exit 1
    #sudo apt install python-dev
    make || exit 1
    #如果不使用root用户，install会异常，但是并不影响使用
    make install
    echo "install ${PHP5} X86 success"
    cp php.ini-development ${X86_INSTALL}/${HTTPD}/conf/extra/php.ini
    cp .libs/libphp5.so ${X86_INSTALL}/${HTTPD}/modules
```
mark:
  * 需要首先安装python-dev包
  如果是`ubuntu`,使用

  ``` shell
  sudo apt install python-dev
  ```

## 配置文件 ##

  * 配置`apache httpd`启动自动加载`libphp5.so`库
  * 配置特定格式文件要经过php解析，使其可以认识php语法
  
``` shell
##  配置apache httpd启动自动加载libphp5.so库
    cat >> ${X86_INSTALL}/${HTTPD}/conf/httpd.conf <<EOF
LoadModule php5_module modules/libphp5.so
<FilesMatch ".+\.p?h(p[3457]?|t|tml)$">
    SetHandler application/x-httpd-php
</FilesMatch>
EOF
```

  * 如果笔记本使用无线网络，则最好添加一张虚拟网卡，把http的IP映射上去

  ``` shell
    #sudo ifconfig eth0:0 192.168.100.10 up
    sudo cat >> /etc/network/interfaces <<EOF
auto eth0:0
iface eth0:0 inet static
address 192.168.100.10
netmask 255.255.255.0
#network 192.168.10.1
#broadcast 192.168.1.255
EOF
    sudo /etc/init.d/networking restart
    sed -n '/^Liston 80/p' ${X86_INSTALL}/${HTTPD}/conf/http.conf | sed 's/80/192.168.100.10:80/g'
  ```

## 使用 ##

  * 启动`httpd`

  ``` shell
      sudo /etc/init.d/httpd start
  ```
  * 在`${X86_INSTALL}/${HTTPD}/htdocs`创建`test.php`,内容如下

  ``` php
  <?php
  phpinfo();
  ?>
  ```

  * 在浏览器地址栏输入`192.168.100.10/test.php`,有如下结果
  
  ![php版本信息]({{"/assets/images/tools/phpinfo.png" | absolute_url}})

  * 通过cgi接口调用C动态库见下一篇
