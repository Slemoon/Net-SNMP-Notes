# Net-SNMP Notes


## Net-SNMP 是什么？

&emsp;&emsp;NET-SNMP 是一种开放源代码的 SNMP 实现。它支持 SNMP v1, SNMP v2c 与 SNMP v3，并可以使用 IPV4 及 IPV6 。也包含 SNMP Trap 的所有相关实现。

[ArchWiki Snmpd](https://wiki.archlinux.jp/index.php/Snmpd)

## Net-SNMP 包括哪些功能？

命令行应用程序
- 从支持 SNMP 的设备检索信息，使用单个请求（snmpget，snmpgetnext），或者使用多个请求（snmpwalk、snmptable、snmpdelta）

- 操作支持 SNMP 的设备（snmpset）上的配置信息

- 从支持 SNMP 的设备（snmpdf、snmpnetstat、snmpstatus）检索一个固定的信息集合

- 在 MIB/OI 的数值形式和文本形式之间进行转换，并显示 MIB 内容和结构（snmptranslate）

图形MIB浏览器 （Tkmib） 使用 Tk/perl

用于接收 SNMP 通知（Snmpd）的守护进程（Systemd deamon） 可以选择的通知（到 Syslog、NT 事件日志或纯文本文件） 转发到另一个 SNMP 管理系统或传递到外部应用程序

用于响应 SNMP 查询以获取有关管理信息（Snmpd） 的可扩展代理 这包括对各种 MIB 信息模块的内置支持，并且可以使用动态加载的模块、外部脚本和命令以及 SNMP 多路复用（Smux）和代理扩展性（AgentX）协议进行扩展。

用于开发具有 C 和 perl API 的新 SNMP 应用程序的库


## 安装 （通过包管理器）

Arch-Linux / Manjaro :
```bash
$ sudo pacman -Sy net-snmp
```

Ubuntu / Debian GNU/Linux :
```bash
$ sudo apt-get install -y net-snmp libperl-dev
```

CentOS :
```bash
$ sudo yum install -y net-snmp net-snmp-devel net-snmp-libs net-snmp-perl net-snmp-utils mrtg
```

## 部署 （通过源码编译）

安装 GNU GCC 编译器

[下载源码](https://sourceforge.net/projects/net-snmp/files/net-snmp/)

&emsp;&emsp;执行文件目录下的 configure 可执行文件，如果想指定程序包的安装路径，那么首先建立相应的文件夹来存放安装信息，可以写成 ./configure --prefix= 指定的路径名。参数 --prefix 用来告诉系统安装信息存放的路径，如果没有指定路径直接执行 ./configure，那么程序包都会安装在系统默认的目录下，通常为 /usr/local 或者 /opt下

例如 :

```bash
$ sudo ./configure --prefix=/usr/local/snmp
```

可以加入支持磁盘I/O监控等等功能
```bash
$ sudo ./configure --prefix=/usr/local/snmp --with-mib-modules='ucd-snmp/diskio ip-mib/ipv4InterfaceTable'
```

在配置过程中需要进行一些简单的选择 :

```bash
default version of-snmp-version: 2 
Systemcontact information: yourname                                       配置该设备的联系人
System location: china                                                    该设备的位置
Location to write logfile: /var/log/snmpd.log                             日志文件位置
Location to Write persistent: /var/net-snmp                               数据存储目录
```

编译并且安装 :

```bash
$ sudo make && sudo make install
```

## Snmpd 基本配置

```
/etc/snmp/snmpd.conf
```

配置只读字符串
```bash
$ sudo echo "rocommunity read_only_community_string" >> /etc/snmp/snmpd.conf
```

添加另一个用于管理的社区字符串
```bash
$ sudo echo "rwcommunity read_write_community_string" >> /etc/snmp/snmpd.conf
```

SNMP v3增加了安全性和加密的身份验证使用不同的配置方案   /var/net-snmp/snmpd.conf
```bash
$ sudo echo "rouser read_only_user" >> /etc/snmp/snmpd.conf
```

或使用向导
```bash
$ sudo snmpconf -g basic_setup
```

配置   /var/net-snmp/snmpd.conf

```bash
$ sudo echo "createUser read_only_user SHA password1 AES password2" >> /var/net-snmp/snmpd.conf
```

或使用配置工具

```bash
$ sudo net-snmp-create-v3-user -ro -a SHA -x AES
```
## Start Systemd Deamon

```bash
$ sudo systemctl enable snmpd.service
$ sudo systemctl start snmpd.service
```
检查 Systemd 进程状态
```bash
$ sudo systemctl status snmpd.service
```

## Testing

```bash
$ sudo snmpwalk -v 1 -c read_only_community_string localhost | less                                              SNMP v1
$ sudo snmpwalk -v 2c -c read_only_community_string localhost | less                                             SNMP v2c
$ sudo snmpwalk -v 3 -u read_only_user -a SHA -A password1 -x AES -X password2 -l authNoPriv localhost | less    SNMP v3
```

不论哪种方式，下方会显示本机 MIB 信息

## 从设备检索信息

snmpget 与 snmpwalk 的区别

- snmpwalk 是对OID值的遍历

&emsp;&emsp;若某个OID值下面有N个节点，则依次遍历出这N个节点的值。如果对某个叶子节点的 OID 值做walk，则取得到数据就不正确了，因为它会认为该节点是某些节点的父节点，而对其进行遍历，而实际上该节点已经没有子节点了，那么它会取出与该叶子节点平级的下一个叶子节点的值，而不是当前请求的节子节点的值

比如 

```bash
$ sudo snmpwalk -v 2c -c read_only_community_string localhost | less
$ sudo snmpwalk -v 2c -c read_only_community_string localhost system  
```

- snmpget 是取具体的OID的值。

snmpget 适用于OID值是一个叶子节点的情况, 而且可以同时查询多个 OID 的值
```bash
$ sudo snmpget -IR localhost sysDescr.0 sysLocation.0
```

snmpgetnext
&emsp;&emsp;模拟SNMP GETNextRequest 动作的工具，用来获取一个管理信息实例的下一个可用实例数据
```bash
$ sudo snmpgetnext -v 2c -c read_only_community_string localhost sysDescr.0
```

## 设置设备信息

snmpset
&emsp;&emsp;模拟SNMP SETRequest 动作的工具，用来设置可写管理信息，一般用来配置设备对设备执行操作

```bash
$ sudo snmpset -v 2c -c read_write_community_string localhost sysContact.0 s netlab
$ sudo snmpset -v 2c -c read_write_community_string localhost sysName.0 s ccut sysLocation.0 s soft
```

## SnmpTrap 的发送和接收

虽然 SNMPTrap SNMPTrapd 和 SNMPInform 功能模块在 Net-SNMP 源码包里。但是 Ubuntu 和 Pop! OS 等发行版将这部分功能单独提供了 binary 软件包

Ubuntu :
```bash
$ sudo apt-get install -y snmptrapd
```

Ubuntu 20.04 LTS &ensp; &ensp; [Ubuntu Launchpad](https://launchpad.net/ubuntu/focal/+package/snmptrapd)

- snmptrap：可以模拟snmp agent发送一个trap到snmp管理端

- snmpinform：可以模拟snmp agent发送一个inform request到snmp管理端

&emsp;&emsp;Trap是发送给SNMP管理者的通知网络状况等的警告消息，而Inform是需要SNMP管理者确认接收的Trap。与Inform 相比较，Trap通知方式为不可靠传输，因为snmp管理端在收到一条Trap通知后无需回复任何确认信息，所以snmp agent无法知道Trap通知是否已经被snmp管理端正确接收

- snmptrapd：一个模拟snmp管理端接收trap/inform通知的程序

snmptrapd 应用程序作为后台 SNMP Trap 服务器，负责接收被管理设备发送过来的 Trap 消息

创建snmptrapd.conf 的配置文件

```bash
$ sudo nano /etc/snmp/snmptrapd.conf

traphandle default lognotify IBM-DW-SAMPLE::nodeDown 
authCommunity log,execute,net public

```

创建 snmpdtrapd 进程

```bash
$ sudo snmpdtrapd -C -c /etc/snmp/snmptrapd.conf udp:1632 -df -Lo
```

发送SNMP v1版本trap报文的方法

```bash
$ sudo snmptrap -v 1 -c public 192.168.16.46  1.3.6.1.4.1.1  192.168.16.46  2  3  1000  1.3.6.1.9.9.44.1.2.1  i  12  1.3.4.1.2.3.1  s  test
```

发送SNMP v2c 版本trap报文的方法

```bash
$ sudo snmptrap -v 2c -c public 192.168.16.46  1.3.6.1.4.1.1  SNMPv2-MIB::sysLocation.0  s  test
```

发送一个 Inform Request
```bash
$ sudo snmpinform
```