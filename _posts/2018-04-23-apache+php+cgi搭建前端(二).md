---
title: apache+php+cgi搭建前端(二).md
date: 2018-04-23 01:04
categories:
- spacemacs
tags:
- spacemacs
---

接着上一篇，搞完了`httpd+php`,接下来就要通过`php`的`cgi`接口调用C函数，例子还是在我的[git仓库](https://github.com/pzh2386034/Black-Jack)

## 编译测试动态库 ##

``` c++
/*hello.c*/
int cc_add(int a, int b)
{
    return a +b;
}
int cc_mul(int a, int b)
{
    return a*b;
}
```

``` c++
/*hello.h*/
int cc_add(int a, int b);
int cc_mul(int a, int b);
```

``` shell
gcc -fPIC -g -c hello.c -o hello.o
gcc -shared hello.o -o libweb_so.so
cp libweb_so.so /usr/local/lib
```

## 编译php扩展模块 ##

预设变量
  * PHP5=php-5.6.32
  * X86_INSTALL="/home/pan/apache/x86"
  * SO\_NAME=web\_so

``` shell
tar xzf php-5.6.32.tar.gz
cd ${PHP7}/ext|| exit 1
./ext_skel --extname=${SO_NAME}|| exit 1
cd ${SO_NAME}|| exit 1
```

### 编辑config.m4文件 ###

``` shell
# 将以下代码去注释；注意是enable函数，后面编译参数是要对应起来的
PHP_ARG_ENABLE(web_so, whether to enable web_so support,
Make sure that the comment is aligned:
[  --enable-web_so           Enable web_so support])
```

### 生成configure脚本 ###

``` shell
    ${X86_INSTALL}/${PHP5}/bin/phpize|| exit 1
```

### 修改php.ini文件 ###

``` shell
# 设置允许加载动态库
enable_dl = On
# 设置加载动态的名称, 如果动态库不在标准的modules目录下，则需要完整路径(未经测试)
extension=web_so.so
```

### 编辑扩展c文件 ###

``` c++
/*编辑php_web_so.h*/
PHP_FUNCTION(confirm_web_so_compiled);
PHP_FUNCTION(hello_add);
```

``` c++
/*编辑web_so.c, 增加如下代码*/
PHP_FUNCTION(confirm_web_so_compiled)
{
	char *arg = NULL;
	int arg_len, len;
	char *strg;

	if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "s", &arg, &arg_len) == FAILURE) {
		return;
	}

	len = spprintf(&strg, 0, "Congratulations! You have successfully modified ext/%.78s/config.m4. Module %.78s is now compiled into PHP.", "web_so", arg);
	RETURN_STRINGL(strg, len, 0);
}
PHP_FUNCTION(hello_add)
{
    long int a, b;
    long int result;
    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "ll", &a, &b) == FAILURE) {
        return;
}

/*在zend_function_entry字符串数组中增加元素*/
const zend_function_entry web_so_functions[] = {
	PHP_FE(confirm_web_so_compiled,	NULL)		/* For testing, remove later. */
	PHP_FE(hello_add, NULL)
	PHP_FE_END	/* Must be the last line in web_so_functions[] */
};
```

### 编译 ###

``` shell
#  php-config文件路径需自行确定，在php安装目录的/bin下
./configure --enable-web_so --with-libdir=/usr/local/lib/libweb_so.so --with-php-config=/home/pan/apache/x86/php-5.6.32/bin/php-config
make LDFLAGS=-lweb_so
# 将编译出的扩展so拷贝至httpd的modules目录
make install
cp modules/web_so.so /home/pan/apache/x86/httpd-2.4.29/modules/
```

重启apache httpd

``` shell
sudo /etc/init.d/httpd restart
```

## 结果验证 ##

  * 启动httpd

  * 编辑测试`php`文件, 放入htdocs目录
  
  ``` php
<?php
echo confirm_web_so_compiled("panzehua");
echo "<br/>";
echo hello_add(2, 4);
?>
  ```
  
  * 显示如下结果即功能OK
  
  ![php test result]({{"/assets/images/tools/phptest.png" | absolute_url}})

  

## 常见问题排查 ##

如果测试页面报错，要分以下情况


  - 如果以下文字显示在测试页面中，则说明**自行编译的动态有问**，httpd扩展动态库`web_so.so`无法链接上`libhello.so`

    >Congratulations! You have successfully modified ext/web_so/config.m4. Module panzehua is now compiled into PHP.
    
    该情况下，`phpinfo()`函数的应该能找到如下信息
    
   ![web so info]({{"/assets/images/tools/webso.png" | absolute_url}})
   

  - 如果没有显示，则说明`web_so.so`扩展库安装有问题，则在`phpinfo()`中应该查不到`web_so.so`信息，排查如下几点

    * 确定编译出来得web_so.so链接是否正常，使用ldd查看
    
    ![ldd check web so]({{"/assets/images/tools/ldd-webso.png" | absolute_url}})
    
    > 该问题注意编译阶段的configure参数**--with-libdir**

    
    * 如果apache错误提示找不到函数则要看/usr/local/lib目录下链接的动态库是否存在，以及是否是动态库
   ![ldd check hello so]({{"/assets/images/tools/ldd-helloso.png" | absolute_url}})
   
   > 该问题应该是编译`libhello.so`有问题
   
### 其余常见问题 ###

  * httpd无法启动，一般是httpd.conf中加载的模块有问题
  * httpd启动后，web无法访问自己虚拟的IP；请用`ifconfig`命令确定IP是否存在
  * 如果在php.ini中加载了extension,但是实际测试没有加载，很可能是编译extension使用的phpsize, php-config工具和libphp5.so不一致；这种情况在httpd error_log文件中会有错误日志
  * 使用phpsize和php-config工具，最好指定具体php版本的绝对路径，避免环境有多个php版本影响
  * 如果phpinfo()能正常现实extension信息，但是使用extension中提供的PHP_FUNCION,却报404错误，一半是core-dump了，这时候观察下error\_log,每一次刷新都会多一条日志
