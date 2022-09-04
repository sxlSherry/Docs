# 卸载安装rabbitmq

​		服务器上的rabbitmq坏了，需要卸载重装。其实安装很简单，但是因为卸载问题，一直出现问题，记录一下。

## 一、卸载

1. 查看之前安装过的rabbitmq。

   rpm安装的查看命令：```rpm -qa|grep rabbitmq```

   删除命令：``` rpm -e --nodeps rabbitmq-server```

   yum安装的查看命令：```yum list installed rabbitmq```

​		删除命令：```yum remove rabbitmq```

2. 此时注意一定要手工删除rabbitmq目录

   **【注意】**：这一步很重要，我之前没删，安装了好多次，一直报错，卸载重装一定要删除安装目录呀！！！

   可以搜索一下rabbitmq，命令：```find / -name rabbitmq```

   一般在以下几个目录下： /etc/rabbitmq、/usr/lib/rabbitmq、/var/lib/rabbitmq

   用rm -rf命令删除。



## 二、安装

安装新版RabbitMQ3.9.10，不需要先安装Erlang了，安装mq时，会自动安装erlang的依赖项。

Erlang和RabbitMQ版本对照：RabbitMQ Erlang Version Requirements — RabbitMQ

1. 下载所需要的包

erlang下载地址：rabbitmq/erlang - Packages · packagecloud

rabbitmq-server下载地址：Releases · rabbitmq/rabbitmq-server · GitHub

2. 安装Erlang

```
# rpm安装
rpm -ivh erlang-24.1.7-1.el8.x86_64.rpm
# yum安装
yum install -y erlang
# 查看版本
erl -v
```

3. 安装RabbitMQ

​	在RabiitMQ安装过程中需要依赖socat插件，首先安装该插件。

​	安装命令```yum install -y socat```

​	然后解压安装RabbitMQ的安装包

```
# rpm安装
rpm -ivh rabbitmq-server-3.9.11-1.el8.noarch.rpm
# yum安装
yum install -y rabbitmq-server
```



## 三、启动RabbitMQ服务

### 启动rabbitmq
systemctl start rabbitmq-server


### 查看rabbitmq状态
systemctl status rabbitmq-server


### 设置rabbitmq服务开机自启动
systemctl enable rabbitmq-server


### 关闭rabbitmq服务
systemctl stop rabbitmq-server


### 重启rabbitmq服务
systemctl restart rabbitmq-server



## 四、RabbitMQ Web管理界面及授权操作

1. 安装启动RabbitMQWeb管理界面

   注：默认情况下，rabbitmq没有安装web端的客户端软件，需要安装才可以生效

打开RabbitMQWeb管理界面插件命令：```rabbitmq-plugins enable rabbitmq_management```

2. 访问

   默认端口:15672 例如：http://192.168.13.171:15672/

   注：rabbitmq有一个默认的账号密码guest，但该情况仅限于本机localhost进行访问，所以需要添加一个远程登录的用户



## 五、用户操作

### 添加用户
```rabbitmqctl add_user 用户名 密码```


### 设置用户角色,分配操作权限
```rabbitmqctl set_user_tags 用户名 角色```


### 为用户添加资源权限(授予访问虚拟机根节点的所有权限)
```rabbitmqctl set_permissions -p / 用户名 ".*" ".*" ".*"```


### 修改密码
```rabbitmqctl change_ password 用户名 新密码```


### 删除用户
```rabbitmqctl delete_user 用户名``


### 查看用户清单
```rabbitmqctl list_users```

### 角色有四种：

administrator：可以登录控制台、查看所有信息、并对rabbitmq进行管理

monToring：监控者；登录控制台，查看所有信息

policymaker：策略制定者；登录控制台指定策略

managment：普通管理员；登录控制

### 创建用户test，密码123456，设置adminstator角色，赋予所有权限

```
rabbitmqctl add_user test 123456

rabbitmqctl set_user_tags test administrator
```





