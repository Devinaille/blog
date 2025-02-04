# 基于PVE虚拟环境的19C RAC安装
> 日期：2025-01-06
>
> tags：oracle,rac,10c,pve,vm,虚拟机

用PVE来搭建一套oracle 19c rac环境，并进行升级、迁移
<!-- more -->
## 本文目标
1. 在PVE环境下安装rac 19.3.0.0
2. 将rac集群升级到19.25.0.0
## 安装环境预设
### HOST&IP信息

| 节点     | 系统      | 物理IP         | 虚拟IP         | 私有IP    | SCANIP         |
| ------- | --------- | -------------- | -------------- | -------- | -------------- |
| rac1    | centos7.9 | 192.168.100.51 | 192.168.100.61 | 10.1.1.1 | 192.168.100.70 |
| rac2    | centos7.9 | 192.168.100.52 | 192.168.100.61 | 10.1.1.2 | 192.168.100.70 |
| storage | centos7.9 | 192.168.100.81 |         N/A    |N/A       |            N/A |
### 磁盘信息
| 类型   | 大小   | 挂载方式 | 文件系统 |
| ------ | ---- | ---- | ---- |
| 系统盘  | 100g |N/A   | xfs  |
| OCR    | 32G  | ASM  | ASM  |
| DATA01 | 100G | ASM  | ASM  |
## 虚拟机设置
设置其中一台，安装好后复制另一台
net0是业务网卡
net1是心跳卡

![微信截图_20241231092237.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231092237.png)

## 安装操作系统
###  语言选择English
![image.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/image.png)
### 磁盘分区
| 目录    | 文件格式    | 大小   |
| ----- | ------- | ---- |
| swap  | swap    | 16g  |
| /boot | xfs     | 1g   |
| /home | lvm+xfs | 按需   |
| /var  | lvm+xfs | 按需   |
| /     | lvm+xfs | 剩余大小 |

![微信截图_20241231090317.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231090317.png)
![微信截图_20241231090432.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231090432.png)

### 设置密码
![微信截图_20241231090545.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231090545.png)
### 配置网络
配置其中一个业务网卡，方便ssh

![微信截图_20241231091001.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231091001.png)

### 选择时区
Asia/shanghai，并勾选网络时间
![微信截图_20241231092944.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231092944.png)


### 选择最小化安装
![微信截图_20241231092554.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231092554.png)

### 开始安装
点击Begin install 安装

### 重启
![微信截图_20241231092153.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231092153.png)


## 复制一台虚拟机

![微信截图_20241231093843.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231093843.png)

### 修改hostname

```bash
hostnamectl set-hostname rac2
```
### 修改ip
因为是虚拟机，就不配置bond了
```bash
cd /etc/sysconfig/network-scripts
vi ifcfg-enp6s18
nmcli c down enp6s18
nmcli c up enp6s18

vi ifcfg-enp6s19
nmcli c down enp6s19
nmcli c up enp6s19
```

### 测试网络
```bash
[root@rac2 ~]# ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.333 ms
64 bytes from 10.1.1.1: icmp_seq=2 ttl=64 time=0.629 ms
^C
--- 10.1.1.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1019ms
rtt min/avg/max/mdev = 0.333/0.481/0.629/0.148 ms
[root@rac2 ~]# ping 192.168.100.51
PING 192.168.100.51 (192.168.100.51) 56(84) bytes of data.
64 bytes from 192.168.100.51: icmp_seq=1 ttl=64 time=0.166 ms
64 bytes from 192.168.100.51: icmp_seq=2 ttl=64 time=0.389 ms
^C
--- 192.168.100.51 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1042ms
rtt min/avg/max/mdev = 0.166/0.277/0.389/0.112 ms
```

## 配置yum源
### 挂载镜像
```bash
mkdir /media/cdrom
mount /dev/sr0 /media/cdrom
```
### 编写配置文件
```bash 
vi /etc/yum.repos.d/cdrom.repo
```

```repo
[AppStream]
name=AppStream
baseurl=file:///mnt/cdrom/AppStream
gpgcheck=0
enabled=1
gpgkey=/mnt/cdrom/RPM-GPG-KEY-redhat-release
[BaseOS]
name=BaseOS
baseurl=file:///mnt/cdrom/BaseOS
gpgcheck=0
enabled=1
gpgkey=/mnt/cdrom/RPM-GPG-KEY-redhat-release
```

### 重新加载yum缓存
```bash
yum --disablerepo=* --enablerepo=c7-media clean all
yum --disablerepo=* --enablerepo=c7-media makecache
```

## 配置iscsi
这里的存储源用truenas分配的iscsi
### TargetCli

#### 安装targetcli
```bash
[root@storage ~]# yum --disablerepo=* --enablerepo=c7-media install targetcli
```

#### 进入targetcli
```bash
[root@storage ~]# targetcli
Warning: Could not load preferences file /root/.targetcli/prefs.bin.
targetcli shell version 2.1.51
Copyright 2011-2013 by Datera, Inc and others.
For help on commands, type 'help'.

/> ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 0]
  | o- fileio ................................................................................................. [Storage Objects: 0]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 0]
  o- loopback ......................................................................................................... [Targets: 0]
```
#### 创建block设备
```bash

/backstores/block> create disk1 /dev/sdb
Created block storage object disk1 using /dev/sdb.
/backstores/block> create disk2 /dev/sdc
Created block storage object disk2 using /dev/sdc.
/backstores/block> create disk3 /dev/sdd
Created block storage object disk3 using /dev/sdd.
...
/backstores/block> cd ..
/backstores> cd ..
/> ls
o- / ......................................................................................................................... [...]
  o- backstores .............................................................................................................. [...]
  | o- block .................................................................................................. [Storage Objects: 3]
  | | o- disk1 ......................................................................... [/dev/sdb (32.0GiB) write-thru deactivated]
  | | | o- alua ................................................................................................... [ALUA Groups: 1]
  | | |   o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | | o- disk2 ......................................................................... [/dev/sdc (32.0GiB) write-thru deactivated]
  | | | o- alua ................................................................................................... [ALUA Groups: 1]
  | | |   o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | | o- disk3 ......................................................................... [/dev/sdd (32.0GiB) write-thru deactivated]
  | |   o- alua ................................................................................................... [ALUA Groups: 1]
  | |     o- default_tg_pt_gp ....................................................................... [ALUA state: Active/optimized]
  | o- fileio ................................................................................................. [Storage Objects: 0]
  | o- pscsi .................................................................................................. [Storage Objects: 0]
  | o- ramdisk ................................................................................................ [Storage Objects: 0]
  o- iscsi ............................................................................................................ [Targets: 0]
  o- loopback ......................................................................................................... [Targets: 0]
```
#### 创建iscsi设备
```bash
/> cd iscsi 
/iscsi> create 
Created target iqn.2003-01.org.linux-iscsi.storage.x8664:sn.4aeba5f26f70.
Created TPG 1.
Global pref auto_add_default_portal=true
Created default portal listening on all IPs (0.0.0.0), port 3260.
/iscsi> ls
o- iscsi .............................................................................................................. [Targets: 1]
  o- iqn.2003-01.org.linux-iscsi.storage.x8664:sn.4aeba5f26f70 ........................................................... [TPGs: 1]
    o- tpg1 ................................................................................................. [no-gen-acls, no-auth]
      o- acls ............................................................................................................ [ACLs: 0]
      o- luns ............................................................................................................ [LUNs: 0]
      o- portals ...................................................................................................... [Portals: 1]
        o- 0.0.0.0:3260 ....................................................................................................... [OK]
...
```

#### 创建lun
```bash
/iscsi/iqn.20...f70/tpg1/luns> create /backstores/block/disk1 
Created LUN 0.
/iscsi/iqn.20...f70/tpg1/luns> create /backstores/block/disk2
Created LUN 1.
/iscsi/iqn.20...f70/tpg1/luns> create /backstores/block/disk3 
Created LUN 2.

```
#### 创建acl（无密码）
这里需要创建2个ACL，不然RAC这边的客户端会报错
```
connection1:0: detected conn error (1020) 
```


```bash
iscsi/iqn.20...f70/tpg1/acls> pwd
/iscsi/iqn.2003-01.org.linux-iscsi.storage.x8664:sn.4aeba5f26f70/tpg1/acls
/iscsi/iqn.20...f70/tpg1/acls> create wwn=iqn.2003-01.org.linux-iscsi.storage.x8664:19c-rac1
Created Node ACL for iqn.2003-01.org.linux-iscsi.storage.x8664:19c-rac1
Created mapped LUN 2.
Created mapped LUN 1.
Created mapped LUN 0.
/iscsi/iqn.20...f70/tpg1/acls> create wwn=iqn.2003-01.org.linux-iscsi.storage.x8664:19c-rac2
Created Node ACL for iqn.2003-01.org.linux-iscsi.storage.x8664:19c-rac2
Created mapped LUN 2.
Created mapped LUN 1.
Created mapped LUN 0.

/> ls
...
  o- iscsi ............................................................................................................ [Targets: 1]
  | o- iqn.2003-01.org.linux-iscsi.storage.x8664:sn.4aeba5f26f70 ......................................................... [TPGs: 1]
  |   o- tpg1 ............................................................................................... [no-gen-acls, no-auth]
  |     o- acls .......................................................................................................... [ACLs: 2]
  |     | o- iqn.2003-01.org.linux-iscsi.storage.x8664:19c-rac1 ................................................... [Mapped LUNs: 3]
  |     | | o- mapped_lun0 ................................................................................. [lun0 block/disk1 (rw)]
  |     | | o- mapped_lun1 ................................................................................. [lun1 block/disk2 (rw)]
  |     | | o- mapped_lun2 ................................................................................. [lun2 block/disk3 (rw)]
  |     | o- iqn.2003-01.org.linux-iscsi.storage.x8664:19c-rac2 ................................................... [Mapped LUNs: 3]
  |     |   o- mapped_lun0 ................................................................................. [lun0 block/disk1 (rw)]
  |     |   o- mapped_lun1 ................................................................................. [lun1 block/disk2 (rw)]
  |     |   o- mapped_lun2 ................................................................................. [lun2 block/disk3 (rw)]
...
```
#### 保存配置
```bash
/> saveconfig                             
Configuration saved to /etc/target/saveconfig.json
```
### RAC服务器配置
#### 安装iSCSI客户端
```bash
yum install -y iscsi-initiator-utils
```
#### 查找iSCSI服务端
```bash
[root@rac3 ~]# iscsiadm -m discovery -t st -p 192.168.100.81 --discover
192.168.100.81:3260,1 iqn.2003-01.org.linux-iscsi.storage.x8664:sn.4aeba5f26f70
```
#### 配置initiatorname.iscsi
```bash
vi /etc/iscsi/initiatorname.iscsi

[root@rac1 ~]# cat /etc/iscsi/initiatorname.iscsi 
InitiatorName=iqn.2003-01.org.linux-iscsi.storage.x8664:19c-rac1

[root@rac2 ~]# cat /etc/iscsi/initiatorname.iscsi 
InitiatorName=iqn.2003-01.org.linux-iscsi.storage.x8664:19c-rac2
```

#### 查找iSCSI服务端
```bash
[root@rac1 ~]# iscsiadm -m discovery -t st -p 192.168.100.81 --discover
192.168.100.81:3260,1 iqn.2003-01.org.linux-iscsi.storage.x8664:sn.4aeba5f26f70
```

#### 添加iscsi
```bash
[root@rac1 ~]# iscsiadm -m discovery -t sendtargets -p 192.168.100.81 -l
192.168.100.81:3260,1 iqn.2003-01.org.linux-iscsi.storage.x8664:sn.4aeba5f26f70
Logging in to [iface: default, target: iqn.2003-01.org.linux-iscsi.storage.x8664:sn.4aeba5f26f70, portal: 192.168.100.81,3260] (multiple)
Login to [iface: default, target: iqn.2003-01.org.linux-iscsi.storage.x8664:sn.4aeba5f26f70, portal: 192.168.100.81,3260] successful.
```

#### 查看是否正常挂载
```bash
[root@rac1 ~]# fdisk -l

Disk /dev/sda: 107.4 GB, 107374182400 bytes, 209715200 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x0008fbfb

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200   209715199   103808000   8e  Linux LVM

Disk /dev/mapper/centos-root: 89.1 GB, 89116377088 bytes, 174055424 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/centos-swap: 17.2 GB, 17179869184 bytes, 33554432 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdb: 34.4 GB, 34359738368 bytes, 67108864 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33550336 bytes


Disk /dev/sdc: 34.4 GB, 34359738368 bytes, 67108864 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33550336 bytes


Disk /dev/sdd: 34.4 GB, 34359738368 bytes, 67108864 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 33550336 bytes
```
## RAC安装
### 配置hosts
修改`/etc/hosts`文件，在两个节点添加以下内容
```bash
192.168.100.51 rac1
192.168.100.52 rac2

192.168.100.61 rac1-vip
192.168.100.62 rac2-vip

10.1.1.1 rac1-priv
10.1.1.2 rac2-priv

192.168.100.70 rac-scan
```

### 关闭THP和NUMA
#### 查看当前THP状态
```bash
[root@rac1 ~]# cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never

[root@rac2 ~]# cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never
```

#### 查看numa是否启动
```bash
[root@rac1 ~]# dmesg | grep numa

[root@rac2 ~]# dmesg | grep numa
```
#### 修改grub文件
```bash
# 原始
GRUB_CMDLINE_LINUX="crashkernel=auto resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet"

# 修改后
GRUB_CMDLINE_LINUX="crashkernel=auto resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet transparent_hugepage=never numa=off"
```

#### 生成grub
```bash
[root@rac1 ~]# grub2-mkconfig -o /boot/grub2/grub.cfg 
Generating grub configuration file ...
done

[root@rac2 ~]# grub2-mkconfig -o /boot/grub2/grub.cfg 
Generating grub configuration file ...
done
```

#### reboot生效
```bash
reboot 
```

#### 查看重启后状态
```bash
[root@rac1 ~]# dmesg | grep numa
[    0.000000] Command line: BOOT_IMAGE=(hd0,msdos1)/vmlinuz-4.18.0-477.10.1.el8_8.x86_64 root=/dev/mapper/rhel-root ro crashkernel=auto resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet transparent_hugepage=never numa=off
[    0.000000] Kernel command line: BOOT_IMAGE=(hd0,msdos1)/vmlinuz-4.18.0-477.10.1.el8_8.x86_64 root=/dev/mapper/rhel-root ro crashkernel=auto resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet transparent_hugepage=never numa=off
[root@rac1 ~]# cat /sys/kernel/mm/transparent_hugepage/enabled
always madvise [never]


[root@rac2 ~]# dmesg | grep numa
[    0.000000] Command line: BOOT_IMAGE=(hd0,msdos1)/vmlinuz-4.18.0-477.10.1.el8_8.x86_64 root=/dev/mapper/rhel-root ro crashkernel=auto resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet transparent_hugepage=never numa=off
[    0.000000] Kernel command line: BOOT_IMAGE=(hd0,msdos1)/vmlinuz-4.18.0-477.10.1.el8_8.x86_64 root=/dev/mapper/rhel-root ro crashkernel=auto resume=/dev/mapper/rhel-swap rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap rhgb quiet transparent_hugepage=never numa=off
[root@rac2 ~]# cat /sys/kernel/mm/transparent_hugepage/enabled
always madvise [never]
```

### 设置hugepage 

```
vi /etc/security/limits.conf

*   soft   memlock    120795955
*   hard   memlock    120795955
```

### 关闭防火墙
```bash
systemctl stop firewalld
systemctl disable firewalld
```

### 安装依赖
```bash
yum install -y  libnsl device-mapper-multipath chrony bc binutils  elfutils-libelf elfutils-libelf-devel fontconfig-devel glibc glibc-devel ksh libaio libaio-devel libX11 libXau libXi libXtst libXrender libXrender-devel libgcc libstdc++ libstdc++-devel libxcb make smartmontools sysstat unixODBC unzip xauth xdpyinfo gcc-c++ nfs-utils net-tools 
```

### 配置时间同步
我这里有外网，所以不用配置，直接装好chrony就会同步时间

### 创建用户及目录
#### 创建用户组
```
groupadd -g 501 oinstall
groupadd -g 502 dba
groupadd -g 503 oper
groupadd -g 504 backupdba
groupadd -g 505 dgdba
groupadd -g 506 kmdba
groupadd -g 507 racdba
groupadd -g 508 asmadmin
groupadd -g 509 asmdba
groupadd -g 510 asmoper
```

#### 创建用户
创建oracle用户
```bash
useradd -u 501 -g oinstall -G oinstall,dba,oper,backupdba,dgdba,racdba,kmdba,asmdba -d /home/oracle -s /bin/bash oracle
```

创建grid用户
```bash
useradd -u 502 -g oinstall -G oinstall,dba,racdba,asmadmin,asmdba,asmoper -m -d /home/grid grid
```

#### 创建目录
```bash
mkdir -p /u01/grid/grid
mkdir -p /u01/grid/19.3.0
mkdir -p /u01/oracle
mkdir -p /u01/oracle/product/19.3.0/db
chown -R grid:oinstall /u01
chown -R oracle:oinstall /u01/oracle
chmod -R 775 /u01
```

#### 添加用户限制
追加到/etc/security/limits.conf
```bash
grid soft nproc 65536
grid hard nproc 65536
grid soft nofile 65536
grid hard nofile 65536
oracle soft nproc 65536
oracle hard nproc 65536
oracle soft nofile 65536
oracle hard nofile 65536
```

把下列参数设置到 /etc/profile
```bash
if [ $USER = "oracle"  ]||[ $USER = "grid" ];then
  if [ $SHELL = "/bin/ksh" ];then
    ulimit -p 16384
    ulimit -n 65536
  else
    ulimit -u 16384 -n 65536
  fi
fi
```
#### 写入环境变量

oracle
```bash
export ORACLE_HOSTNAME=rac1
export ORACLE_SID=db1
export ORACLE_BASE=/u01/oracle
export ORACLE_HOME=/u01/oracle/product/19.3.0/db
export ORACLE_TERM=xterm
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"
export NLS_LANG="SIMPLIFIED CHINESE_CHINA.ZHS16GBK"
export TNS_ADMIN=$ORACLE_HOME/network/admin
export ORA_NLS11=$ORACLE_HOME/nls/data
export PATH=.:${JAVA_HOME}/bin:${PATH}:$HOME/bin:$ORACLE_HOME/bin
export PATH=${PATH}:/usr/bin:/bin:/usr/bin/X11:usr/local/bin
export PATH=${PATH}:/u01/app/common/oracle/bin
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:$ORACLE_HOME/oracm/lib
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/lib:/usr/lib:/usr/local/lib
export CLASSPATH=$ORACLE_HOME/JRE
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/jlib
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/rdbms/jlib
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/network/jlib
export THREADS_FLAG=native
export TEMP=/tmp
export TMPDIR=/tmp
umask 022
```

grid
```bash
export ORACLE_HOSTNAME=rac1
export ORACLE_SID=+ASM1
export ORACLE_BASE=/u01/grid/grid
export ORACLE_HOME=/u01/grid/19.3.0
export ORACLE_TERM=xterm
export NLS_DATA_FORMAT="DD-MON-YYYY HH24:MI:SS"
export TNS_ADMIN=$ORACLE_HOME/network/admin
export ORA_NLS11=$ORACLE_HOME/nls/data
export PATH=.:${JAVA_HOME}/bin:${PATH}:$HOME/bin:$ORACLE_HOME/bin
export PATH=${PATH}:/usr/bin:/bin:/usr/bin/X11:usr/local/bin
export PATH=${PATH}:/u01/app/common/oracle/bin
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:$ORACLE_HOME/oracm/lib
export LD_LIBRARY_PATH=${LD_LIBRARY_PATH}:/lib:/usr/lib:/usr/local/lib
export CLASSPATH=$ORACLE_HOME/JRE
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/jlib
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/rdbms/jlib
export CLASSPATH=${CLASSPATH}:$ORACLE_HOME/network/jlib
export THREADS_FLAG=native
export TEMP=/tmp
export TMPDIR=/tmp
umask 022
```
#### 配置multipath
```bash
modprobe dm-multipath
modprobe dm-round-robin
systemctl enable multipathd.service

multipath --enable
multipath -v3
multipath -a /dev/sdb
multipath -a /dev/sdc
systemctl restart multipathd.service
```


```
[root@rac2 dev]# cat /etc/multipath.conf 
# device-mapper-multipath configuration file

# For a complete list of the default configuration values, run either:
# # multipath -t
# or
# # multipathd show config

# For a list of configuration options with descriptions, see the
# multipath.conf man page.

defaults {
        user_friendly_names yes
        find_multipaths yes
        enable_foreign "^$"
}

# blacklist_exceptions {
#         property "(SCSI_IDENT_|ID_WWN)"
# }

blacklist {
}

multipaths {
    multipath {
        wwid 36589cfc00000063dd1e2719e926d0802
        alias mpathb
  }
    multipath {
        wwid 36589cfc000000b8cb5d6d2970b9fc97b
        alias mpathc
  }
}
```

### 配置udev
目的是为了固定磁盘号码，免得重启导致磁盘顺序变化
写udev配置文件
```bash
[root@rac1 ~]# cat /etc/udev/rules.d/99-oracle.asmdevices.rules
KERNEL=="dm-*",ENV{DM_UUID}=="mpath-36589cfc00000063dd1e2719e926d0802",SYMLINK+="asm/ocr01",OWNER="grid",GROUP="asmadmin",MODE="0660"
KERNEL=="dm-*",ENV{DM_UUID}=="mpath-36589cfc000000b8cb5d6d2970b9fc97b",SYMLINK+="asm/data01",OWNER="grid",GROUP="asmadmin",MODE="0660"
```

生效udev
```bash
[root@rac1 ~]# udevadm control --reload-rules && udevadm trigger
[root@rac1 ~]# ls /dev/asm*
/dev/asm_data01  /dev/asm_ocr01
```
### 安装grid
```bash 
su – grid
$ unzip LINUX.X64_193000_grid_home.zip -d $ORACLE_HOME
$ cd $ORACLE_HOME
$ sh gridSetup.sh
```
![微信截图_20241231161706.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231161706.png)
![微信截图_20241231161731.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231161731.png)
![微信截图_20241231161748.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231161748.png)
![微信截图_20241231170523.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231170523.png)
![微信截图_20241231170546.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231170546.png)
![微信截图_20241231170559.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231170559.png)
![微信截图_20241231170626.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20241231170626.png)
![微信截图_20250107181222.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107181222.png)


![微信截图_20250107180446.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107180446.png)
![微信截图_20250107180514.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107180514.png)
![微信截图_20250107180538.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107180538.png)
![微信截图_20250107180547.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107180547.png)
![微信截图_20250107180654.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107180654.png)
点击fix，
![微信截图_20250107180723.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107180723.png)
![微信截图_20250107181407.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107181407.png)
这些可以忽略，内存除外（虚拟机测试环境无所谓）
![微信截图_20250107181448.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107181448.png)
![微信截图_20250107181535.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107181535.png)

#### 节点1执行命令
```bash
[root@rac1 ~]# /u01/grid/19.3.0/root.sh
Performing root user operation.

The following environment variables are set as:
    ORACLE_OWNER= grid
    ORACLE_HOME=  /u01/grid/19.3.0

Enter the full pathname of the local bin directory: [/usr/local/bin]: 
The contents of "dbhome" have not changed. No need to overwrite.
The contents of "oraenv" have not changed. No need to overwrite.
The contents of "coraenv" have not changed. No need to overwrite.

Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root script.
Now product-specific root actions will be performed.
Relinking oracle with rac_on option
Using configuration parameter file: /u01/grid/19.3.0/crs/install/crsconfig_params
The log of current session can be found at:
  /u01/grid/grid/crsdata/rac1/crsconfig/rootcrs_rac1_2025-01-07_06-15-52PM.log
2025/01/07 18:15:58 CLSRSC-594: Executing installation step 1 of 19: 'SetupTFA'.
2025/01/07 18:15:58 CLSRSC-594: Executing installation step 2 of 19: 'ValidateEnv'.
2025/01/07 18:15:58 CLSRSC-363: User ignored prerequisites during installation
2025/01/07 18:15:58 CLSRSC-594: Executing installation step 3 of 19: 'CheckFirstNode'.
2025/01/07 18:15:59 CLSRSC-594: Executing installation step 4 of 19: 'GenSiteGUIDs'.
2025/01/07 18:16:00 CLSRSC-594: Executing installation step 5 of 19: 'SetupOSD'.
2025/01/07 18:16:00 CLSRSC-594: Executing installation step 6 of 19: 'CheckCRSConfig'.
2025/01/07 18:16:00 CLSRSC-594: Executing installation step 7 of 19: 'SetupLocalGPNP'.
2025/01/07 18:16:09 CLSRSC-594: Executing installation step 8 of 19: 'CreateRootCert'.
2025/01/07 18:16:11 CLSRSC-594: Executing installation step 9 of 19: 'ConfigOLR'.
2025/01/07 18:16:18 CLSRSC-4002: Successfully installed Oracle Trace File Analyzer (TFA) Collector.
2025/01/07 18:16:21 CLSRSC-594: Executing installation step 10 of 19: 'ConfigCHMOS'.
2025/01/07 18:16:21 CLSRSC-594: Executing installation step 11 of 19: 'CreateOHASD'.
2025/01/07 18:16:24 CLSRSC-594: Executing installation step 12 of 19: 'ConfigOHASD'.
2025/01/07 18:16:24 CLSRSC-330: Adding Clusterware entries to file 'oracle-ohasd.service'
2025/01/07 18:16:42 CLSRSC-594: Executing installation step 13 of 19: 'InstallAFD'.
2025/01/07 18:16:45 CLSRSC-594: Executing installation step 14 of 19: 'InstallACFS'.
2025/01/07 18:16:48 CLSRSC-594: Executing installation step 15 of 19: 'InstallKA'.
2025/01/07 18:16:50 CLSRSC-594: Executing installation step 16 of 19: 'InitConfig'.

ASM has been created and started successfully.

[DBT-30001] Disk groups created successfully. Check /u01/grid/grid/cfgtoollogs/asmca/asmca-250107PM061718.log for details.

2025/01/07 18:18:13 CLSRSC-482: Running command: '/u01/grid/19.3.0/bin/ocrconfig -upgrade grid oinstall'
CRS-4256: Updating the profile
Successful addition of voting disk 21bc14685f1c4fd2bfb21b4f80079fed.
Successfully replaced voting disk group with +CRSDG.
CRS-4256: Updating the profile
CRS-4266: Voting file(s) successfully replaced
##  STATE    File Universal Id                File Name Disk group
--  -----    -----------------                --------- ---------
 1. ONLINE   21bc14685f1c4fd2bfb21b4f80079fed (/dev/asm/ocr01) [CRSDG]
Located 1 voting disk(s).
2025/01/07 18:19:34 CLSRSC-594: Executing installation step 17 of 19: 'StartCluster'.
2025/01/07 18:20:45 CLSRSC-343: Successfully started Oracle Clusterware stack
2025/01/07 18:20:45 CLSRSC-594: Executing installation step 18 of 19: 'ConfigNode'.
2025/01/07 18:22:03 CLSRSC-594: Executing installation step 19 of 19: 'PostConfig'.
2025/01/07 18:22:27 CLSRSC-325: Configure Oracle Grid Infrastructure for a Cluster ... succeeded
[root@rac1 ~]# 
```
#### 节点2执行命令
```bash
[root@rac2 ~]# /u01/grid/19.3.0/root.sh
Performing root user operation.

The following environment variables are set as:
    ORACLE_OWNER= grid
    ORACLE_HOME=  /u01/grid/19.3.0

Enter the full pathname of the local bin directory: [/usr/local/bin]: 
The contents of "dbhome" have not changed. No need to overwrite.
The contents of "oraenv" have not changed. No need to overwrite.
The contents of "coraenv" have not changed. No need to overwrite.

Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root script.
Now product-specific root actions will be performed.
Relinking oracle with rac_on option
Using configuration parameter file: /u01/grid/19.3.0/crs/install/crsconfig_params
The log of current session can be found at:
  /u01/grid/grid/crsdata/rac2/crsconfig/rootcrs_rac2_2025-01-07_06-24-37PM.log
2025/01/07 18:24:40 CLSRSC-594: Executing installation step 1 of 19: 'SetupTFA'.
2025/01/07 18:24:41 CLSRSC-594: Executing installation step 2 of 19: 'ValidateEnv'.
2025/01/07 18:24:41 CLSRSC-363: User ignored prerequisites during installation
2025/01/07 18:24:41 CLSRSC-594: Executing installation step 3 of 19: 'CheckFirstNode'.
2025/01/07 18:24:41 CLSRSC-594: Executing installation step 4 of 19: 'GenSiteGUIDs'.
2025/01/07 18:24:41 CLSRSC-594: Executing installation step 5 of 19: 'SetupOSD'.
2025/01/07 18:24:41 CLSRSC-594: Executing installation step 6 of 19: 'CheckCRSConfig'.
2025/01/07 18:24:42 CLSRSC-594: Executing installation step 7 of 19: 'SetupLocalGPNP'.
2025/01/07 18:24:42 CLSRSC-594: Executing installation step 8 of 19: 'CreateRootCert'.
2025/01/07 18:24:42 CLSRSC-594: Executing installation step 9 of 19: 'ConfigOLR'.
2025/01/07 18:24:50 CLSRSC-594: Executing installation step 10 of 19: 'ConfigCHMOS'.
2025/01/07 18:24:50 CLSRSC-594: Executing installation step 11 of 19: 'CreateOHASD'.
2025/01/07 18:24:51 CLSRSC-594: Executing installation step 12 of 19: 'ConfigOHASD'.
2025/01/07 18:24:51 CLSRSC-330: Adding Clusterware entries to file 'oracle-ohasd.service'
2025/01/07 18:25:01 CLSRSC-4002: Successfully installed Oracle Trace File Analyzer (TFA) Collector.
2025/01/07 18:25:06 CLSRSC-594: Executing installation step 13 of 19: 'InstallAFD'.
2025/01/07 18:25:07 CLSRSC-594: Executing installation step 14 of 19: 'InstallACFS'.
2025/01/07 18:25:08 CLSRSC-594: Executing installation step 15 of 19: 'InstallKA'.
2025/01/07 18:25:09 CLSRSC-594: Executing installation step 16 of 19: 'InitConfig'.
2025/01/07 18:25:15 CLSRSC-594: Executing installation step 17 of 19: 'StartCluster'.
2025/01/07 18:26:07 CLSRSC-343: Successfully started Oracle Clusterware stack
2025/01/07 18:26:07 CLSRSC-594: Executing installation step 18 of 19: 'ConfigNode'.
2025/01/07 18:26:17 CLSRSC-594: Executing installation step 19 of 19: 'PostConfig'.
2025/01/07 18:26:21 CLSRSC-325: Configure Oracle Grid Infrastructure for a Cluster ... succeeded
```
#### 继续安装
![微信截图_20250107182830.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107182830.png)
查看日志是因为少于3个SCAN ip，且没有域名解析（也就是前面忽略的）
```bash
INFO:  [Jan 7, 2025 6:28:08 PM] Verifying Single Client Access Name (SCAN) ...FAILED
INFO:  [Jan 7, 2025 6:28:08 PM] rac2: PRVG-11368 : A SCAN is recommended to resolve to "3" or more IP
INFO:  [Jan 7, 2025 6:28:08 PM]       addresses, but SCAN "rac-scan" resolves to only "/192.168.100.70"
INFO:  [Jan 7, 2025 6:28:08 PM] rac1: PRVG-11368 : A SCAN is recommended to resolve to "3" or more IP
INFO:  [Jan 7, 2025 6:28:08 PM]       addresses, but SCAN "rac-scan" resolves to only "/192.168.100.70"
INFO:  [Jan 7, 2025 6:28:08 PM]   Verifying DNS/NIS name service 'rac-scan' ...FAILED
INFO:  [Jan 7, 2025 6:28:08 PM]   PRVG-11826 : DNS resolved IP addresses "" for SCAN name "rac-scan" not found
INFO:  [Jan 7, 2025 6:28:08 PM]   in the name service returned IP addresses "192.168.100.70"
INFO:  [Jan 7, 2025 6:28:08 PM]   PRVG-11827 : Name service returned IP addresses "192.168.100.70" for SCAN
INFO:  [Jan 7, 2025 6:28:08 PM]   name "rac-scan" not found in the DNS returned IP addresses ""
INFO:  [Jan 7, 2025 6:28:08 PM]   rac2: PRVF-4664 : Found inconsistent name resolution entries for SCAN name
INFO:  [Jan 7, 2025 6:28:08 PM]         "rac-scan"
INFO:  [Jan 7, 2025 6:28:08 PM]   rac1: PRVF-4664 : Found inconsistent name resolution entries for SCAN name
INFO:  [Jan 7, 2025 6:28:08 PM]         "rac-scan"
INFO:  [Jan 7, 2025 6:28:08 PM] CVU operation performed:      stage -post crsinst
INFO:  [Jan 7, 2025 6:28:08 PM] Date:                         Jan 7, 2025 6:27:20 PM
INFO:  [Jan 7, 2025 6:28:08 PM] CVU home:                     /u01/grid/19.3.0/
INFO:  [Jan 7, 2025 6:28:08 PM] User:                         grid
INFO:  [Jan 7, 2025 6:28:09 PM] Completed Plugin named: cvu
INFO:  [Jan 7, 2025 6:28:09 PM] Setup completed with overall status as Failed
```
![微信截图_20250107182946.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250107182946.png)
skip就行

#### 查看crs状态
```bash
[grid@rac1 ~]$ crsctl stat res -t 
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details       
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.LISTENER.lsnr
               ONLINE  ONLINE       rac1                     STABLE
               ONLINE  ONLINE       rac2                     STABLE
ora.chad
               ONLINE  ONLINE       rac1                     STABLE
               ONLINE  ONLINE       rac2                     STABLE
ora.net1.network
               ONLINE  ONLINE       rac1                     STABLE
               ONLINE  ONLINE       rac2                     STABLE
ora.ons
               ONLINE  ONLINE       rac1                     STABLE
               ONLINE  ONLINE       rac2                     STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.ASMNET1LSNR_ASM.lsnr(ora.asmgroup)
      1        ONLINE  ONLINE       rac1                     STABLE
      2        ONLINE  ONLINE       rac2                     STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.CRSDG.dg(ora.asmgroup)
      1        ONLINE  ONLINE       rac1                     STABLE
      2        ONLINE  ONLINE       rac2                     STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.LISTENER_SCAN1.lsnr
      1        ONLINE  ONLINE       rac1                     STABLE
ora.asm(ora.asmgroup)
      1        ONLINE  ONLINE       rac1                     Started,STABLE
      2        ONLINE  ONLINE       rac2                     Started,STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.asmnet1.asmnetwork(ora.asmgroup)
      1        ONLINE  ONLINE       rac1                     STABLE
      2        ONLINE  ONLINE       rac2                     STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.cvu
      1        ONLINE  ONLINE       rac1                     STABLE
ora.qosmserver
      1        ONLINE  ONLINE       rac1                     STABLE
ora.rac1.vip
      1        ONLINE  ONLINE       rac1                     STABLE
ora.rac2.vip
      1        ONLINE  ONLINE       rac2                     STABLE
ora.scan1.vip
      1        ONLINE  ONLINE       rac1                     STABLE
--------------------------------------------------------------------------------
```
### RDBMS安装

#### 解压安装包
```bash
[oracle@rac1 ~]$ unzip LINUX.X64_193000_db_home.zip -d $ORACLE_HOME
```

#### 启动安装向导
```bash
[oracle@rac1 ~]$ cd $ORACLE_HOME/
[oracle@rac1 db]$ ./runInstaller 
```

#### 向导流程
![微信截图_20250108103344.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108103344.png)
![微信截图_20250108103407.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108103407.png)
![微信截图_20250108111518.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108111518.png)
![微信截图_20250108111546.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108111546.png)

![微信截图_20250108111613.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108111613.png)
![微信截图_20250108111643.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108111643.png)
![微信截图_20250108111810.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108111810.png)
![微信截图_20250108111826.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108111826.png)
![微信截图_20250108111841.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108111841.png)
![微信截图_20250108111931.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108111931.png)
![微信截图_20250108112044.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108112044.png)
![微信截图_20250108112402.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108112402.png)

### 创建ASM磁盘组DATADG
![微信截图_20250108133728.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108133728.png)


### dbca


![微信截图_20250108170137.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170137.png)
![微信截图_20250108170252.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170252.png)
![微信截图_20250108170313.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170313.png)
![微信截图_20250108170352.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170352.png)
![微信截图_20250108170407.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170407.png)
![微信截图_20250108170433.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170433.png)
![微信截图_20250108170524.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170524.png)
![微信截图_20250108170539.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170539.png)
![微信截图_20250108170603.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170603.png)

![微信截图_20250108170609.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170609.png)
![微信截图_20250108170649.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170649.png)
![微信截图_20250108170709.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170709.png)
![微信截图_20250108170722.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170722.png)
![微信截图_20250108170812.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108170812.png)

## 补丁升级
自动打补丁会牵扯到**关库关集群**，**无需人为关库关集群**，请注意

**若建完实例后自动打补丁不需要将SQL****文件加载到数据库中**


### 检查grid和oracle opatch版本
因为是刚装的，就检查了一边。实际所有节点都要检查
```bash
[grid@rac1 OPatch]$ opatch version
OPatch Version: 12.2.0.1.17

OPatch succeeded.


[oracle@rac1 OPatch]$ opatch version
OPatch Version: 12.2.0.1.17

OPatch succeeded.
```

### 升级opatch
Patch 36878697: OJVM RELEASE UPDATE 19.25.0.0.0
Patch 36912597: DATABASE RELEASE UPDATE 19.25.0.0.0
Patch 36916690: GI RELEASE UPDATE 19.25.0.0.0
其中GI补丁包含
36912597	DATABASE RELEASE UPDATE 19.25.0.0.0	(Oracle Database)
36912597	DATABASE RELEASE UPDATE 19.25.0.0.0	(Oracle Database)
36912597	DATABASE RELEASE UPDATE 19.25.0.0.0	(Oracle Database)
36917416	OCW RELEASE UPDATE 19.25.0.0.0	(Oracle Database)
36912597	DATABASE RELEASE UPDATE 19.25.0.0.0	(Oracle Database)
36912597	DATABASE RELEASE UPDATE 19.25.0.0.0	(Oracle Database)
36912597	DATABASE RELEASE UPDATE 19.25.0.0.0	(Oracle Database)
36912597	DATABASE RELEASE UPDATE 19.25.0.0.0	(Oracle Database)
36917416	OCW RELEASE UPDATE 19.25.0.0.0	(Oracle Database)
36912597	DATABASE RELEASE UPDATE 19.25.0.0.0	(Oracle Database)
#### 备份原有opatch
两边都要执行
```bash
su - grid

[grid@rac1 ~]$ cp -a $ORACLE_HOME/OPatch ~/OPatch_20250108
[grid@rac1 ~]$ rm $ORACLE_HOME/OPatch
su - grid 

[oracle@rac1 OPatch]$ cp -a $ORACLE_HOME/OPatch ~/OPatch_20250108
[oracle@rac1 OPatch]$ rm $ORACLE_HOME/OPatch
```

#### 解压OPatch包到ORACLE_HOME
```bash
su - grid 
[grid@rac1 ~]$ unzip p6880880_190000_Linux-x86-64.zip -d $ORACLE_HOME
su - oracle 
[oracle@rac1 ~]$ unzip p6880880_190000_Linux-x86-64.zip -d $ORACLE_HOME
```

#### 查看升级结果
```bash
[grid@rac1 OPatch]$ opatch version
OPatch Version: 12.2.0.1.44

OPatch succeeded.


[oracle@rac1 OPatch]$ opatch version
OPatch Version: 12.2.0.1.44

OPatch succeeded.
```
### 升级RAC
两边都要升级
Patch 36916690: GI RELEASE UPDATE 19.25.0.0.0

#### 查看当前grid版本
```
[grid@rac1 OPatch]$ pwd
/u01/grid/19.3.0/OPatch
[grid@rac1 OPatch]$ ./opatch lspatches
29585399;OCW RELEASE UPDATE 19.3.0.0.0 (29585399)
29517247;ACFS RELEASE UPDATE 19.3.0.0.0 (29517247)
29517242;Database Release Update : 19.3.0.0.190416 (29517242)
29401763;TOMCAT RELEASE UPDATE 19.0.0.0.0 (29401763)

OPatch succeeded.
```

#### 检查补丁冲突
```bash
[grid@rac1 ~]$ unzip p36916690_190000_Linux-x86-64.zip

[grid@rac1 ~]$ $ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir 36916690/36758186/
Oracle Interim Patch Installer version 12.2.0.1.44
Copyright (c) 2025, Oracle Corporation.  All rights reserved.

PREREQ session

Oracle Home       : /u01/grid/19.3.0
Central Inventory : /u01/grid/oraInventory
   from           : /u01/grid/19.3.0/oraInst.loc
OPatch version    : 12.2.0.1.44
OUI version       : 12.2.0.7.0
Log file location : /u01/grid/19.3.0/cfgtoollogs/opatch/opatch2025-01-08_14-21-38PM_1.log

Invoking prereq "checkconflictagainstohwithdetail"

Prereq "checkConflictAgainstOHWithDetail" passed.

OPatch succeeded.
[grid@rac1 ~]$ $ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir 36916690/36912597/
Oracle Interim Patch Installer version 12.2.0.1.44
Copyright (c) 2025, Oracle Corporation.  All rights reserved.

PREREQ session

Oracle Home       : /u01/grid/19.3.0
Central Inventory : /u01/grid/oraInventory
   from           : /u01/grid/19.3.0/oraInst.loc
OPatch version    : 12.2.0.1.44
OUI version       : 12.2.0.7.0
Log file location : /u01/grid/19.3.0/cfgtoollogs/opatch/opatch2025-01-08_14-21-50PM_1.log

Invoking prereq "checkconflictagainstohwithdetail"

Prereq "checkConflictAgainstOHWithDetail" passed.

OPatch succeeded.
[grid@rac1 ~]$ $ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir 36916690/36917397/
Oracle Interim Patch Installer version 12.2.0.1.44
Copyright (c) 2025, Oracle Corporation.  All rights reserved.

PREREQ session

Oracle Home       : /u01/grid/19.3.0
Central Inventory : /u01/grid/oraInventory
   from           : /u01/grid/19.3.0/oraInst.loc
OPatch version    : 12.2.0.1.44
OUI version       : 12.2.0.7.0
Log file location : /u01/grid/19.3.0/cfgtoollogs/opatch/opatch2025-01-08_14-22-11PM_1.log

Invoking prereq "checkconflictagainstohwithdetail"

Prereq "checkConflictAgainstOHWithDetail" passed.

OPatch succeeded.
[grid@rac1 ~]$ $ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir 36916690/36917416
Oracle Interim Patch Installer version 12.2.0.1.44
Copyright (c) 2025, Oracle Corporation.  All rights reserved.

PREREQ session

Oracle Home       : /u01/grid/19.3.0
Central Inventory : /u01/grid/oraInventory
   from           : /u01/grid/19.3.0/oraInst.loc
OPatch version    : 12.2.0.1.44
OUI version       : 12.2.0.7.0
Log file location : /u01/grid/19.3.0/cfgtoollogs/opatch/opatch2025-01-08_14-22-17PM_1.log

Invoking prereq "checkconflictagainstohwithdetail"

Prereq "checkConflictAgainstOHWithDetail" passed.

OPatch succeeded.
[grid@rac1 ~]$ $ORACLE_HOME/OPatch/opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir 36916690/36940756
Oracle Interim Patch Installer version 12.2.0.1.44
Copyright (c) 2025, Oracle Corporation.  All rights reserved.

PREREQ session

Oracle Home       : /u01/grid/19.3.0
Central Inventory : /u01/grid/oraInventory
   from           : /u01/grid/19.3.0/oraInst.loc
OPatch version    : 12.2.0.1.44
OUI version       : 12.2.0.7.0
Log file location : /u01/grid/19.3.0/cfgtoollogs/opatch/opatch2025-01-08_14-22-24PM_1.log

Invoking prereq "checkconflictagainstohwithdetail"

Prereq "checkConflictAgainstOHWithDetail" passed.

OPatch succeeded.
```

#### 切换到root执行升级
节点1
```
[root@rac1 ~]# export PATH=$PATH:/u01/grid/19.3.0/OPatch
[root@rac1 ~]# cd /tmp/patch
[root@rac1 patch]# opatchauto apply /tmp/patch/36916690/

OPatchauto session is initiated at Wed Jan  8 12:23:43 2025

System initialization log file is /u01/grid/19.3.0/cfgtoollogs/opatchautodb/systemconfig2025-01-08_12-23-47PM.log.

Session log file is /u01/grid/19.3.0/cfgtoollogs/opatchauto/opatchauto2025-01-08_12-24-00PM.log
The id for this session is FPES

Executing OPatch prereq operations to verify patch applicability on home /u01/grid/19.3.0
Patch applicability verified successfully on home /u01/grid/19.3.0


Executing OPatch prereq operations to verify patch applicability on home /u01/oracle/product/19.3.0/db
Patch applicability verified successfully on home /u01/oracle/product/19.3.0/db


Executing patch validation checks on home /u01/grid/19.3.0
Patch validation checks successfully completed on home /u01/grid/19.3.0


Executing patch validation checks on home /u01/oracle/product/19.3.0/db
Patch validation checks successfully completed on home /u01/oracle/product/19.3.0/db


Verifying SQL patch applicability on home /u01/oracle/product/19.3.0/db
SQL patch applicability verified successfully on home /u01/oracle/product/19.3.0/db


Preparing to bring down database service on home /u01/oracle/product/19.3.0/db
Successfully prepared home /u01/oracle/product/19.3.0/db to bring down database service


Performing prepatch operations on CRS - bringing down CRS service on home /u01/grid/19.3.0
Prepatch operation log file location: /u01/grid/grid/crsdata/rac1/crsconfig/crs_prepatch_apply_inplace_rac1_2025-01-08_12-25-56AM.log
CRS service brought down successfully on home /u01/grid/19.3.0


Performing prepatch operation on home /u01/oracle/product/19.3.0/db
Prepatch operation completed successfully on home /u01/oracle/product/19.3.0/db


Start applying binary patch on home /u01/oracle/product/19.3.0/db
Binary patch applied successfully on home /u01/oracle/product/19.3.0/db


Running rootadd_rdbms.sh on home /u01/oracle/product/19.3.0/db
Successfully executed rootadd_rdbms.sh on home /u01/oracle/product/19.3.0/db


Performing postpatch operation on home /u01/oracle/product/19.3.0/db
Postpatch operation completed successfully on home /u01/oracle/product/19.3.0/db


Start applying binary patch on home /u01/grid/19.3.0
Binary patch applied successfully on home /u01/grid/19.3.0


Running rootadd_rdbms.sh on home /u01/grid/19.3.0
Successfully executed rootadd_rdbms.sh on home /u01/grid/19.3.0


Performing postpatch operations on CRS - starting CRS service on home /u01/grid/19.3.0
Postpatch operation log file location: /u01/grid/grid/crsdata/rac1/crsconfig/crs_postpatch_apply_inplace_rac1_2025-01-08_12-39-48AM.log
CRS service started successfully on home /u01/grid/19.3.0


Preparing home /u01/oracle/product/19.3.0/db after database service restarted
No step execution required.........
 

Trying to apply SQL patch on home /u01/oracle/product/19.3.0/db
No SQL patch operations are required on local node for this home

OPatchAuto successful.

--------------------------------Summary--------------------------------

Patching is completed successfully. Please find the summary as follows:

Host:rac1
RAC Home:/u01/oracle/product/19.3.0/db
Version:19.0.0.0.0
Summary:

==Following patches were SKIPPED:

Patch: /tmp/patch/36916690/36917397
Reason: This patch is not applicable to this specified target type - "rac_database"

Patch: /tmp/patch/36916690/36758186
Reason: This patch is not applicable to this specified target type - "rac_database"

Patch: /tmp/patch/36916690/36940756
Reason: This patch is not applicable to this specified target type - "rac_database"


==Following patches were SUCCESSFULLY applied:

Patch: /tmp/patch/36916690/36912597
Log: /u01/oracle/product/19.3.0/db/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-28-28PM_1.log

Patch: /tmp/patch/36916690/36917416
Log: /u01/oracle/product/19.3.0/db/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-28-28PM_1.log


Host:rac1
CRS Home:/u01/grid/19.3.0
Version:19.0.0.0.0
Summary:

==Following patches were SUCCESSFULLY applied:

Patch: /tmp/patch/36916690/36758186
Log: /u01/grid/19.3.0/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-34-40PM_1.log

Patch: /tmp/patch/36916690/36912597
Log: /u01/grid/19.3.0/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-34-40PM_1.log

Patch: /tmp/patch/36916690/36917397
Log: /u01/grid/19.3.0/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-34-40PM_1.log

Patch: /tmp/patch/36916690/36917416
Log: /u01/grid/19.3.0/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-34-40PM_1.log

Patch: /tmp/patch/36916690/36940756
Log: /u01/grid/19.3.0/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-34-40PM_1.log



OPatchauto session completed at Wed Jan  8 12:43:56 2025
Time taken to complete the session 20 minutes, 9 seconds
```


此时rac2上可以看到1节点关闭了在升级
![微信截图_20250108154030.png](https://picgo-1258062315.cos.ap-chengdu.myqcloud.com/image/%E5%BE%AE%E4%BF%A1%E6%88%AA%E5%9B%BE_20250108154030.png)



查看升级后版本
```bash
[grid@rac1 OPatch]$ opatch lspatches
36940756;TOMCAT RELEASE UPDATE 19.0.0.0.0 (36940756)
36917416;OCW RELEASE UPDATE 19.25.0.0.0 (36917416)
36917397;ACFS RELEASE UPDATE 19.25.0.0.0 (36917397)
36912597;Database Release Update : 19.25.0.0.241015 (36912597)
36758186;DBWLM RELEASE UPDATE 19.0.0.0.0 (36758186)

OPatch succeeded.

[oracle@rac1 OPatch]$ opatch lspatches
36917416;OCW RELEASE UPDATE 19.25.0.0.0 (36917416)
36912597;Database Release Update : 19.25.0.0.241015 (36912597)

OPatch succeeded.
```

节点2
```
[root@rac1 ~]# export PATH=$PATH:/u01/grid/19.3.0/OPatch
[root@rac1 ~]# cd /tmp/patch
[root@rac2 patch]# opatchauto apply /tmp/patch/36916690/

OPatchauto session is initiated at Wed Jan  8 12:46:26 2025

System initialization log file is /u01/grid/19.3.0/cfgtoollogs/opatchautodb/systemconfig2025-01-08_12-46-31PM.log.

Session log file is /u01/grid/19.3.0/cfgtoollogs/opatchauto/opatchauto2025-01-08_12-46-44PM.log
The id for this session is DAIR

Executing OPatch prereq operations to verify patch applicability on home /u01/grid/19.3.0
Patch applicability verified successfully on home /u01/grid/19.3.0


Executing OPatch prereq operations to verify patch applicability on home /u01/oracle/product/19.3.0/db
Patch applicability verified successfully on home /u01/oracle/product/19.3.0/db


Executing patch validation checks on home /u01/grid/19.3.0
Patch validation checks successfully completed on home /u01/grid/19.3.0


Executing patch validation checks on home /u01/oracle/product/19.3.0/db
Patch validation checks successfully completed on home /u01/oracle/product/19.3.0/db


Verifying SQL patch applicability on home /u01/oracle/product/19.3.0/db
SQL patch applicability verified successfully on home /u01/oracle/product/19.3.0/db


Preparing to bring down database service on home /u01/oracle/product/19.3.0/db
Successfully prepared home /u01/oracle/product/19.3.0/db to bring down database service


Performing prepatch operations on CRS - bringing down CRS service on home /u01/grid/19.3.0
Prepatch operation log file location: /u01/grid/grid/crsdata/rac2/crsconfig/crs_prepatch_apply_inplace_rac2_2025-01-08_12-49-23AM.log
CRS service brought down successfully on home /u01/grid/19.3.0


Performing prepatch operation on home /u01/oracle/product/19.3.0/db
Prepatch operation completed successfully on home /u01/oracle/product/19.3.0/db


Start applying binary patch on home /u01/oracle/product/19.3.0/db
Binary patch applied successfully on home /u01/oracle/product/19.3.0/db


Running rootadd_rdbms.sh on home /u01/oracle/product/19.3.0/db
Successfully executed rootadd_rdbms.sh on home /u01/oracle/product/19.3.0/db


Performing postpatch operation on home /u01/oracle/product/19.3.0/db
Postpatch operation completed successfully on home /u01/oracle/product/19.3.0/db


Start applying binary patch on home /u01/grid/19.3.0
Binary patch applied successfully on home /u01/grid/19.3.0


Running rootadd_rdbms.sh on home /u01/grid/19.3.0
Successfully executed rootadd_rdbms.sh on home /u01/grid/19.3.0


Performing postpatch operations on CRS - starting CRS service on home /u01/grid/19.3.0
Postpatch operation log file location: /u01/grid/grid/crsdata/rac2/crsconfig/crs_postpatch_apply_inplace_rac2_2025-01-08_01-03-11PM.log
CRS service started successfully on home /u01/grid/19.3.0


Preparing home /u01/oracle/product/19.3.0/db after database service restarted
No step execution required.........
 

Trying to apply SQL patch on home /u01/oracle/product/19.3.0/db
SQL patch applied successfully on home /u01/oracle/product/19.3.0/db

OPatchAuto successful.

--------------------------------Summary--------------------------------

Patching is completed successfully. Please find the summary as follows:

Host:rac2
RAC Home:/u01/oracle/product/19.3.0/db
Version:19.0.0.0.0
Summary:

==Following patches were SKIPPED:

Patch: /tmp/patch/36916690/36917397
Reason: This patch is not applicable to this specified target type - "rac_database"

Patch: /tmp/patch/36916690/36758186
Reason: This patch is not applicable to this specified target type - "rac_database"

Patch: /tmp/patch/36916690/36940756
Reason: This patch is not applicable to this specified target type - "rac_database"


==Following patches were SUCCESSFULLY applied:

Patch: /tmp/patch/36916690/36912597
Log: /u01/oracle/product/19.3.0/db/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-51-49PM_1.log

Patch: /tmp/patch/36916690/36917416
Log: /u01/oracle/product/19.3.0/db/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-51-49PM_1.log


Host:rac2
CRS Home:/u01/grid/19.3.0
Version:19.0.0.0.0
Summary:

==Following patches were SUCCESSFULLY applied:

Patch: /tmp/patch/36916690/36758186
Log: /u01/grid/19.3.0/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-57-40PM_1.log

Patch: /tmp/patch/36916690/36912597
Log: /u01/grid/19.3.0/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-57-40PM_1.log

Patch: /tmp/patch/36916690/36917397
Log: /u01/grid/19.3.0/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-57-40PM_1.log

Patch: /tmp/patch/36916690/36917416
Log: /u01/grid/19.3.0/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-57-40PM_1.log

Patch: /tmp/patch/36916690/36940756
Log: /u01/grid/19.3.0/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_12-57-40PM_1.log



OPatchauto session completed at Wed Jan  8 13:26:51 2025
Time taken to complete the session 40 minutes, 21 seconds
```

查看升级后的版本
```
[grid@rac2 OPatch]$ opatch lspatches
36940756;TOMCAT RELEASE UPDATE 19.0.0.0.0 (36940756)
36917416;OCW RELEASE UPDATE 19.25.0.0.0 (36917416)
36917397;ACFS RELEASE UPDATE 19.25.0.0.0 (36917397)
36912597;Database Release Update : 19.25.0.0.241015 (36912597)
36758186;DBWLM RELEASE UPDATE 19.0.0.0.0 (36758186)

OPatch succeeded.

[oracle@rac2 OPatch]$ opatch lspatches
36917416;OCW RELEASE UPDATE 19.25.0.0.0 (36917416)
36912597;Database Release Update : 19.25.0.0.241015 (36912597)

OPatch succeeded.
```
### 查看升级结果
#### 节点1 grid和rdbms
```bash
[grid@rac1 OPatch]$ opatch lspatches
36940756;TOMCAT RELEASE UPDATE 19.0.0.0.0 (36940756)
36917416;OCW RELEASE UPDATE 19.25.0.0.0 (36917416)
36917397;ACFS RELEASE UPDATE 19.25.0.0.0 (36917397)
36912597;Database Release Update : 19.25.0.0.241015 (36912597)
36758186;DBWLM RELEASE UPDATE 19.0.0.0.0 (36758186)

OPatch succeeded.

[oracle@rac1 OPatch]$ opatch lspatches
36917416;OCW RELEASE UPDATE 19.25.0.0.0 (36917416)
36912597;Database Release Update : 19.25.0.0.241015 (36912597)

OPatch succeeded.
```
#### 节点2 grid和rdbms
```
[grid@rac2 OPatch]$ opatch lspatches
36940756;TOMCAT RELEASE UPDATE 19.0.0.0.0 (36940756)
36917416;OCW RELEASE UPDATE 19.25.0.0.0 (36917416)
36917397;ACFS RELEASE UPDATE 19.25.0.0.0 (36917397)
36912597;Database Release Update : 19.25.0.0.241015 (36912597)
36758186;DBWLM RELEASE UPDATE 19.0.0.0.0 (36758186)

OPatch succeeded.

[oracle@rac2 OPatch]$ opatch lspatches
36917416;OCW RELEASE UPDATE 19.25.0.0.0 (36917416)
36912597;Database Release Update : 19.25.0.0.241015 (36912597)

OPatch succeeded.
```

#### db
```sql
SQL> select * from v$version;
SQL> select * from v$version;

BANNER
--------------------------------------------------------------------------------
BANNER_FULL
--------------------------------------------------------------------------------
BANNER_LEGACY
--------------------------------------------------------------------------------
    CON_ID
----------
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.25.0.0.0
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
         0

```
## 报错
### 安装补丁提示缺少fuser

```bash
[root@rac1 OPatch]# opatchauto apply /home/grid/36916690/

OPatchauto session is initiated at Wed Jan  8 14:28:14 2025

System initialization log file is /u01/grid/19.3.0/cfgtoollogs/opatchautodb/systemconfig2025-01-08_02-28-17PM.log.

Session log file is /u01/grid/19.3.0/cfgtoollogs/opatchauto/opatchauto2025-01-08_02-28-23PM.log
The id for this session is EYE3

Executing OPatch prereq operations to verify patch applicability on home /u01/grid/19.3.0
Patch applicability verification failed on home /u01/grid/19.3.0

Execution of [OPatchAutoBinaryAction] patch action failed, check log for more details. Failures:
Patch Target : rac1->/u01/grid/19.3.0 Type[crs]
Details: [
---------------------------Patching Failed---------------------------------
Command execution failed during patching in home: /u01/grid/19.3.0, host: rac1.
Command failed:  /u01/grid/19.3.0/OPatch/opatchauto  apply /home/grid/36916690/ -oh /u01/grid/19.3.0 -target_type cluster -binary -invPtrLoc /u01/grid/19.3.0/oraInst.loc -jre /u01/grid/19.3.0/OPatch/jre -persistresult /u01/grid/19.3.0/opatchautocfg/db/sessioninfo/sessionresult_analyze_rac1_crs_1.ser -analyze -online -prepare_home
Command failure output: 
==Following patches FAILED in analysis for apply:

Patch: /home/grid/36916690/36912597
Log: /u01/grid/19.3.0/cfgtoollogs/opatchauto/core/opatch/opatch2025-01-08_14-28-36PM_1.log
Reason: Failed during Analysis: CheckSystemCommandsAvailable Failed, [ Prerequisite Status: FAILED, Prerequisite output: 
The details are:
Missing command :fuser] 


After fixing the cause of failure start a new opatchauto session
]
OPATCHAUTO-68061: The orchestration engine failed.
OPATCHAUTO-68061: The orchestration engine failed with return code 1
OPATCHAUTO-68061: Check the log for more details.
OPatchAuto failed.
bash: zip: command not found
opatchauto unable to backup /u01/grid/19.3.0/cfgtoollogs/opatchauto
bash: zip: command not found
opatchauto unable to backup /u01/grid/19.3.0/cfgtoollogs/opatchautodb

OPatchauto session completed at Wed Jan  8 14:29:01 2025
Time taken to complete the session 0 minute, 44 seconds

 opatchauto failed with error code 42
```
安装fuser
```bash
yum --disablerepo=* --enablerepo=c7-media install  psmisc -y 
```

### 节点2升级oracle出错
```bash
[oracle@rac2 36912597]$ $ORACLE_HOME/OPatch/opatch apply
Oracle Interim Patch Installer version 12.2.0.1.44
Copyright (c) 2025, Oracle Corporation.  All rights reserved.


Oracle Home       : /u01/oracle/product/19.3.0/db
Central Inventory : /u01/grid/oraInventory
   from           : /u01/oracle/product/19.3.0/db/oraInst.loc
OPatch version    : 12.2.0.1.44
OUI version       : 12.2.0.7.0
Log file location : /u01/oracle/product/19.3.0/db/cfgtoollogs/opatch/opatch2025-01-08_16-26-22PM_1.log

Verifying environment and performing prerequisite checks...
OPatch continues with these patches:   36912597  

Do you want to proceed? [y|n]
y
User Responded with: Y
All checks passed.

Please shutdown Oracle instances running out of this ORACLE_HOME on the local system.
(Oracle Home = '/u01/oracle/product/19.3.0/db')


Is the local system ready for patching? [y|n]
y
User Responded with: Y
Backing up files...
Applying interim patch '36912597' to OH '/u01/oracle/product/19.3.0/db'
ApplySession: Optional component(s) [ oracle.network.gsm, 19.0.0.0.0 ] , [ oracle.crypto.rsf, 19.0.0.0.0 ] , [ oracle.pg4appc, 19.0.0.0.0 ] , [ oracle.pg4mq, 19.0.0.0.0 ] , [ oracle.precomp.companion, 19.0.0.0.0 ] , [ oracle.rdbms.ic, 19.0.0.0.0 ] , [ oracle.rdbms.tg4db2, 19.0.0.0.0 ] , [ oracle.tfa, 19.0.0.0.0 ] , [ oracle.sdo.companion, 19.0.0.0.0 ] , [ oracle.net.cman, 19.0.0.0.0 ] , [ oracle.oid.client, 19.0.0.0.0 ] , [ oracle.xdk.companion, 19.0.0.0.0 ] , [ oracle.options.olap.api, 19.0.0.0.0 ] , [ oracle.ons.eons.bwcompat, 19.0.0.0.0 ] , [ oracle.rdbms.tg4msql, 19.0.0.0.0 ] , [ oracle.network.cman, 19.0.0.0.0 ] , [ oracle.rdbms.tg4tera, 19.0.0.0.0 ] , [ oracle.rdbms.tg4ifmx, 19.0.0.0.0 ] , [ oracle.rdbms.tg4sybs, 19.0.0.0.0 ] , [ oracle.ldap.ztk, 19.0.0.0.0 ] , [ oracle.ons.cclient, 19.0.0.0.0 ] , [ oracle.options.olap, 19.0.0.0.0 ] , [ oracle.jdk, 1.8.0.191.0 ] , [ oracle.jdk, 1.8.0.391.11 ]  not present in the Oracle Home or a higher version is found.

Patching component oracle.rdbms, 19.0.0.0.0...

Patching component oracle.rdbms.util, 19.0.0.0.0...

Patching component oracle.rdbms.rsf, 19.0.0.0.0...

Patching component oracle.assistants.acf, 19.0.0.0.0...

Patching component oracle.assistants.deconfig, 19.0.0.0.0...

Patching component oracle.assistants.server, 19.0.0.0.0...

Patching component oracle.blaslapack, 19.0.0.0.0...

Patching component oracle.buildtools.rsf, 19.0.0.0.0...

Patching component oracle.ctx, 19.0.0.0.0...

Patching component oracle.dbdev, 19.0.0.0.0...

Patching component oracle.dbjava.ic, 19.0.0.0.0...

Patching component oracle.dbjava.jdbc, 19.0.0.0.0...

Patching component oracle.dbjava.ucp, 19.0.0.0.0...

Patching component oracle.duma, 19.0.0.0.0...

Patching component oracle.javavm.client, 19.0.0.0.0...

Patching component oracle.ldap.owm, 19.0.0.0.0...

Patching component oracle.ldap.rsf, 19.0.0.0.0...

Patching component oracle.ldap.security.osdt, 19.0.0.0.0...

Patching component oracle.marvel, 19.0.0.0.0...

Patching component oracle.network.rsf, 19.0.0.0.0...

Patching component oracle.odbc.ic, 19.0.0.0.0...

Patching component oracle.ons, 19.0.0.0.0...

Patching component oracle.ons.ic, 19.0.0.0.0...

Patching component oracle.oracore.rsf, 19.0.0.0.0...

Patching component oracle.perlint, 5.28.1.0.0...

Patching component oracle.precomp.common.core, 19.0.0.0.0...

Patching component oracle.precomp.rsf, 19.0.0.0.0...

Patching component oracle.rdbms.crs, 19.0.0.0.0...

Patching component oracle.rdbms.dbscripts, 19.0.0.0.0...

Patching component oracle.rdbms.deconfig, 19.0.0.0.0...

Patching component oracle.rdbms.oci, 19.0.0.0.0...

Patching component oracle.rdbms.rsf.ic, 19.0.0.0.0...

Patching component oracle.rdbms.scheduler, 19.0.0.0.0...

Patching component oracle.rhp.db, 19.0.0.0.0...

Patching component oracle.rsf, 19.0.0.0.0...

Patching component oracle.sdo, 19.0.0.0.0...

Patching component oracle.sdo.locator.jrf, 19.0.0.0.0...

Patching component oracle.sqlplus, 19.0.0.0.0...

Patching component oracle.sqlplus.ic, 19.0.0.0.0...

Patching component oracle.wwg.plsql, 19.0.0.0.0...

Patching component oracle.xdk.rsf, 19.0.0.0.0...

Patching component oracle.javavm.server, 19.0.0.0.0...

Patching component oracle.xdk.xquery, 19.0.0.0.0...

Patching component oracle.ctx.rsf, 19.0.0.0.0...

Patching component oracle.ovm, 19.0.0.0.0...

Patching component oracle.oraolap, 19.0.0.0.0...

Patching component oracle.nlsrtl.rsf.lbuilder, 19.0.0.0.0...

Patching component oracle.rdbms.rat, 19.0.0.0.0...

Patching component oracle.ldap.rsf.ic, 19.0.0.0.0...

Patching component oracle.rdbms.dv, 19.0.0.0.0...

Patching component oracle.xdk, 19.0.0.0.0...

Patching component oracle.mgw.common, 19.0.0.0.0...

Patching component oracle.ldap.client, 19.0.0.0.0...

Patching component oracle.install.deinstalltool, 19.0.0.0.0...

Patching component oracle.rdbms.rman, 19.0.0.0.0...

Patching component oracle.oraolap.api, 19.0.0.0.0...

Patching component oracle.dbtoolslistener, 19.0.0.0.0...

Patching component oracle.rdbms.drdaas, 19.0.0.0.0...

Patching component oracle.rdbms.hs_common, 19.0.0.0.0...

Patching component oracle.rdbms.lbac, 19.0.0.0.0...

Patching component oracle.sdo.locator, 19.0.0.0.0...

Patching component oracle.rdbms.dm, 19.0.0.0.0...

Patching component oracle.ldap.ssl, 19.0.0.0.0...

Patching component oracle.xdk.parser.java, 19.0.0.0.0...

Patching component oracle.odbc, 19.0.0.0.0...

Patching component oracle.network.listener, 19.0.0.0.0...

Patching component oracle.ctx.atg, 19.0.0.0.0...

Patching component oracle.rdbms.install.common, 19.0.0.0.0...

Patching component oracle.rdbms.hsodbc, 19.0.0.0.0...

Patching component oracle.network.aso, 19.0.0.0.0...

Patching component oracle.rdbms.locator, 19.0.0.0.0...

Patching component oracle.rdbms.install.plugins, 19.0.0.0.0...

Patching component oracle.nlsrtl.rsf, 19.0.0.0.0...

Patching component oracle.nlsrtl.rsf.core, 19.0.0.0.0...

Patching component oracle.nlsrtl.rsf.ic, 19.0.0.0.0...

Patching component oracle.oraolap.dbscripts, 19.0.0.0.0...

Patching component oracle.network.client, 19.0.0.0.0...

Patching component oracle.precomp.common, 19.0.0.0.0...

Patching component oracle.precomp.lang, 19.0.0.0.0...

Patching component oracle.jdk, 1.8.0.201.0...
ApplySession failed in system modification phase... 'ApplySession::apply failed: java.io.IOException: oracle.sysman.oui.patch.PatchException: java.io.FileNotFoundException: /u01/grid/oraInventory/ContentsXML/oui-patch.xml (Permission denied)'

Restoring "/u01/oracle/product/19.3.0/db" to the state prior to running NApply...
OPatch failed to restore OH '/u01/oracle/product/19.3.0/db'. Consult OPatch document to restore the home manually before proceeding.

NApply was not able to restore the home.  Please invoke the following scripts:
  - restore.[sh,bat]
  - make.txt (Unix only)
to restore the ORACLE_HOME.  They are located under 
"/u01/oracle/product/19.3.0/db/.patch_storage/NApply/2025-01-08_16-26-22PM"

UtilSession failed: ApplySession failed in system modification phase... 'ApplySession::apply failed: java.io.IOException: oracle.sysman.oui.patch.PatchException: java.io.FileNotFoundException: /u01/grid/oraInventory/ContentsXML/oui-patch.xml (Permission denied)'
Log file location: /u01/oracle/product/19.3.0/db/cfgtoollogs/opatch/opatch2025-01-08_16-26-22PM_1.log

OPatch failed with error code 73
```

查看了一下文件`/u01/grid/oraInventory/ContentsXML/oui-patch.xml`，发现属组是oracle:oinstall，但节点1上是grid:oinstall，修改一下
```bash
chown grid:oinstall /u01/grid/oraInventory/ContentsXML/oui-patch.xml 
```

然后回退更新，到目录/u01/oracle/product/19.3.0/db/.patch_storage/NApply/2025-01-08_16-26-22PM下执行restore.sh
```
[oracle@rac2 36912597]$ cd /u01/oracle/product/19.3.0/db/.patch_storage/NApply/2025-01-08_16-26-22PM
[oracle@rac2 2025-01-08_16-26-22PM]$ ls
backup  make.txt  patchlist.txt  restore.sh
[oracle@rac2 2025-01-08_16-26-22PM]$ ./restore.sh 
This script is going to restore the Oracle Home to the previous state.
It does not perform any of the following:
- Running init/pre/post scripts
- Oracle binary re-link
- Customized steps performed manually by user
Please use this script with supervision from Oracle Technical Support.
About to modify Oracle Home( /u01/oracle/product/19.3.0/db )
Do you want to proceed? [Y/N]
y
User responded with : Y
./restore.sh: line 53: cd: /u01/oracle/product/19.3.0/db/inventory/patches: No such file or directory
copy apex
assistants
bin
client
ctx
deinstall
dmu
dv
install
inventory
javavm
jdbc
jdk
jlib
ldap
lib
md
mgw
network
nls
odbc
olap
opmn
oracore
ords
owm
perl
plsql
precomp
rdbms
sdk
sdo
sqlcl
sqldeveloper
sqlpatch
sqlplus
suptools
ucp
wwg
xdk to /u01/oracle/product/19.3.0/db
```

再重新执行更新

### 无法解压
```bash
[root@rac2 patch]# opatchauto apply /tmp/patch/36916690/

OPatchauto session is initiated at Wed Jan  8 12:46:06 2025
OPATCHAUTO-72083: Performing bootstrap operations failed.
OPATCHAUTO-72083: The bootstrap execution failed because Failed to unzip files on path./tmp/patch/36916690/36912597/files/perl.zipError::.
OPATCHAUTO-72083: Fix the reported problem and re-run opatchauto. In case of standalone SIDB installation and Grid is not installed re-run with -sidb option.
com.oracle.glcm.patch.auto.OPatchAutoException: OPATCHAUTO-72083: Performing bootstrap operations failed.
OPATCHAUTO-72083: The bootstrap execution failed because Failed to unzip files on path./tmp/patch/36916690/36912597/files/perl.zipError::.
OPATCHAUTO-72083: Fix the reported problem and re-run opatchauto. In case of standalone SIDB installation and Grid is not installed re-run with -sidb option.
        at com.oracle.glcm.patch.auto.db.util.BootstrapHandler.performBootstrapping(BootstrapHandler.java:542)
        at com.oracle.glcm.patch.auto.db.util.BootstrapHandler.main(BootstrapHandler.java:98)

OPatchauto session completed at Wed Jan  8 12:46:08 2025
Time taken to complete the session 0 minute, 2 seconds

opatchauto bootstrapping failed with error code 255.
```
实际上需要把`/tmp/patch/36916690/`属组设置为grid:oinstall
```bash
chown -R grid:oinstall  /tmp/patch/36916690/
```