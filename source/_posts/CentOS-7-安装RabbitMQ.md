title: CentOS-7 安装RabbitMQ
author: 你丶一顾倾城
tags:
  - RabbitMQ
  - 笔记
  - 安装教程
categories: []
date: 2018-11-11 09:37:00
---
<!-- toc -->

<!-- more -->
> 当前用户 root
> 当前目录 /root/app/rabbitmq

## 相关依赖
### 安装erlang-solution
**下载安装包**

```shell
wget https://packages.erlang-solutions.com/erlang-solutions-1.0.1.noarch.rpm
```
**安装**   
在安装包所在目录下执行

```shell
rpm -Uvh erlang-solutions-1.0.1.noarch.rpm
```
或者直接yum仓库安装
```shell
yum install erlang-solutions
```
### 安装epel-release
```shell
yum install epel-release
```
### 安装erlang
```shell
yum install erlang
```
## 安装RabbitMQ
### 安装
**下载安装包**
```shell
wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.7.8/rabbitmq-server-3.7.8-1.el7.noarch.rpm
```
**执行安装**

```shell
yum install -y rabbitmq-server-3.7.8-1.el7.noarch.rpm
```

### 开启远程访问限制
```shell
vi /etc/rabbitmq/rabbit.config
```
添加如下内容
```shell
[{rabbit,[{loopback_users,[]}]}].
```
**注意，最后面的 . 不要漏掉**


### 开启web访问（需要先开远程访问限制）
```shell
[root@arcgisserver app]# rabbitmq-plugins enable rabbitmq_management
```
这里报了个错：

```shell
19:54:27.271 [error] Cookie file /var/lib/rabbitmq/.erlang.cookie must be accessible by owner only

19:54:28.070 [error] Cookie file /var/lib/rabbitmq/.erlang.cookie must be accessible by owner only
...
Distribution failed: {{:shutdown, {:failed_to_start_child, :auth, {'Cookie file /var/lib/rabbitmq/.erlang.cookie must be accessible by owner only', [{:auth, :init_cookie, 0, [file: 'auth.erl', line: 286]}, {:auth, :init, 1, [file: 'auth.erl', line: 140]}, {:gen_server, :init_it, 2, [file: 'gen_server.erl', line: 374]}, {:gen_server, :init_it, 6, [file: 'gen_server.erl', line: 342]}, {:proc_lib, :init_p_do_apply, 3, [file: 'proc_lib.erl', line: 249]}]}}}, {:child, :undefined, :net_sup_dynamic, {:erl_distribution, :start_link, [[:rabbitmqcli61, :shortnames], false]}, :permanent, 1000, :supervisor, [:erl_distribution]}}

```
看起来貌似是当前用户没权限，那就赋各权限吧。。。
```shell
[root@arcgisserver app]# chmod 600 /var/lib/rabbitmq/.erlang.cookie
```
重新执行：
```shell
[root@arcgisserver app]# rabbitmq-plugins enable rabbitmq_management
The following plugins have been configured:
  rabbitmq_management
  rabbitmq_management_agent
  rabbitmq_web_dispatch
Applying plugin configuration to rabbit@arcgisserver...
The following plugins have been enabled:
  rabbitmq_management
  rabbitmq_management_agent
  rabbitmq_web_dispatch

set 3 plugins.
Offline change; changes will take effect at broker restart.
```
搞定

### 安装消息延迟插件
```shell
[root@arcgisserver plugins]# cd /usr/lib/rabbitmq/lib/rabbitmq_server-3.7.8/plugins

[root@arcgisserver plugins]# wget https://dl.bintray.com/rabbitmq/community-plugins/rabbitmq_delayed_message_exchange-0.0.1.ez
--2018-11-11 21:29:43--  https://dl.bintray.com/rabbitmq/community-plugins/rabbitmq_delayed_message_exchange-0.0.1.ez
Resolving dl.bintray.com (dl.bintray.com)... 75.126.118.188
Connecting to dl.bintray.com (dl.bintray.com)|75.126.118.188|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 32019 (31K) [application/octet-stream]
Saving to: ‘rabbitmq_delayed_message_exchange-0.0.1.ez’

100%[=================================================================================================>] 32,019      22.7KB/s   in 1.4s   
2018-11-11 21:29:46 (22.7 KB/s) - ‘rabbitmq_delayed_message_exchange-0.0.1.ez’ saved [32019/32019]

[root@arcgisserver plugins]# rabbitmq-plugins enable rabbitmq_delayed_message_exchange
The following plugins have been configured:
  rabbitmq_delayed_message_exchange
  rabbitmq_management
  rabbitmq_management_agent
  rabbitmq_web_dispatch
Applying plugin configuration to rabbit@arcgisserver...
The following plugins have been enabled:
  rabbitmq_delayed_message_exchange

set 4 plugins.
Offline change; changes will take effect at broker restart.
```

### 放行端口
```shell
firewall-cmd --add-port=15672/tcp --perament
firewall-cmd --add-port=5672/tcp --perament
systemctl restart firewalld.service
```

### 启动/停止/重启服务
```shell
start rabbitmq-server.service
systemctl stop rabbitmq-server.service
systemctl restart rabbitmq-server.service
#启动rabbitmq，-detached代表后台守护进程方式启动。
#rabbitmq-server -detached 
```
### 查看状态
```shell
rabbitmqctl status
```


### 设为开机自动启动
```shell
chkconfig rabbitmq-server on
```