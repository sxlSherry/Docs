# SpringBoot中yml内list、map写法

​	有时候需要在配置文件中储存list或者map，该篇就是记一下方法，以及我居然会遇到的坑。

​	yml内容如下（map和list各有两种写法）：

```yml
test:
  maps1: {key1: 1,key2: 2}

  maps2:
    key1: 3
    key2: 4

  list1:
    - 1
    - 2
    - 33
    
  list2: 1, 2, 3
```

​	需要获取参数的类（这个是直接获取test下面所有的参数）：

```java
@Data
@Component
@ConfigurationProperties(prefix = "test")
public class MapTest {
    private Map<String, String> maps1;
    private Map<String, String> maps2;
    private List<Integer> list1;
    private List<Integer> list2;
}
```

​	测试方法：

```java
		@Autowired
    private MapTest mapTest;

    @Test
    public void mapTest() {
        System.out.println(mapTest.toString());
    }

```

 	输出结果：

```java
MapTest(maps1={key1=1, key2=2}, maps2={key1=3, key2=4}, list1=[1, 2, 33], list2=[1, 2, 3])
```

​	

### 遇到的坑

​	到此为止都没有遇到什么问题，然后我就想试试，用@Value只获取一个参数试试，然后就报错了。

​	需要获取参数的类：

```java
@Data
@Component
public class MapTest {
    @Value("${test.maps1}")
    private Map<String, String> maps1;
    @Value("${test.maps2}")
    private Map<String, String> maps2;
    @Value("${test.list1}")
    private List<Integer> list1;
    @Value("${test.list2}")
    private List<Integer> list2;
}
```

​	测试的方法一模一样我就不重复放啦，结果：

```java
Caused by: java.lang.IllegalArgumentException: Could not resolve placeholder 'test.maps1' in value "${test.maps1}"
……
```

​	取不到。。。然后我就单独每个都测试了一下，list2是可以取到的。

​	最后找了一个方法：

​	yml内容如下（map和list各有两种写法）：

```yml
test:
  maps1: "{key1: 1,key2: 2}"
  list2: 1, 2, 3
```

​	需要获取参数的类：

```java
@Data
@Component
public class MapTest {
    @Value("#{${test.maps1}}")
    private Map<String, String> maps1;
    @Value("${test.list2}")
    private List<Integer> list2;
}
```

**以上这两种是可以用@Value取值的**。记录一下吧。

具体原因好像是list和map的方式获取到的是一个对象，不能用@Value接收。等下次有时间再去研究一下。



### 在yml中配置map，如果key中含有 / * 等特殊字符，key 需要加 "[ ]"

yml中的格式如下。

```yml
test:
  map1: 
    "default": 30
    "[aaa:bbb:ccc_ddd]": 20
    
  map2: {"default": 30,"[aaa:bbb:ccc_ddd]": 20}
```

