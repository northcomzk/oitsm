Oracle 11gR2 安装手册

[TOC]

# [简介](https://docs.oracle.com/cd/B28359_01/server.111/b28301/intro.htm#ADMQS0111)


主要后台进程

- PMON --Process Monitor Process 进程监控进程
- SMON --System Monitor Process 系统监控进程
- DBWn --Database Write Process 数据库写进程
- LGWR --Log Write Process 日志写进程
- CKPT --Checkpoint Process 检查点进程

# 配置流程

```flow
st=>start: 开始
op0=>operation: 调整系统环境参数
op1=>operation: 安装Oracle数据库软件
op2=>operation: 准备ASM裸磁盘
op3=>operation: 建立CSS进程
op4=>operation: 标记磁盘
op5=>operation: 创建ASM
op6=>operation: 安装数据库软件dbca
op7=>operation: 建数据库、配置监听
op8=>operation: 收尾：配置自启动
e=>end: 结束

st(right)->op0->op1->op2(right)->op3(right)->op4(right)->op5->op6(right)->op7(right)->op8->e
```

# linux 配置及优化

## Root 用户执行

 修改内核参数、增加用户、组、赋权、配置防火墙及开机启动项

``` sh
echo "192.168.111.104 ocplhr" >> /etc/hosts
hostnamectl set-hostname ocplhr
yum install -y binutils compat-libstdc++-33 elfutils-libelf elfutils-libelf-devel glibc glibc-common glibc-devel gcc gcc-c++ libaio-devel libaio libgcc libstdc++ libstdc++-devel make sysstat unixODBC unixODBC-devel ksh numactl-devel zip unzip elfutils-libelf-devel tree nmap sysstat lrzsz dos2unix screen iptraf iotop epel-release htop /dev/null
cat >> /etc/sysctl.conf <<EOF
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmmax = 4294967296
kernel.shmall = 4194304
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
vm.min_free_kbytes=512000
vm.vfs_cache_pressure=150
vm.swappiness=0
EOF
sysctl -p
cat >> /etc/security/limits.conf <<EOF
oracle soft nproc   16384
oracle hard nproc   16384
oracle soft nofile 65535
oracle hard nofile 65535
oracle soft stack 10240
oracle hard stack 10240
EOF
cat >> /etc/pam.d/login <<EOF
session required /lib/security/pam_limits.so
session required pam_limits.so
EOF
cat >> /etc/profile <<EOF
if [ $USER = "oracle" ]; then
if [ $SHELL = "/bin/ksh" ]; then
ulimit -p 16384
ulimit -n 65536
else
ulimit -u 16384 -n 65536
fi
fi
EOF
groupadd oinstall
groupadd dba
useradd -g oinstall -G dba -m oracle
echo oracle|passwd --stdin oracle
mkdir -p /u01/app/oracle/product/11.2.0/dbhome_1
mkdir -p /u01/app/oracle/oradata
mkdir -p /u01/app/oraInventory
mkdir -p /u01/app/oracle/fast_recovery_area
mkdir -p /soft
chown -R oracle:oinstall /u01/app/oracle
chown -R oracle:oinstall /u01/app/oraInventory
chown -R oracle:oinstall /soft/database
chmod -R 777 /soft
chmod -R 755 /u01/app/oracle
chmod -R 755 /u01/app/oraInventory
systemctl stop  firewalld
systemctl disable firewalld
systemctl stop avahi-daemon
systemctl disable avahi-daemon
systemctl stop  cups
systemctl disable cups
systemctl stop  postfix
systemctl disable postfix
systemctl stop smartd
systemctl disable smartd
systemctl stop NetworkManager
systemctl disable NetworkManager
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
setenforce 0
```

## oracle 用户执行

设置用户环境变量，安装数据库，设置监听、自动启动、复制安装文件至/soft

``` sh
su - oracle
cd /home/oracle
cat >> .bash_profile <<EOF
PATH=$PATH:$HOME/.local/bin:$HOME/bin
export ORACLE_BASE=/u01/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/11.2.0/dbhome_1
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export ORACLE_SID=ocplhr
export PATH=$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:$PATH:.
export NLS_LANG=AMERICAN_AMERICA.ZHS16GBK
export NLS_DATE_FORMAT="YYYY-MM-DD HH24:MI:SS"
export SQLPATH=$ORACLE_HOME/sqlplus/admin
export TNS_ADMIN=$ORACLE_HOME/network/admin
export ORACLE_PATH=.:$ORACLE_BASE/dba_scripts/sql:$ORACLE_HOME/rdbms/admin
alias sqlplus='rlwrap sqlplus'
alias rman='rlwrap rman'
alias asmcmd='rlwrap asmcmd'
alias sas='sqlplus / as sysdba'
EOF
source .bash_profile
```

# 安装

## 安装数据库程序

复制安装文件至/soft 目录、解压、安装数据库软件。

``` sh
unzip /soft/'p13390677_112040_Linux-x86-64_*.zip'
/soft/database/runInstaller -silent -force -noconfig -IgnoreSysPreReqs -ignorePrereq -showProgress \
oracle.install.option=INSTALL_DB_SWONLY \
DECLINE_SECURITY_UPDATES=true \
UNIX_GROUP_NAME=oinstall \
INVENTORY_LOCATION=/u01/app/oraInventory \
SELECTED_LANGUAGES=en \
ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1 \
ORACLE_BASE=/u01/app/oracle \
oracle.install.db.InstallEdition=EE \
oracle.install.db.isCustomInstall=false \
oracle.install.db.DBA_GROUP=dba \
oracle.install.db.OPER_GROUP=dba \
oracle.install.db.isRACOneInstall=false \
oracle.install.db.config.starterdb.type=GENERAL_PURPOSE
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false \
oracle.installer.autoupdates.option=SKIP_UPDATES

sh /u01/app/oraInventory/orainstRoot.sh
sh /u01/app/oracle/product/11.2.0/dbhome_1/root.sh
```

## 建库

``` sh
dbca -silent -createDatabase -templateName General_Purpose.dbc -responseFile NO_VALUE \
-gdbname ocplhr -sid ocplhr \
-sysPassword oracle -systemPassword oracle \
-datafileDestination '/u01/app/oracle/oradata' \
-recoveryAreaDestination '/u01/app/oracle/flash_recovery_area' \
-storageType FS \
-characterset AL32UTF8 -nationalCharacterSet AL16UTF16 \
-sampleSchema true \
-memoryPercentage 10 \
-databaseType OLTP \
-emConfiguration NONE
```

## 配置监听

``` sh
netca /silent /responseFile /home/oracle/database/response/netca.rsp
ls $ORACLE_HOME/network/admin/    #正常情况下会自动生成listener.ora sqlnet.ora

lsnrctl status
```

## 配置开机自动启动

配置/etc/oratab

``` sh
[root@oracle ~]#vim /etc/oratab
orcl:/u01/app/oracle/product/11.2.0/dbhome_1:Y #将N改为Y
```

配置数据库启停脚本

``` sh
[root@oracle ~]#vim $ORACLE_HOME/bin/dbstart +80
[root@oracle ~]#vim $ORACLE_HOME/bin/dbshut +50
ORACLE_HOME_LISTNER=$ORACLE_HOME （$1改$ORACLE_HOME）
ORACLE_HOME_LISTNER 的位置：Oracle 11g 的 dbstart 在第 80 行，dbshut 文件中在第 50 行。
```

建立启动脚本

> /etc/rc.d/rc.local 是系统自启动项目之外，需要系统启动的，统一在这里设置。-l 表示同时切换用户目录。－c 则表示执行完命令好再返回到原来的用户。

``` sh
[root@ocplhr ~]# vim /etc/rc.d/rc.local
touch /lock/subsys/local
su - oracle -lc "/u01/app/oracle/product/11.2.0/dbhome_1/bin/lsnrctl start"
su - oracle -lc "/u01/app/oracle/product/11.2.0/dbhome_1/bin/dbstart"

chmod +x /etc/rc.d/rc.local
```

重启 OS，验证是否生效

``` sh
lsnrctl status LISTENER
ps -aux | grep oracle
```

配置 spfile

``` sh
cp /u01/app/oracle/admin/OCPLHR/pfile/init.ora.1132021222827 /u01/app/oracle/product/11.2.0/dbhome_1/dbs/initocplhr.ora
```

安装 rlwrap

``` sh
yum install -y gcc yum install -y libtermcap-devel yum install -y readline yum install -y readline-devel
tar -zxvf rlwrap-0.37.tar.gz
cd rlwrap-0.37
./configure
make install ldconfig
```

测试

``` sql
sqlplus / as sysdba
select status from v$instance;
```

# 导入导出

## 导出
在源数据库内执行

``` sh
create directory dmpdir as '/expdump';     # /expdump  是目标库挂在在源库的转储文件目录
```

在源库操作系统执行

``` sh
expdp \'/ as sysdba\' directory=dmpdir schemas=DRM_SIMIS,YB_OLD_SL,SIHIS,DRM_SIHIS,YB_OLD_LH,ERP,YB_OLD_GZL,YB_OLD_LS,SIHIS_TMP,SPSIMIS,SPSIMIS1,IMPORT,SIHIS1,DRM_SIMIS1,SPSIMIS_CARD,YB_OLD_SP,DRM_SIMIS_TEST,DRM_SIHIS_TEST,LNSICW,SYYH,SSJK,BACKUPUSER,ORACLE_OCM,DRM_SIMIS_CP,CPSIMIS,SPYHJK,BI_XX,DSS_XX,SP_WEB,SPMIS,SBJXXC,DSS_SP_DWH,DSS_SP_UNIEAP,DSS_SP_REPORT,DSS_SP_BIFILE,DRM_SIMIS_TOMCAT,SGMIS,DRM_SGMIS,SYBJC,SPYD,SPYHJK1,WZJ,SPSI_SSJK,YLBX_CC,YLBX_GSSJZH,DSG,YUNHIS_WH,EXECSQL,SPSI,SPCHS,SWJK,SJHL,JLSI,SJZH dumpfile=full20201223.dmp logfile=full20201223.log;   #full%U.dmp生成序列号
```

## 导入
在目标库内执行

``` sql
create directory dmpdir as '/expdump';
```

在目标库操作系统执行导入

``` sh
impdp  \'/ as sysdba\' directory=dmpdir  dumpfile=full20201223.dmp logfile=full20201223.log;
```

# 软件卸载

---

# grid 安装

[Oracle 官网文档](https://docs.oracle.com/cd/B28359_01/server.111/b31107/asmcon.htm#OSTMG036)

## 简介

Gird 概述

- Grid 中文意思为网格，从 oracle10g、11g 后面的 g 便是 grid 的代称
- 集群分为高可用集群和负载均衡集群（oracle 的 clusterware，微软的 MSCS）
- 10g 中 grid 需要使用的软件单独存在，11g 后正好到一起构成 Grid Infrastructure(GI)
- GI 主要包括 clusterware 和 ASM，11gR2 开始，需要单独下载
- 11gR2 之前 ASM 功能通过 dbca 实现，之后使用 asmca 实现。

ASM 概述

- ASM（自动存储管理）提供的卷管理器.
- 替代 LVM
- 支持单实例和 RAC。
- 平台无关的文件系统、逻辑卷管理以及软 RAID 服务。
- 支持条带化和磁盘镜像，在数据库被加载的情况下添加或移除磁盘
- 自动平衡 I/O 以删除“热点” 支持直接和异步的 I/O
- 支持第三方多路径管理管理软件
- 使用 OMF 管理文件


ASM 进程介绍

- 除了传统的 DBWR,LGWR,CKPT,SMON,PMON 等进程还包含如下四个新后台进程：
- RBAL：负责协调磁盘组的负载均衡
- ARB0-ARBn：在同一时刻可以存在许多此类进程，执行实际的负载均衡。
- GMON：用于 ASM 磁盘组监控
- O0nn 01-10：这组进程建立到 ASM 实例的连接
- ASMB 与 ASM 实例的前台进程连接，周期性的检查两个 instance 的健康状况。
- 每个数据库实例同时只能与一个 ASM 实例连接。
- RBAL 用来进行全局调用，以打开某个磁盘组内的磁盘。
- CSS 集群同步服务。要使用 ASM，必须确保已经运行了 CSS 集群同步服务，CSS 负责 ASM 实例和数据库实例之间的同步。

块设备、裸设备区别

- 块设备：系统可以随机读写数据的设备，比如硬盘，通过 OS 缓冲后，可以被 mount。Linux 默认只提供对磁盘设备的块设备访问方式(比如/dev/sda1）
- 裸设备：也叫裸分区（原始分区）是一种没有经过格式化，不被 Unix/Linux 通过文件系统来读取的特殊字符设备，不被缓冲，直接写入硬盘。裸设备可以绑定一个分区，也可以绑定一个磁盘。 （比如/dev/rdsk）

udev 概述

- udev: 是为了防止在机器因为重启的时候，因为盘符发生改变，导致 asm disk 不能正常被 dg 应用，从而出现 asm 磁盘组不能 mount 的故障
- 11.2 版本之前，ASM 可以通过裸设备方式和 AMSlib 方式，允许在块设备上创建 ASM。
- 11.2 版本之后: ASM 本身直接支持块设备。
- 11.2 版本之前: 使用 udev 是将块设备绑定裸设备上，并控制访问权限，本质是在裸设备上创建 ASM；
- 11.2 版本及之后使用 udev 是固定盘符，并控制访问权限，本质是在块设备上创建 ASM。

asmlib

- asmlib: oracle 官方 10g 以上版本的 ASM 支持库。
- linux 上面给磁盘/分区头上面打上 asm 的标记，供 asm 使用，而且当磁盘的盘符发生改变的时候，不会影响到 asm disk。
- 从效果上说，和 udev 没有本质区别，
- 在 redhat4 和 5 中 oracle 提供 asmlib 程序，在 6 中，oracle 只为 oel 提供，其他 linux 不再提供。

RAW

- RAW：是以前 4 中常用的绑定裸设备的方法，到了 5.4 还是 5.5 之后，不建议使用该方法，而直接使用 udev 取代它

> - 建议：没有使用多路径管理软件时，使用 udev。在使用多路径软件时，直接使用多路径管理软件提供的路径。

## 系统配置

> - root用户执行

- 扩展磁盘

```sh
vgextend vg_orasoft /dev/sdb5 /dev/sdb6
lvextend -L 30G /dev/vg_orasoft/lv_orasoft_u01
xfs_growfs /dev/vg_orasoft/lv_orasoft_u01
```

- 新建/asmdisk 逻辑卷 20g

```sh
vgcreate vg_oraasm /dev/sdb7 /dev/sdb8 /dev/sdb9
vcreate -n lv_asmdisks -L 20G vg_oraasm
mkdir /asmdisk
mkfs.xfs /dev/vg_oraasm/lv_asmdisks
mount /dev/vg_oraasm/lv_asmdisks /asmdisk/
```

- 加入/etc/fstab

```sh
/dev/mapper/vg_oraasm-lv_asmdisks /asmdisk      xfs     defaults        0 0
```

- vim /etc/security/limits.conf

``` sh
grid soft nproc 2047
grid hard nproc 16384
grid soft nofile 1024
grid hard nofile 65536
```

- 配置用户

``` sh
groupadd oinstall
groupadd dba
groupadd oper
groupadd asmadmin
groupadd asmoper
groupadd asmdba
useradd -g oinstall -G dba,asmdba,oper,asmadmin oracle
useradd -g oinstall -G asmadmin,asmdba,asmoper,dba grid
usermod -a -G dba,asmdba,oper,asmadmin oracle
echo oracle | passwd --stdin oracle
echo grid | passwd --stdin grid
```

- 创建目录

``` sh
mkdir -p /u01/app/oracle
mkdir -p /u01/app/grid
mkdir -p /u01/app/11.2.0/grid
chown -R grid:oinstall /u01/app/11.2.0/grid
chown -R grid:oinstall /u01/app/grid
chown -R oracle:oinstall /u01/app/oracle
chgrp -R oinstall /u01
chmod -R 775 /u01
```

> - 切换到Grid用户

- vim /home/grid/.bash_profile

```sh
umask 022
export ORACLE_SID=+ASM
export ORACLE_BASE=/u01/app/grid
export ORACLE_HOME=/u01/app/11.2.0/grid
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export NLS_DATE_FORMAT="YYYY-MM-DD HH24:MI:SS"
export TMP=/tmp
export TMPDIR=$TMP
export PATH=$ORACLE_HOME/bin:$ORACLE_HOME/OPatch:$PATH
export EDITOR=vi
export TNS_ADMIN=$ORACLE_HOME/network/admin
export ORACLE_PATH=.:$ORACLE_BASE/dba_scripts/sql:$ORACLE_HOME/rdbms/admin
export SQLPATH=$ORACLE_HOME/sqlplus/admin
#export NLS_LANG="SIMPLIFIED CHINESE_CHINA.ZHS16GBK" --AL32UTF8 SELECT userenv('LANGUAGE')
#db_NLS_LANG FROM DUAL;
export NLS_LANG="AMERICAN_CHINA.ZHS16GBK"
alias sqlplus='rlwrap sqlplus'
alias rman='rlwrap rman'
alias asmcmd='rlwrap asmcmd'
alias sas='sqlplus / as sysdba'
alias sam='sqlplus / as sysasm'
#export PS1="[\u@\h-\`echo \$ORACLE_SID\` \W]$ "
#export PS1='[$LOGNAME@'`hostname`:'$PWD'']# '
```

## 创建 ASM


|  创建方式   | 命令                                    | 读取的配置文件                                 | 文件内容   |
| :---------:| -------------------------------------  | -------------------------------------------- | -------- |
| 11.2 版之前 |   /sbin/scsi_id                        | /etc/udev/rules.d/60-raw.rules               |          |
| 11.2 版之后 | /usr/lib/udev/scsi_id -g -u /dev/sdb8  | /etc/udev/rules.d/99-oracle-asmdevices.rules |          |


### uuid 方式

- 虚拟机或存储中添加硬盘后 fdisk 分区并刷新

```sh
[root@ocplhr ~]# fdisk /dev/sdf
```

查看分区结果

```sh
[root@ocplhr ~]# fdisk -l |grep /sd*
磁盘 /dev/sdb：107.4 GB, 107374182400 字节，209715200 个扇区
/dev/sdb1            2048    20973567    10485760   8e  Linux LVM
/dev/sdb2        20973568    41945087    10485760   8e  Linux LVM
/dev/sdb3        41945088    62916607    10485760   8e  Linux LVM
/dev/sdb4        62916608   209715199    73399296    5  Extended
/dev/sdb5        62918656    83890175    10485760   8e  Linux LVM
/dev/sdb6        83892224   104863743    10485760   8e  Linux LVM
/dev/sdb7       104865792   125837311    10485760   8e  Linux LVM
/dev/sdb8       125839360   146810879    10485760   8e  Linux LVM
/dev/sdb9       146812928   167784447    10485760   8e  Linux LVM
/dev/sdb10      167786496   188758015    10485760   8e  Linux LVM
/dev/sdb11      188760064   209715199    10477568   8e  Linux LVM
磁盘 /dev/sda：21.5 GB, 21474836480 字节，41943040 个扇区
/dev/sda1   *        2048      411647      204800   83  Linux
/dev/sda2          411648    38176767    18882560   8e  Linux LVM
/dev/sda3        38176768    41943039     1883136   8e  Linux LVM
磁盘 /dev/sdc：21.5 GB, 21474836480 字节，41943040 个扇区
/dev/sdc1            2048    41943039    20970496   83  Linux
磁盘 /dev/sdd：21.5 GB, 21474836480 字节，41943040 个扇区
/dev/sdd1            2048    41943039    20970496   83  Linux
磁盘 /dev/sde：21.5 GB, 21474836480 字节，41943040 个扇区
/dev/sde1            2048    41943039    20970496   83  Linux
磁盘 /dev/sdf：21.5 GB, 21474836480 字节，41943040 个扇区
/dev/sdf1            2048    41943039    20970496   83  Linux

```

用 partprobe 命令使系统重新识别所有分区

``` sh
[root@test ~]# partprobe
```

vim /etc/scsi_id.config #这个貌似可以不做

```sh
options=--whitelisted --replace-whitespace`
```

获取新增硬盘 uuid

```sh
[root@ocplhr ~]# /usr/lib/udev/scsi_id -g -u /dev/sdc
36000c29523915755fab569675081db9a
[root@ocplhr ~]# /usr/lib/udev/scsi_id -g -u /dev/sdd
36000c2985c6df2e6cf0046387ae1933a
[root@ocplhr ~]# /usr/lib/udev/scsi_id -g -u /dev/sde
36000c29f5c27a646f4a586ed656b7ece
[root@ocplhr ~]# /usr/lib/udev/scsi_id -g -u /dev/sdf
36000c290931f1fc0dcc1340fd6743fc1
```



vim /etc/udev/rules.d/99-oracle-asmdevices.rules
> 注意：这里的 GROUP=“asmadmin”, 最好修改成 GROUP=“asmdba”，不然最后可能用 dbca 创建数据库实例的时候找不见磁盘组
```sh
KERNEL=="sd*",ENV{DEVTYPE}=="disk",SUBSYSTEM=="block",PROGRAM=="/usr/lib/udev/scsi_id -g -u -d $devnode",RESULT=="36000c29523915755fab569675081db9a", RUN+="/bin/sh -c 'mknod /dev/asmdiskc b  $major $minor; chown grid:asmdba /dev/asmdiskc; chmod 0660 /dev/asmdiskc'"
KERNEL=="sd*",ENV{DEVTYPE}=="disk",SUBSYSTEM=="block",PROGRAM=="/usr/lib/udev/scsi_id -g -u -d $devnode",RESULT=="36000c2985c6df2e6cf0046387ae1933a", RUN+="/bin/sh -c 'mknod /dev/asmdiskd b  $major $minor; chown grid:asmdba /dev/asmdiskd; chmod 0660 /dev/asmdiskd'"
KERNEL=="sd*",ENV{DEVTYPE}=="disk",SUBSYSTEM=="block",PROGRAM=="/usr/lib/udev/scsi_id -g -u -d $devnode",RESULT=="36000c29f5c27a646f4a586ed656b7ece", RUN+="/bin/sh -c 'mknod /dev/asmdiske b  $major $minor; chown grid:asmdba /dev/asmdiske; chmod 0660 /dev/asmdiske'"
KERNEL=="sd*",ENV{DEVTYPE}=="disk",SUBSYSTEM=="block",PROGRAM=="/usr/lib/udev/scsi_id -g -u -d $devnode",RESULT=="36000c290931f1fc0dcc1340fd6743fc1", RUN+="/bin/sh -c 'mknod /dev/asmdiskf b  $major $minor; chown grid:asmdba /dev/asmdiskf; chmod 0660 /dev/asmdiskf'"

```

重启服务或系统
``` sh
/sbin/udevadm control --reload
```
重启后识别出新硬盘
``` sh
[root@ocplhr ~]# ll /dev/asm*
brw-rw---- 1 grid asmdba 8, 32 2月  17 15:56 /dev/asmdiskc
brw-rw---- 1 grid asmdba 8, 48 2月  17 15:56 /dev/asmdiskd
brw-rw---- 1 grid asmdba 8, 64 2月  17 15:56 /dev/asmdiske
brw-rw---- 1 grid asmdba 8, 80 2月  17 15:56 /dev/asmdiskf
```

## 静默安装 grid 软件

切换到Grid用户下执行

``` log
/soft/grid/runInstaller -silent -force -noconfig -IgnoreSysPreReqs -ignorePrereq -showProgress \
ORACLE_HOSTNAME=ocplhr \
INVENTORY_LOCATION=/u01/app/oraInventory \
SELECTED_LANGUAGES=en \
oracle.install.option=CRS_SWONLY \
ORACLE_BASE=/u01/app/grid \
ORACLE_HOME=/u01/app/11.2.0/grid \
oracle.install.asm.OSDBA=asmdba \
oracle.install.asm.OSOPER=asmoper \
oracle.install.asm.OSASM=asmadmin \
oracle.install.crs.config.storageOption=ASM_STORAGE \
oracle.install.crs.config.sharedFileSystemStorage.votingDiskRedundancy=NORMAL \
oracle.install.crs.config.sharedFileSystemStorage.ocrRedundancy=NORMAL \
oracle.install.crs.config.useIPMI=false \
oracle.install.asm.SYSASMPassword=lhr \
oracle.install.asm.diskGroup.name=OCR \
oracle.install.asm.diskGroup.redundancy=NORMAL \
oracle.install.asm.diskGroup.disks=/dev/asm-diskc,/dev/asm-diskd,/dev/asm-diske \
oracle.install.asm.monitorPassword=lhr \
oracle.installer.autoupdates.option=SKIP_UPDATES
```

切换到root下执行
``` sh
[root@ocplhr soft]# /u01/app/11.2.0/grid/root.sh
Check /u01/app/11.2.0/grid/install/root_ocplhr_2021-02-17_19-16-51.log for the output of root script
[root@ocplhr soft]# more /u01/app/11.2.0/grid/install/root_ocplhr_2021-02-17_19-16-51.log
Performing root user operation for Oracle 11g

The following environment variables are set as:
    ORACLE_OWNER= grid
    ORACLE_HOME=  /u01/app/11.2.0/grid
   Copying dbhome to /usr/local/bin ...
   Copying oraenv to /usr/local/bin ...
   Copying coraenv to /usr/local/bin ...

Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root script.
Now product-specific root actions will be performed.

To configure Grid Infrastructure for a Stand-Alone Server run the following command as the root user:
/u01/app/11.2.0/grid/perl/bin/perl -I/u01/app/11.2.0/grid/perl/lib -I/u01/app/11.2.0/grid/crs/install /u01/app/11.2.0/grid/crs/install/roothas.pl


To configure Grid Infrastructure for a Cluster execute the following command:
/u01/app/11.2.0/grid/crs/config/config.sh
This command launches the Grid Infrastructure Configuration Wizard. The wizard also supports silent operation, and the parameters can be pas
sed through the response file that is available in the installation media.
```

> - 由于是在centos7使用 systemd 而不是 initd 运行进程和重启进程，而 root.sh 是通过传统的 initd 运行ohasd进程。必须手动添加ohas.service为系统服务并启动。

在root下执行
``` sh
touch /usr/lib/systemd/system/ohas.service
chmod 777 /usr/lib/systemd/system/ohas.service
```
vim /usr/lib/systemd/system/ohas.service
``` sh
[Unit]
Description=Oracle High Availability Services
After=syslog.target

[Service]
ExecStart=/etc/init.d/init.ohasd run >/dev/null 2>&1 Type=simple
Restart=always

[Install]
WantedBy=multi-user.target
```

以root用户执行
``` sh
systemctl daemon-reload
systemctl enable ohas.service
systemctl start ohas.service
```
查看状态
``` sh
systemctl status ohas.service
```

因为刚才已经安装过一次，现在重复安装需要强制初始化（加参数：-deconfig -force -verbose）
``` sh
[root@ocplhr grid]# /u01/app/11.2.0/grid/perl/bin/perl -I/u01/app/11.2.0/grid/perl/lib -I/u01/app/11.2.0/grid/crs/install /u01/app/11.2.0/grid/crs/install/roothas.pl  -deconfig -force -verbose
Using configuration parameter file: /u01/app/11.2.0/grid/crs/install/crsconfig_params
CRS-2613: Could not find resource 'ora.cssd'.
CRS-4000: Command Stop failed, or completed with errors.
CRS-2613: Could not find resource 'ora.cssd'.
CRS-4000: Command Delete failed, or completed with errors.
CRS-4133: Oracle High Availability Services has been stopped.
Successfully deconfigured Oracle Restart stack
```
以ROOT用户，重新执行pl脚本
``` sh
[root@ocplhr grid]# /u01/app/11.2.0/grid/perl/bin/perl -I/u01/app/11.2.0/grid/perl/lib -I/u01/app/11.2.0/grid/crs/install /u01/app/11.2.0/grid/crs/install/roothas.pl
Using configuration parameter file: /u01/app/11.2.0/grid/crs/install/crsconfig_params
User ignored Prerequisites during installation
LOCAL ADD MODE
Creating OCR keys for user 'grid', privgrp 'oinstall'..
Operation successful.
LOCAL ONLY MODE
Successfully accumulated necessary OCR keys.
Creating OCR keys for user 'root', privgrp 'root'..
Operation successful.
CRS-4664: Node ocplhr successfully pinned.
Adding Clusterware entries to inittab

ocplhr     2021/02/17 19:47:21     /u01/app/11.2.0/grid/cdata/ocplhr/backup_20210217_194721.olr
Successfully configured Oracle Grid Infrastructure for a Standalone Server
```

检查状态
``` sh
[grid@ocplhr ~]$ crsctl stat res -t
--------------------------------------------------------------------------------
NAME           TARGET  STATE        SERVER                   STATE_DETAILS
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.ons
               OFFLINE OFFLINE      ocplhr
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.cssd
      1        OFFLINE OFFLINE
ora.diskmon
      1        OFFLINE OFFLINE
ora.evmd
      1        ONLINE  ONLINE       ocplhr
```

## 静默安装 grid 实例

/u01/app/11.2.0/grid/bin/asmca -silent -configureASM -sysAsmPassword lhr -asmsnmpPassword lhr -diskGroupName OCR -diskList /dev/asmdiskc,/dev/asmdiskd,/dev/asmdiske -redundancy NORMAL

alter system set asm_diskstring='/dev/asmdiskc'  scope=spfile;

## 创建磁盘组

## 创建 ASM 管理的数据库

## 创建 EM

## 其它
