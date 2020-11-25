{"data":"xyz.openbmc_project.Software.Activation.RequestedActivations.Active"}
/redfish/v1/AccountService/Accounts
/xyz/openbmc_project/software/enumerate


busctl call  xyz.openbmc_project.Settings /xyz/openbmc_project/time/sync_method org.freedesktop.DBus.Properties Get ss xyz.openbmc_project.Time.Synchronization TimeSyncMethod |cat
busctl call xyz.openbmc_project.User.Manager /xyz/openbmc_project/user xyz.openbmc_project.User.Manager CreateUser sassb pan 4 ipmi redfish ssh web priv-admin true|cat
busctl call xyz.openbmc_project.Network  /xyz/openbmc_project/network/config org.freedesktop.DBus.Properties Set ssv \
            xyz.openbmc_project.Network.SystemConfiguration HostName s zehua
busctl call  xyz.openbmc_project.Settings /xyz/openbmc_project/time/sync_method org.freedesktop.DBus.Properties Set ssv \
            xyz.openbmc_project.Time.Synchronization TimeSyncMethod s "xyz.openbmc_project.Time.Synchronization.Method.NTP"


busctl call xyz.openbmc_project.Ldap.Config /xyz/openbmc_project/user/ldap xyz.openbmc_project.User.Ldap.Create CreateConfig ssssssss     ldap://192.168.100.5 cn=panzehua,ou=users,dc=ldap,dc=com dc=ldap,dc=com 123456 xyz.openbmc_project.User.Ldap.Create.SearchScope.sub    xyz.openbmc_project.User.Ldap.Create.Type.OpenLdap   aa bb

busctl call xyz.openbmc_project.LDAP.PrivilegeMapper /xyz/openbmc_project/user/ldap xyz.openbmc_project.User.PrivilegeMapper Create ss student  priv-admin


busctl call xyz.openbmc_project.LDAP.PrivilegeMapper /xyz/openbmc_project/user/ldap xyz.openbmc_project.User.PrivilegeMapper Create ss 


curl -c cjar -b cjar -k -H "Content-Type: application/json" -X POST -d '{"data":[false,"ldap://192.168.100.5", "cn=panzehua,ou=users,dc=ldap,dc=com", "dc=ldap,dc=com","123456","0","1"]}''  https://192.168.100.153/xyz/openbmc_project/user/ldap/action/CreateConfig


./testldap -H 192.168.100.5 -b  dc=ldap,dc=com  -l cn=panzehua,ou=users,dc=ldap,dc=com -p 123456


 busctl  introspect xyz.openbmc_project.Time.Manager /xyz/openbmc_project/time/bmc|cat
 busctl call xyz.openbmc_project.Time.Manager /xyz/openbmc_project/time/bmc org.freedesktop.DBus.Properties GetAll s xyz.openbmc_project.Time.EpochTime





openldap服务器：
LDAP地址：192.168.2.194，用户anna在anna组下面
anna的dn：uid=anna,ou=People,dc=example,dc=com
填的数据：
uri:ldap://192.168.2.194
binddn:uid=anna,ou=People,dc=example,dc=com
密码就是anna的密码
basedn:dc=example,dc=com
groupattr:gidNumber
userattr:uid
创建的组：anna

windowsAD：
LDAP地址：192.168.2.28，用户配置的primary group是Domain Users
anna的dn：cn=testyu,cn=Users,dc=test,dc=com
填的数据：
uri:ldap://192.168.2.28
binddn:cn=testyu,cn=Users,dc=test,dc=com
密码就是testyu的密码
basedn:dc=test,dc=com
groupattr:primartGroupID这个是默认的，可以改
userattr:cn
创建的组：Domain Users


* LDAP

netstat -anp |grep 3466 #进程号查端口号
git commit --amend --reset-author

* 网卡启动流程

1. 生成链路本地地址
2. 生成"被请求节点多播地址"
3. 多播成员报告
4. 重复地址检测
5. 无状态地址自动分配

# 自动化测试

## 抓包工具

1. Postman:未能抓到包
2. charles-proxy

## restful: httprunner
1. pip3 install httprunner
2. httprunner -V
in=$( base64 <<< "123456") 
pwd=$( base64 -d <<< MQo= )
升级
pip3 install -U httprunner
pip3 install -U git+https://github.com/httprunner/httprunner.git@master

## 管理框架: httprunnerManager

## 生成报告
1. html
hrun /path/to/testcase --html=report.html --self-contained-html  
2. allure(可以在Jenkins中安装allure插件支持)
pip3 install "httprunner[allure]"
下载 https://github.com/allure-framework/allure2/releases/download/2.7.0/allure-2.7.0.zip
hrun /path/to/testcase --alluredir=/tmp/my_allure_results
sudo ln -s /mnt/disk1/softwares/allure-2.7.0/bin/allure /usr/bin/allure
allure serve /tmp/my_allure_results
## web:selenium

localhost, 127.0.0.0/8, ::1


1. 2020/6/1--2020/6/30，参与海康定制BMC功能开发
	主要涉及KVM安全登入定制化
2. 2020/7/1--2020/9/30，参与BMC主线功能开发，主要如下：
	a. 对标华为，进行restful接口竞争力分析，并指定功能补齐路标
	b. 开发restful接口，涉及manager模块13个rest接口；
	c. 开发特性涉及LDAP；用户登入OEM鉴权；最大会话数，web会话超时时间等
3. 2020/10/1--Now，负责601项目(百度，阿里，国电通)维护及新需求开发
	a. 百度DHCPV6功能开发
	b. 国电通紧急需求用户oem鉴权登入，web会话超时时间设置，密码安全校验规则等
	c. 阿里解决bug 
	d.redfish使用手册开发


    auto method = this->bus.new_method_call(
        "org.openbmc.control.Power", "/org/openbmc/control/power0",
        "org.freedesktop.DBus.Properties", "Get");

    method.append("org.openbmc.control.Power", "pgood");

        "xyz.openbmc_project.State.Chassis",
        "/xyz/openbmc_project/state/chassis0",
        "org.freedesktop.DBus.Properties", "Get",
        "xyz.openbmc_project.State.Chassis", "CurrentPowerState");