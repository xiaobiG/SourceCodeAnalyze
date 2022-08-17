# 概述

SELinux是Google从android 5.0开始，强制引入的一套非常严格的权限管理机制，主要用于增强系统的安全性。

然而，在开发中，我们经常会遇到由于SELinux造成的各种权限不足，即使拥有“万能的root权限”，也不能获取全部的权限。本文旨在结合具体案例，讲解如何根据log来快速解决90%的SELinux权限问题。


# 调试确认SELinux问题

为了澄清是否因为SELinux导致的问题，可先执行：

```shell
# （临时禁用掉SELinux）
adb shell setenforce 0 

#  （得到结果为Permissive）
adb shell getenforce
```

如果问题消失了，基本可以确认是SELinux造成的权限问题，需要通过正规的方式来解决权限问题。



遇到权限问题，在 logcat 或者 kernel 的log中一定会打印 `avc denied` 提示缺少什么权限，可以通过命令过滤出所有的 `avc denied`，再根据这些log各个击破：

```
cat /proc/kmsg | grep avc 
# 或
dmesg | grep avc
```

例如：

```
audit(0.0:67): avc: denied { write } for path="/dev/block/vold/93:96" dev="tmpfs" ino=1263 scontext=u:r:kernel:s0 tcontext=u:object_r:block_device:s0 tclass=blk_file permissive=0
```

**可以看到有avc denied，且最后有permissive=0，表示不允许。**





# SELinux 模式

https://wiki.centos.org/zh/HowTos/SELinux

SELinux 拥有三个基本的操作模式，当中 **Enforcing** 是预设的模式。此外，它还有一个 **targeted** 或 **mls** 的修饰语。这管制 SELinux 规则的应用有多广泛，当中 **targeted** 是较宽松的级别。

- **Enforcing：** 这个预设模式会在系统上启用并实施 SELinux 的安全性政策，拒绝存取及记录行动
- **Permissive：** 在 Permissive 模式下，SELinux 会被启用但不会实施安全性政策，而只会发出警告及记录行动。Permissive 模式在排除 SELinux 的问题时很有用
- **Disabled：** SELinux 已被停用



可使用 sestatus 这个指令来检视现时的 SELinux 状况：

```
# sestatus
SELinux status:                 enabled
SELinuxfs mount:                /selinux
Current mode:                   enforcing
Mode from config file:          enforcing
Policy version:                 21
Policy from config file:        targeted
```

setenforce 这个指令可以即时切换 **Enforcing** 及 **Permissive** 这两个模式，但请注意这些改动在系统重新开机时不会被保留。

要令改动过渡系统开机，请在 `/etc/selinux/config` 内修改 SELINUX= 这一行为 enforcing、permissive 或 disabled。例如：SELINUX=permissive。



# 在 Permissive 模式下搜集审计日志

当一个程式重复地被 SELinux 拒绝某个操作，有时在 permissive 模式下进行侦错会较容易。一般的做法是 `setenforce 0`，但这样做会导致所有区域都进行 permissive 模式，而不限于有问题的进程。为了避免这个情况，SELinux 支援 permissive 类别，让管理员可选择只将一个区域放进 permissive 模式，而不是整个系统。

若要根据审计日志的记录来设定，请查阅 **scontext** 栏内的类别：

```
type=AVC msg=audit(1218128130.653:334): avc: denied { connectto } for pid=9111 comm="smtpd" path="/var/spool/postfix/postgrey/socket"
scontext=system_u:system_r:postfix_smtpd_t:s0 tcontext=system_u:system_r:initrc_t:s0 tclass=unix_stream_socket
```

```
semanage permissive -a postfix_smtpd_t
```

```
semanage permissive -d postfix_smtpd_t
```





# 某app例子

```
<4>[ 1215.035144]  .(2)[191:logd.auditd]type=1400 audit(1590568534.052:9): avc: denied { read } for pid=1670 comm="ee.smarthome_wl" name="libstlport_shared.so" dev="mmcblk0p20" ino=81935 scontext=u:r:untrusted_app:s0:c512,c768 tcontext=u:object_r:system_data_file:s0 tclass=file permissive=1

<38>[ 1215.037839] .(2)[191:logd.auditd]type=1400 audit(1590568534.052:10): avc: denied { getattr } for pid=1670 comm="ee.smarthome_wl" path="/data/app-lib/DoorAlarm/libstlport_shared.so" dev="mmcblk0p20" ino=81935 scontext=u:r:untrusted_app:s0:c512,c768 tcontext=u:object_r:system_data_file:s0 tclass=file permissive=1

<38>[ 1215.310770] .(0)[191:logd.auditd]type=1400 audit(1590568534.322:11): avc: denied { ioctl } for pid=1670 comm="ee.smarthome_wl" path="socket:[16315]" dev="sockfs" ino=16315 ioctlcmd=8927 scontext=u:r:untrusted_app:s0:c512,c768 tcontext=u:r:untrusted_app:s0:c512,c768 tclass=udp_socket permissive=1
```

```
allow untrusted_app system_data_file:file read;
allow untrusted_app system_data_file:file getattr;
allow untrusted_app untrusted_app:udp_socket ioctl;
```

make installclean后重新编译，刷boot.img才会生效。



### 修改文件

```
device/mediatek/common/sepolicy/bsp/untrusted_app.te
```





# 四个案例：

> https://blog.csdn.net/tung214/article/details/72734086

## 案例1

```
audit(0.0:67): avc: denied { write } for path="/dev/block/vold/93:96" dev="tmpfs" ino=/1263 scontext=u:r:kernel:s0 tcontext=u:object_r:block_device:s0 tclass=blk_file permissive=0
```

 

分析过程：

缺少什么权限：      { write }权限，

谁缺少权限：        scontext=u:r:kernel:s0

对哪个文件缺少权限：tcontext=u:object_r:block_device

什么类型的文件：    tclass=blk_file

完整的意思： kernel进程对block_device类型的blk_file缺少write权限。

 

解决方法：在上文A位置，找到kernel.te这个文件，加入以下内容：

allow  kernel  block_device:blk_file  write;

make installclean后重新编译，刷boot.img才会生效。

 

## 案例2

```
audit(0.0:53): avc: denied { execute } for  path="/data/data/com.mofing/qt-reserved-files/plugins/platforms/libgnustl_shared.so" dev="nandl" ino=115502 scontext=u:r:platform_app:s0 tcontext=u:object_r:app_data_file:s0 tclass=file permissive=0
```

 

分析过程：

缺少什么权限：      { execute}权限，

谁缺少权限：        scontext = u:r:platform_app:s0

对哪个文件缺少权限：tcontext = u:object_r:app_data_file

什么类型的文件：    tclass= file

完整的意思： platform_app进程对app_data_file类型的file缺少execute权限。

 

解决方法：在上文A位置，找到platform_app.te这个文件，加入以下内容：

allow  platform_app  app_data_file:file  execute;

make installclean后重新编译，刷boot.img才会生效。

 

## 案例3

```
audit(1444651438.800:8): avc: denied { search } for pid=158 comm="setmacaddr" name="/" dev="nandi" ino=1 scontext=u:r:engsetmacaddr:s0 tcontext=u:object_r:vfat:s0 tclass=dir permissive=0
```

解决方法 ：engsetmacaddr.te

allow  engsetmacaddr  vfat:dir  { search write add_name create }; 或者

allow  engsetmacaddr   vfat:dir  create_dir_perms;

(create_dir_perms包含search write add_name create可参考external/sepolicy/global_macros的定义声明)

 

## 案例4

```
audit(1441759284.810:5): avc: denied { read } for pid=1494 comm="sdcard" name="0" dev="nandk" ino=245281 scontext=u:r:sdcardd:s0 tcontext=u:object_r:system_data_file:s0 tclass=dir permissive=0
```

解决方法 ：sdcardd.te 

allow  sdcardd  system_data_file:dir  read;  或者
allow  sdcardd  system_data_file:dir  rw_dir_perms;

 (rw_dir_perms包含read write，可参考external/sepolicy/global_macros的定义声明)







> 参考 https://blog.csdn.net/tung214/article/details/72734086

> http://gityuan.com/2015/06/13/SEAndroid-permission/