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
2. charles-proxy:ok

## restful auto test: httprunner
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


```bash
auto method = this->bus.new_method_call(
    "org.openbmc.control.Power", "/org/openbmc/control/power0",
    "org.freedesktop.DBus.Properties", "Get");

method.append("org.openbmc.control.Power", "pgood");

    "xyz.openbmc_project.State.Chassis",
    "/xyz/openbmc_project/state/chassis0",
    "org.freedesktop.DBus.Properties", "Get",
    "xyz.openbmc_project.State.Chassis", "CurrentPowerState");
```

​        

## kernel debug

~~~bash
* cat /proc/sys/kernel/printk 

``` bash
7 4 1 7
#console_loglevel、default_message_loglevel、minimum_c onsole_loglevel、default_console_loglevel
echo 8 4 1 7 > /proc/sys/kernel/printk #改变console_loglevel值，使得所有的打印消息都能输出到终端
```

.compatible = "aspeed,ast2500-spi"-->aspeed_smc_probe-->aspeed_smc_setup_flash-->aspeed_smc_chip_setup_init(aspeed_smc_chip_enable_write)-->add /dev/mtd*
-->spi_nor_scan(regist mtd read,write, erase)
~~~

mtd_device_parse_register:将mtd_info设备注册到mtd子系统中

dts pattitions parse function register: module_init(ofpart_parser_init);

mtd_device_parse_register()->parse_mtd_partitions()->mtd_part_of_parse()[register mtd0 image-bmc]->add_mtd_partitions[complete register different partitions]

​		

mtd_part_of_parse: get partitions from dts file.



## open kernel dev_dbg

```bash
CONFIG_DEBUG_FS=y
CONFIG_DYNAMIC_DEBUG=y
mount -t debugfs none /sys/kernel/debug                       #路径也可以自己选择，这里用系统默认路径        
echo -n 'file xxx.c +p' > /data/debugfs/dynamic_debug/control  #增加xxx.c文件dynamic debug的输出
echo -n 'file xxx.c -p' > /data/debugfs/dynamic_debug/control  #去掉xxx.c文件dynamic debug的输出
CONFIG_CONSOLE_LOGLEVEL_DEFAULT=10


cat /etc/mtab
umount -v debugfs             通过设备名卸载 
umount -v /sys/kernel/debug         通过挂载点卸载

echo 0 > /sys/class/gpio/gpio427/value
echo 1e630000.spi >/sys/bus/platform/drivers/aspeed-smc/bind
```

## udev

``` bash
udevadm test /sys/class/tty/ttyS5
```




## bind kernel stack

```shell
[  147.452193] [<80107f9c>] (dump_backtrace) from [<80108220>] (show_stack+0x20/0x24)
[  147.459888]  r7:00000000 r6:00000000 r5:00000000 r4:00000009
[  147.465610] [<80108200>] (show_stack) from [<806c6e50>] (dump_stack+0x20/0x28)
[  147.472976] [<806c6e30>] (dump_stack) from [<801178bc>] (__warn.part.0+0xb8/0xe0)
[  147.480566] [<80117804>] (__warn.part.0) from [<80117a6c>] (warn_slowpath_null+0x50/0x5c)
[  147.488843]  r6:802c4aa4 r5:00000147 r4:807f63b0
[  147.493504] [<80117a1c>] (warn_slowpath_null) from [<802c4aa4>] (sysfs_create_file_ns+0x64/0xc8)
[  147.502402]  r6:94c5d548 r5:00000001 r4:80a357b8
[  147.507135] [<802c4a40>] (sysfs_create_file_ns) from [<8049a4b4>] (spi_nor_scan+0x6ec/0xa60)
[  147.515584]  r7:94c5d6b0 r6:9e111e10 r5:80747a58 r4:94c5d448
[  147.521362] [<80499dc8>] (spi_nor_scan) from [<8049d734>] (aspeed_smc_probe+0x5ac/0x8f4)
[  147.529576]  r10:94c5d420 r9:9e111e10 r8:9ecbc9d0 r7:00030600 r6:94c5d448 r5:8d956bc0
[  147.537493]  r4:8d956bc0
[  147.540083] [<8049d188>] (aspeed_smc_probe) from [<8046d458>] (platform_drv_probe+0x44/0x80)
[  147.548625]  r10:00000004 r9:80a357e4 r8:00000000 r7:80a80498 r6:00000000 r5:80a357e4
[  147.556545]  r4:9e111e10
[  147.559118] [<8046d414>] (platform_drv_probe) from [<8046b200>] (really_probe+0x104/0x3dc)
[  147.567475]  r5:80a80494 r4:9e111e10
[  147.571088] [<8046b0fc>] (really_probe) from [<8046b7bc>] (driver_probe_device+0x130/0x170)
[  147.579552]  r10:00000051 r9:8da9df60 r8:019a86a0 r7:0000000d r6:80a357e4 r5:9e111e10
[  147.587464]  r4:80a357e4
[  147.590037] [<8046b68c>] (driver_probe_device) from [<8046ba8c>] (device_driver_attach+0x68/0x70)
[  147.599006]  r9:8da9df60 r8:019a86a0 r7:0000000d r6:80a357e4 r5:00000000 r4:9e111e10
[  147.606858] [<8046ba24>] (device_driver_attach) from [<80469888>] (bind_store+0x98/0x10c)
[  147.615053]  r7:0000000d r6:80a33f48 r5:80a357e4 r4:9e111e10
[  147.620821] [<804697f0>] (bind_store) from [<80468cc8>] (drv_attr_store+0x30/0x3c)
[  147.628482]  r7:8d956a20 r6:8d956a30 r5:00000000 r4:804697f0
[  147.634178] [<80468c98>] (drv_attr_store) from [<802c42f0>] (sysfs_kf_write+0x48/0x54)
[  147.642179]  r5:00000000 r4:80468c98
[  147.645785] [<802c42a8>] (sysfs_kf_write) from [<802c3818>] (kernfs_fop_write+0x118/0x204)
[  147.654136]  r5:00000000 r4:00000000
[  147.657817] [<802c3700>] (kernfs_fop_write) from [<8024ba50>] (__vfs_write+0x4c/0x1d8)
[  147.665752]  r10:00000000 r9:00000000 r8:00000000 r7:802c3700 r6:0000000d r5:8da9df60
[  147.673671]  r4:9e36e9a0
[  147.676298] [<8024ba04>] (__vfs_write) from [<8024e538>] (vfs_write+0xb0/0x194)
[  147.683712]  r9:00000000 r8:00000000 r7:8da9df60 r6:019a86a0 r5:9e36e9a0 r4:0000000d
[  147.691561] [<8024e488>] (vfs_write) from [<8024e7f4>] (ksys_write+0x70/0xf8)
[  147.698795]  r8:00000000 r7:0000000d r6:019a86a0 r5:9e36e9a0 r4:9e36e9a0
[  147.705530] [<8024e784>] (ksys_write) from [<8024e894>] (sys_write+0x18/0x1c)
[  147.712765]  r9:8da9c000 r8:801011e4 r7:00000004 r6:46661098 r5:019a86a0 r4:0000000d
[  147.720607] [<8024e87c>] (sys_write) from [<80101000>] (ret_fast_syscall+0x0/0x54)
[  147.728269] Exception stack(0x8da9dfa8 to 0x8da9dff0)

```
