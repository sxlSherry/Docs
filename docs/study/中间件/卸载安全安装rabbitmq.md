# 卸载安全安装rabbitmq





一、卸载

[root@zabbix_server lib]# rpm -qa|grep rabbitmq
rabbitmq-server-3.6.5-1.noarch



yum list installed rabbitmq

yum remove



[root@zabbix_server lib]# rpm -e --nodeps rabbitmq-server

二、此时注意一定要手工删除rabbitmq目录

rm -rf /etc/rabbitmq
rm -rf /usr/lib/rabbitmq

rm -rf /var/lib/rabbitmq



安装新版RabbitMQ3.9.10，不需要先安装Erlang了，安装mq时，会自动安装erlang的依赖项

Erlang和RabbitMQ版本对照：RabbitMQ Erlang Version Requirements — RabbitMQ

一、下载所需要的包

erlang下载地址：rabbitmq/erlang - Packages · packagecloud

rabbitmq-server下载地址：Releases · rabbitmq/rabbitmq-server · GitHub



二、安装Erlang

rpm -Uvh erlang-24.1.7-1.el8.x86_64.rpm
yum install -y erlang
erl -v
1
2
3


三、安装RabbitMQ

在RabiitMQ安装过程中需要依赖socat插件，首先安装该插件

yum install -y socat
1
然后解压安装RabbitMQ的安装包

# 解压 rpm -Uvh rabbitmq-server-3.9.11-1.el8.noarch.rpm
# 安装 yum install -y rabbitmq-server
1
2


四、启动RabbitMQ服务

# 启动rabbitmq
systemctl start rabbitmq-server


# 查看rabbitmq状态
systemctl status rabbitmq-server
1
2
3
4
5
6


其他命令：

# 设置rabbitmq服务开机自启动
systemctl enable rabbitmq-server


# 关闭rabbitmq服务
systemctl stop rabbitmq-server


# 重启rabbitmq服务
systemctl restart rabbitmq-server
1
2
3
4
5
6
7
8
9
10
五、RabbitMQ Web管理界面及授权操作

1、安装启动RabbitMQWeb管理界面注：默认情况下，rabbitmq没有安装web端的客户端软件，需要安装才可以生效

# 打开RabbitMQWeb管理界面插件
rabbitmq-plugins enable rabbitmq_management
1
2


访问ip:15672 例如：http://192.168.206.171:15672/

rabbitmq有一个默认的账号密码guest，但该情况仅限于本机localhost进行访问，所以需要添加一个远程登录的用户



2、用户操作

# 添加用户
rabbitmqctl add_user 用户名 密码


# 设置用户角色,分配操作权限
rabbitmqctl set_user_tags 用户名 角色


# 为用户添加资源权限(授予访问虚拟机根节点的所有权限)
rabbitmqctl set_permissions -p / 用户名 ".*" ".*" ".*"


# 修改密码
rabbitmqctl change_ password 用户名 新密码


# 删除用户
rabbitmqctl delete_user 用户名


# 查看用户清单
rabbitmqctl list_users
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
角色有四种：

administrator：可以登录控制台、查看所有信息、并对rabbitmq进行管理

monToring：监控者；登录控制台，查看所有信息

policymaker：策略制定者；登录控制台指定策略

managment：普通管理员；登录控制

创建用户twm，密码123456，设置adminstator角色，赋予所有权限

rabbitmqctl add_user twm 123456

rabbitmqctl set_user_tags twm administrator


