# openbmc bmcweb debug

>本篇主要想说明清楚，如何调试一个openbmc模块，分为4个部分，以bmcweb(openbmc的https请求入口)为例；当然更为全局性的SDK使用，bitbake新增模块,dbus接口增删改，systemd等不会在本文中介绍

## 工具准备

* [openbmc 编译环境](https://iwiki.woa.com/pages/viewpage.action?pageId=537127652)

* docker or qemu-system-arm (用于qemu虚拟openbmc)


## 主要内容

* bmcweb模块修改源码，编译，打开debug日志，编译环境变量确认

* 替换进程测试修改内容

* 使用qemu在编译机中虚拟openbmc，查看效果

* 刷新单板固件，确认修改

## bmcweb 新增redfish接口(查询BmcName)

### 在编译自带的git目录修改源码

* 加载 bitbake 环境工具

```bash
cd openbmc
. setup starlake
```

* 进入bmcweb源码路径

``` bash
## bmcweb源码路径
cd /you/openbmc/dir/build/starlake/tmp/work/arm1176jzs-openbmc-linux-gnueabi/bmcweb/1.0+gitAUTOINC+ab41ea10f7-r0/git
```

* 打开bmcweb模块调试日志开关

```bash
echo "    add_definitions (-DBMCWEB_ENABLE_LOGGING)
    add_definitions (-DBMCWEB_ENABLE_DEBUG)" >>CMakeLists.txt
```

* 增加一个redfish查询接口(`/redfish/v1/Managers/bmc/info`)

>在bmcweb中，每一个redfish url都是一个class，类中定义了不同操作的方法、权限；例如我们要在`/redfish/v1/Managers/bmc`这颗资源树下新增资源链接，首先要找到父节点，并在其中加入子节点信息

* `redfish-core/lib/Managers.cpp`中，`Class Managers`类的`doGet`方法加入如下信息

```c++
res.jsonValue["somethinginfo"] = {
        {"@odata.id", "/redfish/v1/Managers/bmc/info"}};
```

* 在`redfish-core/lib/Managers.cpp`中，类似`class ManagerCollection`，新增类

``` c++
class Something : public Node
{
  public:
    Something(CrowApp& app) : Node(app, "/redfish/v1/Managers/bmc/info")
    {
        entityPrivileges = {
            {boost::beast::http::verb::get, {{"Login"}}},
            {boost::beast::http::verb::head, {{"Login"}}},
            {boost::beast::http::verb::patch, {{"ConfigureManager"}}},
            {boost::beast::http::verb::put, {{"ConfigureManager"}}},
            {boost::beast::http::verb::delete_, {{"ConfigureManager"}}},
            {boost::beast::http::verb::post, {{"ConfigureManager"}}}};
    }

  private:
    void doGet(crow::Response& res, const crow::Request& req,
               const std::vector<std::string>& params) override
    {
        // Collections don't include the static data added by SubRoute
        // because it has a duplicate entry for members
        res.jsonValue["@odata.id"] = "/redfish/v1/Managers/info";
        res.jsonValue["@odata.type"] = "#ManagerInfo.ManagerInfo";
        res.jsonValue["Name"] = "Manager Info Get";
        
        res.jsonValue["BmcName"] = "Starlake";
        res.end();
    }
};
```

* 最后在`redfish-core/include/redfish.hpp`中，将新类`class Something`通过父类的注册到路由中

``` c++ 
nodes.emplace_back(std::make_unique<KvmServer>(app));
```

### 代码编译

>到目前，简单的新增redfish GET接口就完成，下面就是编译;

* 在加载bitbake环境的shell(. setup starlake)中执行以下命令

``` bash
bitbake -c compile -f bmcweb
bitbake -c package -f bmcweb
```

### 生效最新的bmcweb

>有3种不同的方法生效，其中qemu虚拟bmc，由于没法获取到底层的硬件信息，只能调试纯软件功能，使用不多

#### 替换进程调试

>在`build/starlake/tmp/work/arm1176jzs-openbmc-linux-gnueabi/bmcweb/1.0+gitAUTOINC+ab41ea10f7-r0`下的`packages-split/bmcweb`目录；再执行完`bitbake -c package -f bmcweb`命令后，会刷新最新的进程文件，将bmcweb进程替换单板上的进程

``` bash
scp packages-split/bmcweb/usr/bin/bmcweb bmc:~
### 下面这条命令在单板执行
root@starlake:~# mv bmcweb /usr/bin/bmcweb
root@starlake:~# systemctl restart bmcweb
```

#### qemu虚拟bmc

>待更新..........

#### 单板刷新固件

>在单板的`/run/initramfs`路径,将`image-bmc`文件拷贝至该目录，再`reboot`重启；会自动刷新固件

```bash
bitbake obmc-phosphor-image
scp build/starlake/tmp/deploy/images/starlake/image-bmc bmc:/run/initramfs
### 下面这条命令在单板执行
root@starlake:~# reboot
### 接下来你会看见固件刷新的串口打印
```

## 调用redfish接口，并观察日志

* 打开系统日志

```bash
root@starlake:~# journalctl -f|grep -i bmcweb
```

* 在本机通过curl查询新增的redfish接口

```bash
curl --insecure -u root:0penBmc -X GET https://192.168.1.3/redfish/v1/Managers/bmc/info
## 以下为输入内容
{
  "@odata.id": "/redfish/v1/Managers/info",
  "@odata.type": "#ManagerInfo.ManagerInfo",
  "BmcName": "Starlake",
  "Name": "Manager Info Get"
}%
```

* 分析https请求解析函数栈

>回到单板的shell，发现journal日志已经打印出https整个解析流程关键步骤

```bash
....
Jan 01 00:07:08 starlake bmcweb[394]: (1970-01-01 00:07:08) [ERROR "http_connection.h":709] 0x2030690 async_read_header 142 Bytes
Jan 01 00:07:08 starlake bmcweb[394]: (1970-01-01 00:07:08) [DEBUG "http_connection.h":750] 0x2030690 doRead
Jan 01 00:07:08 starlake bmcweb[394]: (1970-01-01 00:07:08) [DEBUG "http_connection.h":757] 0x2030690 async_read 0 Bytes
Jan 01 00:07:08 starlake bmcweb[394]: (1970-01-01 00:07:08) [DEBUG "http_connection.h":827] 0x2030690 timer cancelled: 0x1f73930 6
Jan 01 00:07:08 starlake bmcweb[394]: (1970-01-01 00:07:08) [INFO "http_connection.h":511] Request:  0x2030690 HTTP/1.1 GET /redfish/v1/Managers/bmc/info
Jan 01 00:07:08 starlake bmcweb[394]: (1970-01-01 00:07:08) [DEBUG "token_authorization_middleware.hpp":196] [AuthMiddleware] X-Auth-Token authentication
Jan 01 00:07:08 starlake bmcweb[394]: (1970-01-01 00:07:08) [DEBUG "token_authorization_middleware.hpp":212] [AuthMiddleware] Cookie authentication
Jan 01 00:07:08 starlake bmcweb[394]: (1970-01-01 00:07:08) [DEBUG "token_authorization_middleware.hpp":142] [AuthMiddleware] Basic authentication
Jan 01 00:07:08 starlake bmcweb[394]: (1970-01-01 00:07:08) [DEBUG "token_authorization_middleware.hpp":164] [AuthMiddleware] Authenticating user: root
Jan 01 00:07:08 starlake bmcweb[394]: pam_succeed_if(webserver:auth): requirement "user ingroup redfish" was met by user "root"
Jan 01 00:07:08 starlake bmcweb[394]: (1970-01-01 00:07:08) [CRITICAL "token_authorization_middleware.hpp":67] /redfish/v1/Managers/bmc/info
....
```

## FAQ

* 如果万一发现工作的git目录被bitbake还原了怎么办？

* 某个bbapped项目支持哪些命令？

``` bash
bitbake -c listtasks bmcweb
```