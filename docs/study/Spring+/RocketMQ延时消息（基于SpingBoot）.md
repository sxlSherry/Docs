# RocketMQ延时消息及自定义重试次数和重试间隔时间（基于SpingBoot）

​	RocketMQ 的文档上写了支持延时消息，但是它不支持任意时间自定义的延迟消息，仅支持内置预设值的延迟时间间隔的延迟消息。

​	我呢之前业务用RabbitMQ的时候用了一个延时消息发送的方案，用代码逻辑实现的，算是分享一种实现思路吧。

## Rocket支持的延时消息

​	RocketMQ预设值的延迟时间间隔一共有18个等级，分别是：1s、 5s、 10s、 30s、 1m、 2m、 3m、 4m、 5m、 6m、 7m、 8m、 9m、 10m、 20m、 30m、 1h、 2h。对应的延时等级分别是1-18。

​	这个延时等级配置是在RocketMQ的配置文件中的，是 `messageDelayLevel`参数。也**支持修改**。修改后重启MQ就会生效。这边我就不演示了，就简单演示一下这个延时消息的发送。

### 发送消息的Service：

```java
		/**
     * 发送同步延时消息
     * @param message
     */
    public void sendDelayMsg(String message) {
        rocketMQTemplate.syncSend(topic, MessageBuilder.withPayload(message).build(),2000,6);
        log.info("发送同步延时消息:{}，发送时间：{}", message, System.currentTimeMillis());
    }
```

**此处延时等级是6，如果没有改过配置文件，则对应的是2分钟**

### 接受消息的Listener：

```java
@Slf4j
@Component
@RocketMQMessageListener(topic = "TEST_TOPIC", consumerGroup = "Test_Group")
public class MQConsumerListener implements RocketMQListener<TestMessageModel> {

    @Resource
    private MQProducerService mqProducerService;

    @Override
    public void onMessage(String message) {
        log.info("接收到消息：{}，接收时间：{}", message, System.currentTimeMillis());
    }

}
```

## 自定义重试次数和重试间隔时间

不在做任何设置的情况下，MQ是默认重试18次，按照延时等级重试。但我们的需求是最好能按照客户的要求，自定义重试的次数和时间。

一个思路，仅供参考。

自定义次数和重试时间，定义一个运行时间。如果报错则正常消费，重试次数+1，根据重试时间计算运行时间，监听的时候如果是第一次或者已过运行时间则放行，否则消费并重新放入队列。

### 通用的Message类

```java
@Data
@Builder
public class TestMessageModel {

    /**
     * 消息发送时间
     */
    private long sendTime;

    /**
     * 消息运行时间
     */
    private long runTime;

    /**
     * 尝试数
     */
    private int tryTimes;
  
    /**
     * 已经尝试次数
     */
    private int hasTryTimes;

    /**
     * 尝试间隔时间（秒）
     */
    private long intervals;

    /**
     * 消息内容
     */
    private String message;
}

```

### 发送消息的Service：

```java
		/**
     * 发送异步消息（通过线程池执行发送到broker的消息任务，执行完后回调：在SendCallback中可处理相关成功失败时的逻辑）
     * （适合对响应时间敏感的业务场景）
     */
    public void sendAsyncMsg(TestMessageModel testMessageModel) {
        rocketMQTemplate.asyncSend(topic, testMessageModel, new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {
                // 处理消息发送成功逻辑
                log.info("消息发送成功:{}", sendResult);
            }
            @Override
            public void onException(Throwable throwable) {
                // 处理消息发送异常逻辑
                log.info("消息发送失败:{}", throwable.getMessage());
            }
        });
    }
```

### 接受消息的Listener：

```java
@Slf4j
@Component
@RocketMQMessageListener(topic = "TEST_TOPIC", consumerGroup = "Test_Group")
public class MQConsumerListener implements RocketMQListener<TestMessageModel> {

    @Resource
    private MQProducerService mqProducerService;


    @Override
    public void onMessage(TestMessageModel testMessageModel) {
        try {
            //第一次或者到了过期时间再执行,不然重新放入队列
            if (testMessageModel.getExpTime() == null || testMessageModel.getExpTime() >= System.currentTimeMillis()) {
                log.info("接收到消息：{}，接收时间：{}", testMessageModel, System.currentTimeMillis());
                int n = 0 / 1;
            } else {
                mqProducerService.sendAsyncMsg(testMessageModel);
            }
        } catch (Exception e) {
            log.error("消息发送失败：{}", testMessageModel);
            //如果已经尝试次数小于需要尝试次数则继续重试，而且重试次数+1
            if (testMessageModel.getTryTimes() > testMessageModel.getHasTryTimes()) {
                testMessageModel.setExpTime(System.currentTimeMillis() + testMessageModel.getIntervals());
                testMessageModel.setHasTryTimes(testMessageModel.getHasTryTimes() + 1);
                mqProducerService.sendAsyncMsg(testMessageModel);
            }
        }

    }

}
```

### 测试方法

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = App.class)
public class AppTest 
{

    @Autowired
    private MQProducerService mqProducerService;

    @Test
    public void sendMessageTest()
    {
        TestMessageModel testMessageModel = TestMessageModel.builder().message("消息内容1111").intervals(120).tryTimes(3).build();
        mqProducerService.sendAsyncMsg(testMessageModel);
    }
}

```

