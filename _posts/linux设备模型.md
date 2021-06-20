bus:所有设备都应链接到总线上(cpu内部总线，虚拟总线，platform 总线)
class:集合具有相似功能或属性的设备，用于抽象出一套可以在多个设备间公用的数据结构和接口函数，用于继承
device:抽象系统中所有的硬件设备，描述名字、属性、从属的bus，从属的class
device driver:Linux设备模型用Driver抽象硬件设备的驱动程序，它包含设备初始化、电源管理相关的接口实现。而Linux内核中的驱动开发，基本都围绕该抽象进行（实现所规定的接口函数）

只要任何device,device driver具有相同的名字，内核就会执行device driver结构体中初始化函数probe(每个bus的match回调)，初始化设备，进入可状态
device driver一直存在内核中，当设备没有插入时，device结构不存在，probe函数不会执行；当设备插入时，内核会创建一个devic结构(名称和driver相同),则触发driver执行

通过bus->device的树形结构解决设备之间依赖

使用class结构，在设备模型中引入面对对象概念，可以最大限度地抽象共性，减少驱动开发过程中的重复劳动

## kobject

documentation/kobject.txt; include/linux/kobject.h; lib/kobject.c

1. 通过parent指针，将所有kobject以层次结构的形式组合起来

2. 使用引用计数(reference count),记录kobject被引用的次数，并在引用次数变0是释放

3. 和sysfs虚拟文件系统配合，将每一个kobject及其特性，以文件形式，开放到用户空间

### 主要数据结构

kobject, kset, ktype

kobject:基本数据类型，每个kobject都会在/sys/文件系统中以目录的形式出现

ktype:代表kobject的属性操作集合；多个kobject可能共用同一个属性操作集

kset:特殊的kobject，用来集合相似的kobject(可以有相同的属性，不同的属性)

``` c
 2: struct kobject {
 3:     const char *name; kobject名称，对应sysfs目录名称；添加到kernel时，需根据名字注册到sysfs中，之后就不能在直接修改该字段
 4:     struct list_head    entry; 用于将kobject加入到kset中的list_head
 5:     struct kobject      *parent;  指向parent kobject,以此形成层次结构(在sysfs中表现为目录结构)
 6:     struct kset     *kset;  该object属于的kset,如果未制定，则会把kset作为parent
 7:     struct kobj_type    *ktype; 该kobject属于的obj_type，每个kobject必须有一个ktype
 8:     struct sysfs_dirent *sd; 该kobject在sysfs中的表示
 9:     struct kref     kref;  "struct kref”类型（在include/linux/kref.h中定义）的变量，为一个可用于原子操作的引用计数
 10:    unsigned int state_initialized:1; 指示该Kobject是否已经初始化，以在Kobject的Init，Put，Add等操作时进行异常校验。
 11:    unsigned int state_in_sysfs:1;  指示该Kobject是否已在sysfs中呈现，以便在自动注销时从sysfs中移除
 12:    unsigned int state_add_uevent_sent:1;
 13:    unsigned int state_remove_uevent_sent:1;记录是否已经向用户空间发送ADD uevent，如果有，且没有发送remove uevent，则在自动注销时，补发REMOVE uevent，以便让用户空间正确处理
 14:    unsigned int uevent_suppress:1;如果该字段为1，则表示忽略所有上报的uevent事件
 15: };
```

uevent提供了用户空间通知功能，当内核中有Kobject添加、删除、修改动作时，会通知到用户空间


``` c
 1: /* include/linux/kobject.h, line 159 */
 2: struct kset {
 3:     struct list_head list;
 4:     spinlock_t list_lock;  用于保存该kset下所有kobject的链表
 5:     struct kobject kobj;  kset自己的kobject(kset时一个特殊的kobject，会在sysfs中以目录形式出现)
 6:     const struct kset_uevent_ops *uevent_ops; 该kset的uevent操作函数集，当任何kobject需要上报uevent时，都要调用它从属的kset的uevent_ops，添加环境变量，或者过滤event(决定那些event可以上报);当一个kobject不属于任何kset时，不允许发送uevent
 7: };
 ```

 ``` c
 1: /* include/linux/kobject.h, line 108 */
 2: struct kobj_type {
 3:     void (*release)(struct kobject *kobj);  回调函数，可以将包含该类型kobject的数据结构的内存释放掉
 4:     const struct sysfs_ops *sysfs_ops;     该种类型的kobject的sysfs文件系统接口
 5:     struct attribute **default_attrs;     该种类型的kobject的attribute列表(对应sysfs文件系统的一个文件)
 6:     const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
 7:     const void *(*namespace)(struct kobject *kobj);
 8: };
 ```



