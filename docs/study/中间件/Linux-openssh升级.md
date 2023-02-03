# Linux-openssh升级

先说一下背景吧，服务器被扫描出来好多高危、中危漏洞，必须要把openssh升级到8.8或以上才可以。

### 升级说明

1. 升级OpenSSH后，原有公钥失效，信任关系需要重新配置；
2. 升级过程需要停止sshd服务，会导致ssh和sftp无法使用；如果你使用的是xshell的ssh2连接则不会受影响。
3. 升级需要关闭防火墙服务；
4. 升级前需要开启telnet，防止升级失败，系统无法登录，对应的防火墙需要开启23端口，安装需要telnet相关包（推荐通过系统ISO安装）
5. 升级过程中需要刷新lib库：ldconfig -v；
6. 升级顺序：顺序是zlib库-> openssl -> openssh；
7. 升级需要gcc、make、perl、zlib、zlib-devel、pam、pam-devel依赖包；

### 安装并启用telnet（用xshell的ssh2连接跳过，我这边只做记录，没有尝试）

#### 1. 安装telnet服务端

```undefined
yum -y install xinetd telnet-server
```

#### 2. 默认情况下，系统是不允许root用户telnet远程登录的。如果要使用root用户直接登录，需设置如下内容。或者可以添加一个可以登录的用户，登录并su到root用户（建议采用此方法，保证系统安全）。此步骤可跳过！

允许root用户通过telnet登陆：
 编辑/etc/pam.d/login，注释掉下面这行

```bash
vi /etc/pam.d/login

#auth [user_unknown=ignore success=ok ignore=ignore default=bad] pam_securetty.so
```

#### 3. 添加超级用户登陆设备至/etc/securetty文件

```bash
cp /etc/securetty /etc/securetty.bak
echo "pts/0" >> /etc/securetty
echo "pts/1" >> /etc/securetty
echo "pts/2" >> /etc/securetty
```

#### 4. 开启root用户远程登陆。此步骤可跳过!

编辑/etc/pam.d/remote，注释下列这行：

```bash
vi /etc/pam.d/remote
#auth required pam_securetty.so
```

#### 5. 重启telnet和xinetd服务【telnet服务依赖于xinetd服务】

```bash
systemctl restart telnet.socket
systemctl restart xinetd
```

PS：如果开启了防火墙，需要将23端口（系统默认23为telnet端口）添加到防火墙允许的端口的列表中。



### 更新前准备

#### 1. 关闭系统防火墙

```bash
systemctl stop firewalld.service
```

#### 2. 安装相关依赖包

```bash
yum -y install gcc make perl zlib zlib-devel pam pam-devel

rpm -qa | egrep "gcc|make|perl|zlib|zlib-devel|pam|pam-devel"
```

#### 3. 停止ssh服务

```bash
systemctl stop sshd
```

备份ssh配置文件

```undefined
cp -r /etc/ssh /etc/ssh.old 
```

#### 4. 查看系统原有openssh包

```bash
rpm -qa | grep openssh
根据上面查询出的结果，卸载系统里原有Openssh（一般有三个包，全部卸载）
rpm -e --nodeps  xxxxxxxxxx
rpm -e --nodeps openssh-server-7.4p1-21.el7.x86_64
rpm -e --nodeps openssh-7.4p1-21.el7.x86_64
rpm -e --nodeps openssh-clients-7.4p1-21.el7.x86_64
卸载完成后执行rpm -qa | grep openssh，确保没有回显
rpm -qa | grep openssh
```



### 更新zlib（这一步我看很多教程上都有就记录下来了，我没有进行这一步，也升级成功了）

```bash
#下载安装包，解压并编译安装
wget https://nchc.dl.sourceforge.net/project/libpng/zlib/1.2.11/zlib-1.2.11.tar.gz
tar -xzvf zlib-1.2.11.tar.gz
cd zlib-1.2.11
./configure --prefix=/usr/local/zlib
make
make install
```



### 更新openssl

```bash
#下载安装包
wget https://www.openssl.org/source/openssl-1.1.1l.tar.gz --no-check-certificate
#备份
mv /usr/bin/openssl /usr/bin/openssl.old
mv /usr/include/openssl /usr/include/openssl.old
#解压并进入到目录
tar -zxvf openssl-1.1.1l.tar.gz
cd openssl-1.1.1l/
#编译并安装
./config --prefix=/usr/local/openssl
make
make install
#openssl库文件进行链接
ln -s /usr/local/openssl/bin/openssl /usr/bin/openssl
ln -s /usr/local/openssl/include/openssl /usr/include/openssl
#重新加载openssl库文件，否则执行openssl命令时会提示找不到共享库，这一步也很重要，如果没有这一步，后面升级openssh时会configure过不去
echo "/usr/local/openssl/lib" >> /etc/ld.so.conf
ldconfig -v
#查看版本
openssl version
```



### 更新openssh

#### 1. 升级

```bash
#权限要改为600,否则会报警
chmod 600 /etc/ssh/* 
#下载安装包
wget -c http://ftp.jaist.ac.jp/pub/OpenBSD/OpenSSH/portable/openssh-8.8p1.tar.gz
#备份
cp /usr/bin/ssh /usr/bin/ssh.bak
cp /usr/sbin/sshd /usr/sbin/sshd.bak
mv /etc/ssh /etc/ssh.bak
#解压并进入到目录
tar -zxvf openssh-8.8p1.tar.gz
cd openssh-8.8p1.tar.gz
#编译并安装
./configure –prefix=/usr/local/openssh –with-zlib=/lib64 –with-ssl-dir=/usr/local/openssl –with-ssl-engine –with-pam –with-pam-service=sshd –with-privsep-user=sshd –with-selinux –with-privsep-path=/var/empty –with-md5-passwords 
make -j 2
make install
# 修改启动文件和pam
cp ./contrib/redhat/sshd.init /etc/init.d/sshd
cp -a contrib/redhat/sshd.pam /etc/pam.d/sshd.pam
mv /usr/lib/systemd/system/sshd.service /usr/lib/systemd/system/sshd.service_bak
```

**configure选项说明：**
–prefix 指定安装路径
–with-zlib 指定zlib库路径，需要安装zlib和zlib-devel包
–with-ssl-dir 指定openssl路径，如果之前升级ssl时没有执行ldconfig 的操作，那么此处configure时会报错。
–with-ssl-engine 启用硬件引擎支持
–with-pam? 启用pam，需要安装pam和pam-devel包
–with-privsep-user? 指定权限分离一户
–with-selinux 激活selinux支持
–with-privsep-path 指定权限分离的chroot目录
–with-md5-passwords 允许使用md5密码



#### 2. 修改配置文件

编辑/etc/ssh/sshd_config，最下方添加PermitRootLogin yes

我在重启后出现了sftp无法连接的情况，需要注释掉 `Subsystem sftp /usr/lib/openssh/sftp-server`，添加 `Subsystem sftp internal-sftp`



#### 3. 重启sshd

```bash
systemctl daemon-reload
systemctl restart sshd
systemctl status sshd
# 查看sshd版本
sshd -V
```



**注：所有的安装包都可以根据地址找到最新的下载。**