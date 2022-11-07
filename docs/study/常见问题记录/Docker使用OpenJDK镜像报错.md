# Docker使用OpenJDK镜像导致没有汉字字体报错

### 问题

​	使用Docker部署Jar包，下载excel报错，但是本地却没有问题。查看日志，报错：

```
java.lang.NullPointerException: null
    at sun.awt.FontConfiguration.getVersion(FontConfiguration.java:1264)
    at sun.awt.FontConfiguration.readFontConfigFile(FontConfiguration.java:219)
    at sun.awt.FontConfiguration.init(FontConfiguration.java:107)
    at sun.awt.X11FontManager.createFontConfiguration(X11FontManager.java:774)
    at sun.font.SunFontManager$2.run(SunFontManager.java:431)
    at java.security.AccessController.doPrivileged(Native Method)
    at sun.font.SunFontManager.<init>(SunFontManager.java:376)
    at sun.awt.FcFontManager.<init>(FcFontManager.java:35)
```

### 分析

**OpenJDK比OracleJDK简化了一些功能，默认没有安装的字体，所以后端绘制验证码所要用到Java的AWT组件就报出空指针。**

有三种解决思路：

- 1、将字体拷贝到服务器并和容器做映射

- 2、更换Docker镜像为 OracleJDK

- 3、基于操作系统安装FontConfig组件

  综合比较快捷的是第三种方式：下面主要介绍一下第三种方式。

  

## 基于操作系统安装FontConfig组件

### 在线安装

- 1、在centos7系统安装FontConfig

```
yum install fontconfig
```

- 2、修改dockerfile ，添加一行，安装字体 `ttf-dejavu`

```
RUN apk add --update font-adobe-100dpi ttf-dejavu fontconfig
```

- 3、重启docker容器

```
docker restart 容器ID或容器名
```



但是我们使用的政务服务器是内网环境，所以不能使用上面的方式，只能用**离线安装**的方式。

### 离线安装 

#### 1. 离线安装 fontconfig

- 下载 fontconfig离线包：fontconfig-2.13.0-4.3.el7.x86_64.rpm
- 上传到服务器，执行命令：

```
rpm -ivh fontconfig-2.13.0-4.3.el7.x86_64.rpm  --nodeps --force
```

在 `/usr/share` 下多出 `fontconfig` 和 `fonts` 目录。



#### 2. 安装字体 ttf-dejavu

- 下载字体 `ttf-dejavu` ： https://packages.msys2.org/package/mingw-w64-x86_64-ttf-dejavu (失效自行查找)
- 上传字体：将 字体文件打包上传到服务器 `/usr/share/fonts` 目录，解压

- 刷新字体：`fc-cache --force` , 刷新完成之后可以使用 `fc-list` 查看安装的字体

![image.png](https://ucc.alicdn.com/pic/developer-ecology/1bfea994b69b489bb078a858443d0d5d.png)



#### 3. 容器中安装字体

- 将上传的 `ttf-dejavu` 字体文件夹拷贝到容器`/usr/share/fonts` 目录中

```
docker cp -a TTF/  [容器id]:/usr/share/fonts
```

- 进入容器 ，刷新字体

```
# 进入容器
docker exec -it [容器id] bash
# 刷新字体
fc-cache --force
```

- `fc-list` 就可以看到安装的 ttf-dejavu 字体

![image.png](https://ucc.alicdn.com/pic/developer-ecology/4b9f5fb0871c40c49677188891f0c8a7.png)

#### 4. 重启容器

最后重启容器：

```
docker restart [容器id]
```


