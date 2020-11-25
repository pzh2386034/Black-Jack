

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

  ``` bash
  [gerrit]
          basePath = git
          canonicalWebUrl = http://10.11.15.200:8081/
          serverId = 512185d4-d07e-441b-9874-3c7914c0ee2f
  [container]
          javaOptions = "-Dflogger.backend_factory=com.google.common.flogger.backend.log4j.Log4jBackendFactory#getInstance"
          javaOptions = "-Dflogger.logging_context=com.google.gerrit.server.logging.LoggingContext#getInstance"
          user = pan
          javaHome = /usr/lib/jvm/java-11-openjdk-amd64
  [index]
          type = lucene
  [auth]
          type = LDAP
  [ldap]
          server = ldap://10.11.15.200
          accountFullName = gerritSchool
          accountBase = dc=ldap,dc=com
          groupBase = cn=students,ou=school,dc=ldap,dc=com
          accountPattern = (&(objectClass=person)(cn=${username}))
          sslVerify = false
  [receive]
          enableSignedPush = false
  [sendemail]
          smtpServer = localhost
  [sshd]
          listenAddress = *:29418
  [httpd]
          listenUrl = http://*:8091/
  [cache]
          directory = cache
  
  ```

  

* 启动gerrit

```bash
sudo ./bin/gerrit.sh start 
```

## apach2

* 首先打开反向代理支持，否则后续配置文件会报语法错误

  ``` bash
  sudo a2enmod proxy
  sudo a2enmod proxy_ajp
  sudo a2enmod proxy_balancer
  cd /etc/apache2/mods-enable
  sudo ln -s ../mods-available/proxy.load
  sudo ln -s ../mods-available/proxy.conf
  sudo ln -s ../mods-available/proxy_http.load
  sudo ln -s ../mods-available/proxy_balancer.conf
  sudo ln -s ../mods-available/proxy_balancer.load
  sudo ln -s ../mods-available/rewrite.load
  sudo ln -s ../mods-available/ssl.conf
  sudo ln -s ../mods-available/ssl.load
  sudo ln -s ../mods-available/slotmem_shm.load
  sudo ln -s ../mods-available/socache_shmcb.load
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
  	#如果使用ldap鉴权，则下面这段去掉
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

## docker

* [Install and Uninstall](https://docs.docker.com/engine/install/ubuntu/)

  ``` bash
  sudo apt-get remove docker docker-engine docker.io containerd runc
  sudo apt-get update
  sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
  sudo apt-key fingerprint 0EBFCD88
  sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  sudo apt-get update
  sudo apt-get install docker-ce docker-ce-cli containerd.io
  apt-cache madison docker-ce
  sudo apt-get install docker-ce=<VERSION_STRING> docker-ce-cli=<VERSION_STRING> containerd.io
  sudo docker run hello-world
  #uninstall
  sudo apt-get purge docker-ce docker-ce-cli containerd.io
  sudo rm -rf /var/lib/docker
  ```

## [jenkins in docker](https://www.jenkins.io/doc/book/installing/docker/)

* 创建docker bridge

  ``` bash
  docker network create jenkins创建docker bridge
  ```

* 下载并执行docker:dind, 支持在jenkins node中执行docker命令

  ``` bash
  docker run --name jenkins-docker --rm --detach \
    --privileged --network jenkins --network-alias docker \
    --env DOCKER_TLS_CERTDIR=/certs \
    --volume jenkins-docker-certs:/certs/client \
    --volume jenkins-data:/var/jenkins_home \
    --publish 2376:2376 docker:dind
  ```

* 定制化offical jenkins docker image

  * 使用以下内容创建`Dockerfile`

    ``` bash
    FROM jenkins/jenkins:2.249.3-slim
    USER root
    RUN apt-get update && apt-get install -y apt-transport-https \
           ca-certificates curl gnupg2 \
           software-properties-common
    RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
    RUN apt-key fingerprint 0EBFCD88
    RUN add-apt-repository \
           "deb [arch=amd64] https://download.docker.com/linux/debian \
           $(lsb_release -cs) stable"
    RUN apt-get update && apt-get install -y docker-ce-cli
    USER jenkins
    RUN jenkins-plugin-cli --plugins blueocean:1.24.3
    ```

  * 从`Dockerfile`中创建一个新的docker image

    ``` bash
    docker build -t myjenkins-blueocean:1.1 .
    ```

    

* docker运行jenkins-blueocean

  ``` bash
  docker run --name jenkins-blueocean --rm --detach \
    --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
    --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
    --publish 8080:8080 --publish 50000:50000 \
    --volume jenkins-data:/var/jenkins_home \
    --volume jenkins-docker-certs:/certs/client:ro \
    myjenkins-blueocean:1.1
  ```

  