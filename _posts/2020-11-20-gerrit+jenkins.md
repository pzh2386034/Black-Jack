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

systemctl status slapd|ca

/usr/share/slapd/

## gerrit

```bash
java -jar gerrit-2.15.14.war init
./bin/gerrit.sh start
```