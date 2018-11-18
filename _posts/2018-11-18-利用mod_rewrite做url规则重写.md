---
title: 利用mod_rewrite做url规则重写
date: 2018-11-18 22:11
categories:
- web
tags:
- mod\_rewrite
---

>利用mod\_rewrite做url规则重写,使得制作的网站url更简介明了

## apache配置 ##

* 在conf目录的httpd.conf文件中找到

``` shell
利用mod_rewrite做url规则重写.md
```

* 在要支持url rewrite的目录启用配置

通过如下配置使www目录支持url rewrite.

``` shell
<Directory "/usr/local/var/www">
　　　　Options Indexes FollowSymLinks
　　　　AllowOverride All
　　　　Order allow,deny
　　　　Allow from all
Directory>
```

* 在www根目录下添加.htaccess文件，说明rewrite规则

例如我在使用codeigniter框架，想从url中移除index.php，则可以如下配置

``` shell
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.*)$ index.php/$1 [L]
</IfModule>
```

## CodeIgniter配置 ##

如果使用codeigniter框架，则还需如下配置

* 修改application/config目录下的`config.php`

``` php
//  Find the below code

$config['index_page'] = "index.php"

//  Remove index.php

$config['index_page'] = ""
```

* 修改application/config目录下的`config.php`

``` php
//  Find the below code

$config['uri_protocol'] = "AUTO"

//  Replace it as

$config['uri_protocol'] = "REQUEST_URI" 
```

## 附录 ##

简单举例说明`.htaccess`文件如何编写

``` shell
RewriteEngine on # 开启rewrite
RewriteCond $1 !(index.php|images) #如果文件不为index.php或目录不为images
RewriteRule ^(.*)$ /index.php?page=$1 [L] 
```

* `$1`代表引用RewriteRule中的第一个正则(.*)代表的字符
* 结尾标识。停止重写操作，并不再应用其他重写规则。防止本条规则被后续规则影响
* 当访问dmyz.org/about时，实际是访问dmyz.org/index.php?page=about

``` shell
RewriteEngine on
RewriteCond %{REQUEST_FILENAME} !-f
RewriteRule (.*) /.htm
```

* -f用来检测当前值所代表的路径是否是一个常规文件
* 当访问的文件不是一个常规文件时，转到404.htm页面

## FAQ ##

* 当配置完rewrite后，可能会显示Forbidden  You don't have permission to access \*\*\*\*\*\*, 在`httpd.conf`中加上`Options FollowSymLinks`即可，注意不能加`Indexes`,这个表示显示站点目录

