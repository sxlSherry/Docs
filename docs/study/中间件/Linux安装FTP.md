# Linux安装FTP

[TOC]

##一、FTP的安装

### 1. 安装vsftpd

查看是否已经安装vsftpd ：rpm -qa|grep vsftpd

卸载ftp：rpm -e vsftpd

安装ftp：yum -y install vsftpd

设置开机启动：chkconfig vsftpd on

### 2. 管理vsftpd相关命令：

启动vsftpd: service vsftpd start 
停止vsftpd: service vsftpd stop 
重启vsftpd: service vsftpd restart

### 3. 配置防火墙（如果防火墙开启，需要配置）

打开/etc/sysconfig/iptables文件 
vi /etc/sysconfig/iptables 

在REJECT行之前添加如下代码 
-A RH-Firewall-1-INPUT -m state –state NEW -m tcp -p tcp –dport 21 -j ACCEPT

保存和关闭文件，重启防火墙 
service iptables restart

###4. FTP用户管理

#### 创建FTP用户并指定分组和主目录

useradd -d /data/ftp -s /sbin/nologin -g ftp -G root ftpUser

解析： 
useradd 添加用户ftpUser 
-d 指定用户根目录为/data/ftp 
-s 指定shell脚本为/sbin/nologin，表示不允许shell登录 
-g 创建分组ftp 
-G 指定root分组 
PS：创建有问题可以删除重新创建 userdel -r ftpUser

#### 设定密码

passwd ftpUser

#### 设置访问权限

chown ftpUser /data/ftp

###5. 配置vsftpd

1. 默认的配置文件是/etc/vsftpd/vsftpd.conf，可以用文本编辑器打开。 

   vim /etc/vsftpd/vsftpd.conf

2. 添加ftp用户

   下面是添加ftpuser用户，设置根目录为/home/wwwroot/ftpuser,禁止此用户登录SSH的权限，并限制其访问其它目录。 
   \#chroot_list_enable=YES 
   \# (default follows) 
   \#chroot_list_file=/etc/vsftpd.chroot_list

   改为 
   chroot_list_enable=YES 
   \# (default follows) 
   chroot_list_file=/etc/vsftpd/chroot_list

3. 编辑文件chroot_list: 
   vi /etc/vsftpd/chroot_list

   内容为ftp用户名,每个用户占一行,如：

   peter 
   john 
   ftpUser

4. 超时设置(可以不设置使用默认的) 
   idle_session_timeout=2400 #空闲连接超时时间40分钟 
   data_connection_timeout=120 #数据传输超时时间2分钟

5. 用户连接选项(可以不设置，使用默认的) 
   max_clients=100 #可接受的最大client数目 
   max_per_ip=10 #每个ip的最大client数目

6. 重新启动vsftpd 
   service vsftpd restart

7. 测试 
   windows 开始 —-> 输入CMD —–> ftp 47.114.62.126
   提示，输入用户名，然后输入密码。

   或者浏览器输入ftp://47.114.62.126进行测试。

## 二、Java使用FTP

​	原本代码是连接的Windows服务器的ftp，后来改为连接Linux服务器的ftp时，遇到了一些问题，有些小坑需要注意。

### 1. 创建文件夹报错

​	先贴上原有代码：

```java
// 判断ftp目录是否存在，如果不存在则创建目录，包括创建多级目录
String s = "/"+storePath;//基础路径
String[] dirs = s.split("/");
ftp.changeWorkingDirectory("/");//1
//按顺序检查目录是否存在，不存在则创建目录
for(int i=1; dirs!=null&&i<dirs.length; i++) {
    if (StringUtils.isEmpty(dirs[i])) {
        continue;
    }
    if(!ftp.changeWorkingDirectory(dirs[i])) {
        if(ftp.makeDirectory(dirs[i])) {			//2
            if(!ftp.changeWorkingDirectory(dirs[i])) {
                return null;
            }
        }else {
            return null;
        }
    }
}
```

​	这段代码在连接的Windows服务器的ftp的时候是正常的，但是在连接Linux服务器的时候，//2处创建文件夹失败。查看源码之后发现报错是权限不足。可是ftp服务器上我明明设置好了权限，查询了好多设置设置权限的方法，统统没用。

​	后来终于查到是**Linux服务器不允许直接在根目录创建文件夹**，代码在//1处更改文件夹路径时候，直接更改为之前给用户设定的路径，不要使用“/”就可以继续执行了！！！！

### 2. 上传文件为空

​	先贴上原有接上代码：

```java
// 设置文件操作目录
ftp.changeWorkingDirectory(storePath);
// 设置文件类型，二进制
ftp.setFileType(FTPClient.BINARY_FILE_TYPE);
// 设置缓冲区大小
ftp.setBufferSize(3072);
//获取文件名
String fileName = multipartFile.getOriginalFilename();
//输入流
in = (FileInputStream) multipartFile.getInputStream();
ftp.enterLocalPassiveMode();//1
//写入文件
boolean result = ftp.storeFile(new String((newFileName.toString()+"."+fileNames[fileNames.length-1]).getBytes("GBK"),"ISO_8859_1"), in);//2

```

​	这段代码在连接的Windows服务器的ftp的时候是正常的，但是在Linux服务器上//2处返回失败，查看服务器上的文件，文件存在，但是大小为0。

​	查了很久，都说要加上//1处的这个方法，改为被动模式才行，可是试了很多次还是不行（简直怀疑人生=。=）。第二天想到是不是本人电脑是Mac的原因，叫同事用Windows电脑试一下，竟然上传成功！！！！@_@于是，就不管三七二十一就部署啦～可是，部署到Linux服务器上，还是有一样的问题！！！

​	解决方法是加上**ftp.setFileTransferMode(FTP.STREAM_TRANSFER_MODE);**，终于让我找到啦！！！在Linux和Mac服务器上要设置传输方式为流方式。

###3. 文件名为中文乱码问题

​	这个问题好解决，由于Linux默认的中文编码是utf-8，所以在读取文件名的时候，编码改为**utf-8**就可以啦～



## 总结

​	对于Linux服务器，ftp安装和使用都不是很熟悉，所以检查错误的时候很耗费时间，在有空余时间的时候还是要多研究研究。记下遇到的问题，以后再次遇到就能轻松解决啦～





