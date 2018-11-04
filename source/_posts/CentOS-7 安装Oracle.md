title: CentOS-7 安装Oracle
author: John Doe
tags: []
categories: []
date: 2018-11-04 08:33:00
---
> 由于客户生产环境Oracle需要做交叉备份及归档备份，且自己Oracle水平又不高，不敢直接在生产环境动手。但公司的集成环境也有很多项目要跑，所以干脆自己虚拟机装一个测。因为虚拟机已有装好的ubuntu，前两天就直接在ubuntu上装了。但是中间遇到了不小的阻碍，主要是某些依赖拉不到，而ubuntu的依赖安装又和yum冲突，所以昨天决定直接不用ubuntu，改用CentOS-7，总算是顺利完成了。详细过程记录如下。

---
[TOC]

#### 0. 安装JDK1.8

###### 0.1 删除预装的jdk

如果系统中事先存在自带的jdk，则需要删除  
卸载centos原本自带的openjdk，运行命令：rpm -qa | grep java  
然后通过    rpm -e --nodeps   后面跟系统自带的jdk名    这个命令来删除系统自带的jdk
例如：  

```
rpm -e --nodeps java-1.8.0-openjdk-1.8.0.102-4.b14.el7.x86_64
rpm -e --nodeps java-1.8.0-openjdk-headless-1.8.0.102-4.b14.el7.x86_64
rpm -e --nodeps java-1.7.0-openjdk-headless-1.7.0.111-2.6.7.8.el7.x86_64
rpm -e --nodeps java-1.7.0-openjdk-1.7.0.111-2.6.7.8.el7.x86_64
```

###### 0.2 下载jdk
```
wget http://download.oracle.com/otn-pub/java/jdk/8u161-b12/2f38c3b165be4555a1fa6e98c45e0808/jdk-8u161-linux-x64.tar.gz
```
可以wget下，我这里本地有jdk-8u161-linux-x64.tar.gz就直接用了  
执行步骤如下
上传安装包至/root/java目录下，执行
```
tar -zxvf jdk-8u161-linux-x64.tar.gz
```
###### 0.3 配置环境变量
全局环境变量是通过/etc/profile配置的

```
[root@192 jdk1.8.0_161]# vi /etc/profile
```
在文件最下面添加

```
export JAVA_HOME=/root/java/jdk1.8.0_161
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
```
wq保存后执行

```
. /etc/profile
```
使环境变量生效，注意 . 与 / 之间的空格  

###### 0.4 检查jdk是否生效
```
[root@192 jdk1.8.0_161]# java -version
java version "1.8.0_161"
Java(TM) SE Runtime Environment (build 1.8.0_161-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.161-b12, mixed mode)
```



#### 1. 通过yum快速安装Oracle依赖      

之前用ubuntu，各种依赖下不到（撞墙）。换了CentOS，yum还是挺好用的，依赖也比较好找。关键是不需要一个个去找依赖（ubuntu的apt-get很多东西下不到，比如很重要的一个glibc）
最简便的方法：  
执行  
```
wget http://public-yum.oracle.com/public-yum-ol7.repo -O /etc/yum.repos.d/public-yum-ol7.repo
wget http://public-yum.oracle.com/RPM-GPG-KEY-oracle-ol7 -O /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
```
执行结果如下
````
[root@bogon yum.repos.d]# wget http://public-yum.oracle.com/public-yum-ol7.repo -O /etc/yum.repos.d/public-yum-ol7.repo
--2018-10-25 20:05:26--  http://public-yum.oracle.com/public-yum-ol7.repo
Resolving public-yum.oracle.com (public-yum.oracle.com)... 69.192.9.199
Connecting to public-yum.oracle.com (public-yum.oracle.com)|69.192.9.199|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 14368 (14K) [text/plain]
Saving to: ‘/etc/yum.repos.d/public-yum-ol7.repo’

100%[====================================================================================================================================================================================================================================>] 14,368      --.-K/s   in 0.01s   

2018-10-25 20:05:26 (1.33 MB/s) - ‘/etc/yum.repos.d/public-yum-ol7.repo’ saved [14368/14368]

[root@bogon yum.repos.d]# wget http://public-yum.oracle.com/RPM-GPG-KEY-oracle-ol7 -O /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
--2018-10-25 20:05:40--  http://public-yum.oracle.com/RPM-GPG-KEY-oracle-ol7
Resolving public-yum.oracle.com (public-yum.oracle.com)... 69.192.9.199
Connecting to public-yum.oracle.com (public-yum.oracle.com)|69.192.9.199|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1011 [text/plain]
Saving to: ‘/etc/pki/rpm-gpg/RPM-GPG-KEY-oracle’

100%[====================================================================================================================================================================================================================================>] 1,011       --.-K/s   in 0s      

2018-10-25 20:05:41 (296 MB/s) - ‘/etc/pki/rpm-gpg/RPM-GPG-KEY-oracle’ saved [1011/1011]
````
**注意：这里有个坑，CentOS-7的linux版本需要下public-yum-ol7.repo，网上很多教程都是让下public-yum-ol6.repo，其实是CentOS-6用的，yum install的时候会有依赖冲突，在这里卡了很久。。。**  
完成后备份一下这个目录的文件到其他目录，这个文件夹是修改系统后日志和原本的内核配置备份
```
/var/log/oracle-rdbms-server-11gR2-preinstall
```


#### 2. 系统中的参数设置
###### 2.1  修改/etc/sysctl.conf
在/etc/sysctl.conf文件中，添加以下内核参数  
```
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 536870912
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048586
```
输入wq保存即可

###### 2.2  加载内核参数
执行
```
[root@bogon yum.repos.d]# sysctl -f
fs.file-max = 6815744
kernel.sem = 250 32000 100 128
kernel.shmmni = 4096
kernel.shmall = 1073741824
kernel.shmmax = 4398046511104
kernel.panic_on_oops = 1
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048576
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
fs.aio-max-nr = 1048576
net.ipv4.ip_local_port_range = 9000 65500
fs.aio-max-nr = 1048576
fs.file-max = 6815744
kernel.shmall = 2097152
kernel.shmmax = 536870912
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
net.ipv4.ip_local_port_range = 9000 65500
net.core.rmem_default = 262144
net.core.rmem_max = 4194304
net.core.wmem_default = 262144
net.core.wmem_max = 1048586

```
该命令sysctl.conf -p的效果一样

###### 2.3 关闭防火墙
```
systemctl disable firewalld.service
```
执行结果

```
[root@192 ~]# systemctl disable firewalld.service
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.

```

###### 2.4 关闭SELINUX（需重启生效）

```
[root@192 ~]# vi /etc/selinux/config
[root@192 ~]# cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
#SELINUX=enforcing
SELINUX=disabled #此处修改为disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted 

```

#### 3. 配置oracle系统配置文件&授权
执行
```
cat >> /etc/oraInst.loc <<EOF
inventory_loc=/home/oracle/ora11g/oraInventory
inst_group=oinstall
EOF
```
执行结果如下
```
[root@bogon ~]# pwd
/root
[root@bogon ~]# cat >> /etc/oraInst.loc <<EOF
> inventory_loc=/home/oracle/ora11g/oraInventory
> inst_group=oinstall
> EOF
[root@bogon ~]# ll
total 4
-rw-------. 1 root root 1528 Oct 25 16:03 anaconda-ks.cfg
[root@bogon ~]# chmod 664 /etc/oraInst.loc
```



#### 4. 创建oracle安装的目录&授权
依次执行
```
[root@bogon ~]# mkdir -p /u01/app/
[root@bogon ~]# mkdir /u01/tmp
[root@bogon ~]# chown -R oracle:oinstall /u01/app/
[root@bogon ~]# chmod -R 775 /u01/app/
[root@bogon ~]# chmod a+wr /u01/tmp
[root@bogon ~]# passwd oracle
Changing password for user oracle.
New password: 
Retype new password: 
passwd: all authentication tokens updated successfully.
```
给oinstall组添加oracle用户

```
[root@bogon ~]# usermod -a -G oinstall oracle
```

---
**新建oracle用户连接，切换至oracle用户**

#### 5. 配置用户环境&上传安装包

###### 5.1 为oracle用户添加一些必要的环境
```
cat >> /home/oracle/.bash_profile <<EOF
TMP=/u01/tmp
TMPDIR=/u01/tmp
export TMP TMPDIR
ORACLE_BASE=/u01/app/oracle
ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1
ORACLE_SID=orcl
PATH=$ORACLE_HOME/bin:$PATH
export ORACLE_BASE ORACLE_SID ORACLE_HOME PATH
EOF
```
最后执行以下命令使环境变量生效
```
source .bash_profile
```

执行结果如下
```
[oracle@bogon ~]$ cat >> /home/oracle/.bash_profile <<EOF
> TMP=/u01/tmp
> TMPDIR=/u01/tmp
> export TMP TMPDIR
> ORACLE_BASE=/u01/app/oracle
> ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1
> ORACLE_SID=orcl
> PATH=$ORACLE_HOME/bin:$PATH
> export ORACLE_BASE ORACLE_SID ORACLE_HOME PATH
> EOF
[oracle@bogon ~]$ source .bash_profile
[oracle@bogon ~]$ $ORACLE_HOME
-bash: /u01/app/oracle/product/11.2.0/dbhome_1: No such file or directory
```
###### 5.2 上传
将  
linux.x64_11gR2_database_1of2.zip  
linux.x64_11gR2_database_2of2.zip  
上传至/home/oracle

###### 5.3 解压
这个没啥好说的，直接解压
```
unzip linux.x64_11gR2_database_1of2.zip 
unzip linux.x64_11gR2_database_2of2.zip 
```

###### 5.4 检查文件夹权限（root操作）
database文件夹需要切换成oracle用户权限
```
chown -R oracle:oinstall /home/oracle/database
```


#### 6 安装数据库
###### 6.1 配置db_install.rsp
该文件用于静默安装使用
备份/home/oracle/database/response到/home/oracle/rsp/
```
cp -r /home/oracle/database/response /home/oracle/rsp
```
进入rsp文件夹，新建安装响应文件db_install.rsp
```
[oracle@bogon ~]$ cd rsp
[oracle@bogon rsp]$ touch db_install.rsp
```
编辑该文件

```
##我的/home/oracle/rsp/db_install.rsp
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v11_2_0
#INSTALL_DB_AND_CONFIG安装并自动配置数据库实例和监听 建议首次安装用这个
#不然配置另外两个文件，新建实例和监听
oracle.install.option=INSTALL_DB_AND_CONFIG
ORACLE_HOSTNAME=localhost
UNIX_GROUP_NAME=oinstall
INVENTORY_LOCATION=/home/oracle/ora11g/oraInventory
SELECTED_LANGUAGES=zh_CN,en
ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1
ORACLE_BASE=/u01/app/oracle
oracle.install.db.InstallEdition=EE
oracle.install.db.isCustomInstall=true
oracle.install.db.customComponents=oracle.server:11.2.0.1.0,oracle.sysman.ccr:10.2.7.0.0,oracle.xdk:11.2.0.1.0,oracle.rdbms.oci:11.2.0.1.0,oracle.network:11.2.0.1.0,oracle.network.listener:11.2.0.1.0,oracle.rdbms:11.2.0.1.0,oracle.options:11.2.0.1.0,oracle.rdbms.partitioning:11.2.0.1.0,oracle.oraolap:11.2.0.1.0,oracle.rdbms.dm:11.2.0.1.0,oracle.rdbms.dv:11.2.0.1.0,orcle.rdbms.lbac:11.2.0.1.0,oracle.rdbms.rat:11.2.0.1.0
oracle.install.db.DBA_GROUP=dba
oracle.install.db.OPER_GROUP=oinstall
oracle.install.db.config.starterdb.type=GENERAL_PURPOSE
#这个是服务名
oracle.install.db.config.starterdb.globalDBName=orcl.lts
#实例sid
oracle.install.db.config.starterdb.SID=orcl
oracle.install.db.config.starterdb.characterSet=AL32UTF8
oracle.install.db.config.starterdb.memoryOption=true
#最小256M
oracle.install.db.config.starterdb.memoryLimit=1024
#是否安装scott和hr
oracle.install.db.config.starterdb.installExampleSchemas=true
oracle.install.db.config.starterdb.enableSecuritySettings=true
#密码全设置成Helloworld_123，初始的sys的dba账号会是这个密码 (需要大小写字母加某些特殊符号，比如下划线，井号，很多其他特殊字符不支持)
oracle.install.db.config.starterdb.password.ALL=Helloworld_123
oracle.install.db.config.starterdb.control=DB_CONTROL
oracle.install.db.config.starterdb.dbcontrol.enableEmailNotification=false
oracle.install.db.config.starterdb.automatedBackup.enable=false
oracle.install.db.config.starterdb.storageType=FILE_SYSTEM_STORAGE
oracle.install.db.config.starterdb.fileSystemStorage.dataLocation=/u01/app/oracle/oradata
#true
DECLINE_SECURITY_UPDATES=true
```

###### 6.2 执行静默安装
```
/home/oracle/database/runInstaller -silent -ignorePrereq  -responseFile /home/oracle/rsp/db_install.rsp
```
可以用

```
tail -f /home/oracle/ora11g/oraInventory/logs/installActions2018-10-25_11-30-37PM.log 400
```
追踪日志

安装成功后会有这样一段提示
```
INFO: Read: Look at the log file "/u01/app/oracle/cfgtoollogs/dbca/orcl/orcl.log" for further details.
The following configuration scripts need to be executed as the "root" user. 
 #!/bin/sh 
 #Root scripts to run

/u01/app/oracle/product/11.2.0/dbhome_1/root.sh
To execute the configuration scripts:
	 1. Open a terminal window 
	 2. Log in as "root" 
	 3. Run the scripts 
	 4. Return to this window and hit "Enter" key to continue 

Successfully Setup Software.

```
提示说需要用root去执行一下/u01/app/oracle/product/11.2.0/dbhome_1/root.sh  

---

切换到root账号


```
[root@192 ~]# cd /u01/app/oracle/product/11.2.0/dbhome_1/
[root@192 dbhome_1]# ./root.sh 
Check /u01/app/oracle/product/11.2.0/dbhome_1/install/root_192.168.192.142_2018-10-25_23-46-33.log for the output of root script
[root@192 dbhome_1]# cat /u01/app/oracle/product/11.2.0/dbhome_1/install/root_192.168.192.142_2018-10-25_23-46-33.log

Running Oracle 11g root.sh script...

The following environment variables are set as:
    ORACLE_OWNER= oracle
    ORACLE_HOME=  /u01/app/oracle/product/11.2.0/dbhome_1

Creating /etc/oratab file...
Entries will be added to the /etc/oratab file as needed by
Database Configuration Assistant when a database is created
Finished running generic part of root.sh script.
Now product-specific root actions will be performed.
Finished product-specific root actions.
```

搞定了！！！

---

切换到oracle用户

#### 7 测试连接

```
[oracle@192 ~]$ sqlplus
-bash: sqlplus: command not found
```
WTF？？？  
先添加环境变量

```
[oracle@192 ~]$ vim /home/oracle/.bash_profile
[oracle@192 ~]$ source .bash_profile
[oracle@192 ~]$ cat /home/oracle/.bash_profile
# .bash_profile

# Get the aliases and functions
if [ -f ~/.bashrc ]; then
	. ~/.bashrc
fi

# User specific environment and startup programs

PATH=$PATH:$HOME/.local/bin:$HOME/bin

export PATH
TMP=/u01/tmp
TMPDIR=/u01/tmp
export TMP TMPDIR
ORACLE_BASE=/u01/app/oracle
ORACLE_HOME=/u01/app/oracle/product/11.2.0/dbhome_1
ORACLE_SID=orcl
PATH=/bin:/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/home/oracle/.local/bin:/home/oracle/bin:$PATH:$HOME/bin:$ORACLE_HOME/bin
export ORACLE_BASE ORACLE_SID ORACLE_HOME PATH

```
PATH后面多添了一些  
如果还是不行，试试下面的

```
[oracle@dg1 ~]$ sqlplus /nolog
bash: sqlplus: command not found
[oracle@dg1 ~]$ ln -s $ORACLE_HOME/bin/sqlplus /usr/bin
ln: creating symbolic link `/usr/bin/sqlplus' to `/bin/sqlplus': Permission deni ed
[oracle@dg1 ~]$ su - root
Password:
[root@dg1 ~]# ln -s $ORACLE_HOME/bin/sqlplus /usr/bin
[root@dg1 ~]# su - oracle
```
再不行，没有什么是重启服务器解决不了的。。。

至此就可以用sqlplus连接数据库了

但是window上配置net manager测试却无法访问，错误为ORA-12514 : TNS:监听程序当前无法识别连接描述符中请求的服务  
处理方法如下
修改listener.ora  
```
# listener.ora Network Configuration File: /u01/app/oracle/product/11.2.0/dbhome_1/network/admin/listener.ora
# Generated by Oracle configuration tools.
SID_LIST_LISTENER =
    (SID_DESC =
      (GLOBAL_DBNAME = orcl.lts)
      (ORACLE_HOME = /u01/app/oracle/product/11.2.0/dbhome_1)
      (SID_NAME = orcl)
    )
  )
LISTENER =
  (DESCRIPTION_LIST =
    (DESCRIPTION =
      (ADDRESS = (PROTOCOL = IPC)(KEY = EXTPROC1521))
      (ORACLE_HOME = /u01/app/oracle/product/11.2.0/dbhome_1)
      (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.192.142)(PORT = 1521))
    )
  )

ADR_BASE_LISTENER = /u01/app/oracle
```
重启监听

```
lsnrctl stop
lsnrctl start
```

net manager中的服务名为orcl.lts，与tnsnames.ora中的SERVICE_NAME对应


好了  大功告成！