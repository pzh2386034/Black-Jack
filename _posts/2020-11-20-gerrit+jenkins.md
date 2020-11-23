## install mysql

apt install mysql-server-5.7

## install gerrit

wget https://gerrit-releases.storage.googleapis.com/gerrit-3.2.5.1.war

## install jdk

```bash
sudo add-apt-repository ppa:ts.sch.gr/ppa
sudo apt-get update
sudo apt-get install oracle-java8-installer
```

``` bash
sudo apt install openjdk-8-jdk
```



## openldap

* sudo apt-get update
* sudo apt-get upgrade
* sudo apt install slapd ldap-utils
* sudo dpkg-reconfigure slapd(修改默认配置，可选，可配置cn=admin,dc=school,dc=com)

#### phpldapadmin

```bash
sudo apt remove --purge phpldapadmin
sudo apt install phpldapadmin#(/etc/apache2/ports.conf)
```

* 下载最新phpldapadmin替换

systemctl status slapd

/usr/share/slapd/

## gerrit

* 初始化配置全enter，使用默认配置即可

  ``` bash
  java -jar gerrit-2.15.14.war init -d gerrit
  ```

* 修改配置参数`canonicalWebUrl`, `listenUrl`

* 启动gerrit

```bash
./bin/gerrit.sh start 
```

## apach2

* 首先打开反向代理支持，否则后续配置文件会报语法错误

  ``` bash
  sudo a2enmod proxy
  sudo a2enmod proxy_ajp
  sudo a2enmod proxy_balancer
  ```

  

* 新增VirtualHost(*记得去掉注释*):  `/etc/apache2/sites-available/gerrit.conf`

  ``` bash
  <VirtualHost *:8081>  //这里是反射代理的端口号，
  
      ServerName 10.11.15.200    //这里是填写Apache反射代理的ip地址，也就是你服务器的ip地址
  
      ProxyRequests Off
      ProxyVia Off
      ProxyPreserveHost On
  
      ErrorLog ${APACHE_LOG_DIR}/error.log
      CustomLog ${APACHE_LOG_DIR}/access.log combined
      <Proxy *>
            Order deny,allow
            Allow from all
      </Proxy>
  
      <Location "/login/">
          AuthType Basic
          AuthName "Gerrit Code Review"
          Require valid-user
          AuthBasicProvider file
          AuthUserFile /home/gerrit/review_site/passwords    //这个路径是gerrit账户密码管理，后续的步骤中会创建此文件。路径有写正确
      </Location>
  
      AllowEncodedSlashes On
  
      ProxyPass / http://127.0.0.1:8091/ nocanon  //这里是代理反射，照着写就OK了
  
  </VirtualHost>
  
  ```

  

* 创建软链接

  ``` bash
  sudo ln -s /etc/apache2/site-available/gerrit.conf /etc/apache2/site-enable/gerrit.conf
  ```

  

* 重启apach2

  ``` bash
  sudo systemctl restart apache2.service
  ```

  

* 新增Apache2监听端口号: `/etc/apache2/ports.conf`

  ``` bash
  Listen 8081
  ```

  *