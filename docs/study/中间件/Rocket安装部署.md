# Rocket安装部署

## windows部署

### 下载安装

下载地址：https://rocketmq.apache.org

Windows部署很简单，下载解压就好了。

### 配置

#### 1. 系统环境变量配置（必须配，不然启动报错）

​    变量名：ROCKETMQ_HOME

​    变量值：MQ解压路径\MQ文件夹名

#### 2. 配置文件修改

​	需要外网访问需要在**MQ文件夹conf/broker.conf** 文件中加一行```brokerIP1=xx.xx.xx.xx```，“xx.xx.xx.xx”是服务器的外网IP。

### 启动

#### 1. 启动NAMESERVER
cmd命令框执行进入至‘MQ文件夹\bin’下，然后执行‘**start mqnamesrv.cmd**’，启动NAMESERVER。成功后会弹出提示框，**此框勿关闭**。

#### 2.启动BROKER

cmd命令框执行进入至‘MQ文件夹\bin’下，然后执行‘**start mqbroker.cmd -n 127.0.0.1:9876 autoCreateTopicEnable=true**’，启动BROKER。成功后会弹出提示框，**此框勿关闭**。

### 遇到的问题

1. 需要外网访问必须开的端口：9876、10909、10911。

2. 配置的系统环境（JAVA_HOME和ROCKETMQ_HOME）路径有空格，启动会找不到类。

   解决办法：

   1. 换成没有空格的路径；
   2. 打开runbroker.cmd，然后将‘%CLASSPATH%’加上英文双引号。（这个方法亲测无效，但很多小伙伴都说有用）；
   3. https://blog.csdn.net/s724542558/article/details/119996755



## docker部署

### 下载安装

1. 通过docker的hub.docker.com上进行搜索；
2. Linux下通过docker的search命令进行搜索```docker search rocketmq```;
   本次使用的是foxiswho/rocketmq，以下是一个查看当前镜像所有的版本shell命令：

```
curl https://registry.hub.docker.com/v1/repositories/foxiswho/rocketmq/tags\
| tr -d '[\[\]" ]' | tr '}' '\n'\
| awk -F: -v image='foxiswho/rocketmq' '{if(NR!=NF && $3 != ""){printf("%s:%s\n",image,$3)}}'
```


注：如果要查看其它的镜像，只需要将地址中的foxiswho/rocketmq和其中的镜像名称foxiswho/rocketmq替换为其它镜像即可。

###启动

#### 1. server

##### server 无日志目录映射

```bash
docker run -d \
      --name rmqnamesrv \
      -e "JAVA_OPT_EXT=-Xms512M -Xmx512M -Xmn128m" \
      -p 9876:9876 \
      foxiswho/rocketmq:4.8.0 \
      sh mqnamesrv
```

##### server 有日志目录映射

```bash
docker run -d -v $(pwd)/logs:/home/rocketmq/logs \
      --name rmqnamesrv \
      -e "JAVA_OPT_EXT=-Xms512M -Xmx512M -Xmn128m" \
      -p 9876:9876 \
      foxiswho/rocketmq:4.8.0 \
      sh mqnamesrv
```

映射本地目录`logs`权限一定要设置为 777 权限，否则启动不成功

#### 2. broker

##### broker 无 目录映射

```bash
docker run -d \
      --name rmqnamesrv \
      -e "JAVA_OPT_EXT=-Xms512M -Xmx512M -Xmn128m" \
      -p 9876:9876 \
      foxiswho/rocketmq:4.8.0 \
      sh mqbroker -c /home/rocketmq/conf/broker.conf
```

##### broker 目录映射

```bash
docker run -d  -v $(pwd)/logs:/home/rocketmq/logs -v $(pwd)/store:/home/rocketmq/store \
      -v $(pwd)/conf:/home/rocketmq/conf \
      --name rmqbroker \
      -e "NAMESRV_ADDR=rmqnamesrv:9876" \
      -e "JAVA_OPT_EXT=-Xms512M -Xmx512M -Xmn128m" \
      -p 10911:10911 -p 10912:10912 -p 10909:10909 \
      foxiswho/rocketmq:4.8.0 \
      sh mqbroker -c /home/rocketmq/conf/broker.conf
```

注意

> 如果你的微服务没有使用`docker`,那么需要把`/etc/rocketmq/broker.conf` 配置文件中的`brokerIP1=192.168.0.253` 这个启用，IP 地址填写 你docker 所在 宿主机的IP ，否则报错
>
> 映射本地目录`logs`权限一定要设置为 777 权限，否则启动不成功

**重点**：这边我一直启动不成功，原因是我本地没有broker.conf这个文件，这个文件需要自己创建。

如果一切正常，NameServer和Broker一会儿就会安装好，为了管理上的方便，rocketmq console也是必不可少的工具了，通过上面查询的方式找到需要启动的版本，启动方式如下：

```bash
docker run -d --name rmqconsole -p 8180:8080 --link rmqserver:namesrv\
 -e "JAVA_OPTS=-Drocketmq.namesrv.addr=namesrv:9876\
 -Dcom.rocketmq.sendMessageWithVIPChannel=false"\
 -t styletang/rocketmq-console-ng
```














