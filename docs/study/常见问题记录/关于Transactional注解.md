#关于Transactional注解

###Transactional注解失效原因查询

​	之前公司用JFinal的框架，发现在一些情况下用注解操作事务时会失效，有些阴影，所以特地查了下用springboot时用事务会失效的情况，防止犯错。

1. 方法是不是public；
2. 异常类型是不是unchecked异常：

   如果想让checked异常也回滚，在注解上面写明异常类型即可:

   @Transactional(rollbackFor=Exception.class)

   类似还有norollbackFor，自定义不回滚的异常

3. 数据库引擎要支持事务，如果是Mysql，注意表要使用支持事务的引擎，比如innaodb，如果是myisam，事务是不起作用的；
4. 是否开启了对注解的解析；

5. spring是否扫描到你使用注解事务的这个类所在的包；

6. 检查是不是同一个类中的方法调用(如a方法调用同一个类中的b方法)；

7. 异常是不是被catch住了；



###动态数据源切换失效

​	最近遇到了一个问题，用了动态数据源切换，可还是报错，切换没有生效。问题是在别的方法中切换是有效的，但是单单在这个方法中失效，研究了一下，发现方法上加了**@Transactional**注解，发现开启事务后不能动态切换数据源（~！应该是可以想到的点）。

​	切换数据库是AOP调用，的确会触发，但数据源真正切换的关键是 AbstractRoutingDataSource 的 determineCurrentLookupKey() 被调用，此方法是在open connection时触发，事务是在connection层面管理的，启用事务后，一个事务内部的connection是复用的，所以就算AOP切了数据源字符串，但是数据源并不会被真正修改。。。