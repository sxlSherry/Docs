# Docker镜像源修改

docker默认的源为国外官方源，下载速度较慢，可改为国内，加速

方案一

修改或新增 /etc/docker/daemon.json

vi /etc/docker/daemon.json
{

“registry-mirrors”: [“http://hub-mirror.c.163.com”]

}

systemctl restart docker



# linux更换源合集

http://t.zoukankan.com/davis12-p-14293721.html