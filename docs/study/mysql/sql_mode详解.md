# sql_mode详解

[TOC]

​	sql_mode是一组语法校验规则。

​	为什么想写这个，mysql换个环境经常碰到有些语句报错，发现都是因为这个值的原因。本篇就记录一下常用各个模式的意义和各种因为规则存在的报错。



## sql_mode查看

1. 命令：

   查看全局sql_mode：```select @@GLOBAL.sql_mode```

   查看当前sql_mode：```select @@sql_mode```

2. 查看my.conf/my.ini/my.cnf配置文件



## 常见规则说明

### ONLY_FULL_GROUP_BY

#### 说明

​	对于GROUP BY操作，如果在SELECT中的列，没有在GROUP BY中出现，那么这个SQL是不合法的，因为列不在GROUP BY从句中。

#### 常见报错

​	如果sql_mode中有ONLY_FULL_GROUP_BY这个模式，查询语句如果在SELECT中的列，没有在GROUP BY中出现则会报错。例如下面的sql：```select name, age from person group by name```执行之后就会报错。报错内容如下：

```sql
Error Code: 1055. Expression #3 of SELECT list is not in GROUP BY clause and contains nonaggregated column ‘×××’ which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
```

​	解决方案这边都是一致的，所以我就放在最后一起写了。这边还有一个解决方案，可以使用函数ANY_VALUE()包含报错字段，即吧上面的sql改成```select name, ANY_VALUE(age) from person group by name```就不会报错了。



### NO_AUTO_VALUE_ON_ZERO

#### 说明

​	NO_AUTO_VALUE_ON_ZERO影响AUTO_INCREMENT列的处理。一般情况，在自增列插入NULL或0会生成下一个序列号。NO_AUTO_VALUE_ON_ZERO禁用0，因此只有NULL可以生成下一个序列号。如果需要将0保存到表的AUTO_INCREMENT列，该模式会很有用。

​	注：不推荐采用该惯例。例如，如果你用mysqldump转储表并重载，MySQL 遇到0值一般会生成新的序列号，生成的表的内容与转储的表不同。



###  STRICT_TRANS_TABLES

#### 说明

​	对于单个insert操作，插入单行数据与字段类型不兼容，则insert操作失败并回滚；插入多行数据，如果插入数据的第一行内容与字段类型不兼容，则insert操作失败并回滚，如果插入数据的第一行内容与字段类型兼容，但后续的数据行存在不兼容的情况，则兼容的数据正常插入，不兼容的数据会转换成符合字段类型的格式再插入，不会中断和回滚；



### NO_ZERO_IN_DATE和NO_ZERO_DATE

#### 说明

​	**NO_ZERO_IN_DATE**：z不允许插入日期或月份值为零的日期。但是不影响'0000-00-00'零值日期或者年份为零但是日期和月份值非零的日期值的插入(例如'0000-01-01')。支持 0000-00-00 0000-01-01 （年月日都为0，月日不为0）。

​	**NO_ZERO_DATE**：不允许插入'0000-00-00'这样的零日期，也就是说这种日期不是合法日期，不允许插入。你仍然可以用IGNORE选项插入零日期。支持 1000-00-00 0000-01-00 0000-00-01（年月日中任何一个不为0）。

​	若要控制0000-00-00这样的数据插入必须两个都设置。

#### 常见报错

```sql
ERROR 1067 (42000): Invalid default value for ‘xxx_time’
```

​	这里我之前遇到一个**STR_TO_DATE()**函数无效的问题，原因就是开启了**NO_ZERO_DATE**，记录一下。



### ERROR_FOR_DIVISION_BY_ZERO

#### 说明

​	在insert或update过程中，如果数据被零除，则产生错误而非警告。如果未给出该模式，那么数据被零除时Mysql返回NULL。



## 常见错误解决方法

### 1. 通过sql语句暂时性修改sql_mode

去掉需要去掉的，重新设置值

```sql
SET @@global.sql_mode ='去掉需要去除的模式之后的mode';
```

上面是改变了全局sql_mode，对于新建的数据库有效。对于已存在的数据库，则需要在对应的数据库下执行：

```sql
SET sql_mode ='去掉需要去除的模式之后的mode';
```

**注：**重启mysql数据库服务之后，mode还是会还原，所以这只是暂时性的。



### 2. 通过配置文件永久修改sql_mode

 mysql安装在服务器上和安装在本地，修改配置文件的方式有点区别。

#### Linux下修改配置文件

1）登录进入MySQL
使用命令 mysql -u username -p 进行登陆，然后输入密码，输入SQL：

```sql
show variables like ‘%sql_mode’;
```

2）编辑my.cnf文件
文件地址一般在：/etc/my.cnf，/etc/mysql/my.cnf
找到sql-mode的位置，去掉对应模式，
然后重启MySQL；
有的my.cnf中可能没有sql-mode，需要追加：

```
sql-mode=[mode]
```

注意要加入到**[mysqld]**下面，如加入到其他地方，重启后也不生效。

 3）修改成功后重启MySQL服务

```
service mysql restart
```

重启好后，再登录mysql，输入SQL：```show variables like ‘%sql_mode’```; 如果和设置的一致，就说明已经成功了。

 

#### window下修改配置文件

1）找到mysql安装目录，用记事本直接打开my.ini文件；

 2）编辑my.cnf文件，在[mysql]标签下追加sql-mode的内容：

```
sql-mode=[mode]
```

 3）重启mysql 服务。

 
