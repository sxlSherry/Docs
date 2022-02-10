# Maven新增本地jar包

pom.xml加依赖

```xml
<dependency>
  <groupId>cn.zzd</groupId>
  <artifactId>zwdd</artifactId>
  <version>1.2.0</version>
  <scope>system</scope>
  <systemPath>${pom.basedir}/libs/zwdd-sdk-java-1.2.0.jar</systemPath>
</dependency>
```



打包处加配置

```xml
<configuration>
  <includeSystemScope>true</includeSystemScope>
</configuration>
```

