# SpringBoot实现动态增删启停定时任务

​		需求需要实现动态启停定时任务功能，比较广泛的做法是集成Quartz框架。但是由于项目本身就比较大，是里面的一个小需求，所以就希望尽量少的依赖其它框架，避免项目过于臃肿和复杂。

​		话不多说，上代码。

1. 添加执行定时任务的线程池配置类

   ```java
   @Configuration  
   public class SchedulingConfig {  
       @Bean  
       public TaskScheduler taskScheduler() {  
           ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();  
           // 定时任务执行线程池核心线程数  
           taskScheduler.setPoolSize(10);  
           taskScheduler.setRemoveOnCancelPolicy(true);  
           taskScheduler.setThreadNamePrefix("TaskSchedulerThreadPool-");  
           return taskScheduler;  
       }  
   } 
   
   ```

2. 添加ScheduledFuture的包装类

   ScheduledFuture是ScheduledExecutorService定时任务线程池的执行结果。

   ```java
   @Data
   public final class ScheduledTask {
       @Nullable
       volatile ScheduledFuture<?> future;
   
       /*** 取消定时任务*/
       public void cancel() {
           ScheduledFuture<?> future = this.future;
           if (future != null) {
               future.cancel(true);
           }
       }
   }
   ```

3. 添加Runnable接口实现类，被定时任务线程池调用，用来执行指定bean里面的方法。

   ```java
   @Data
   @Slf4j
   public class SchedulingRunnable implements Runnable{
       private String beanName;
       private String methodName;
   
       public SchedulingRunnable(String beanName, String methodName) {
           this.beanName = beanName;
           this.methodName = methodName;
       }
   
       @Override
       public void run() {
           log.info("定时任务开始执行 - bean：{}，方法：{}", beanName, methodName);
           long startTime =System.currentTimeMillis();
           try{
               Object target = SpringContextUtils.getBean(beanName);
               Method method = target.getClass().getDeclaredMethod(methodName);
               ReflectionUtils.makeAccessible(method);
               method.invoke(target);
           }catch(Exception ex) {
               log.error("定时任务执行异常 - bean：{}，方法：{}", beanName, methodName, ex);
           }
           long times = System.currentTimeMillis() - startTime;
           log.info("定时任务执行结束 - bean：{}，方法：{}，耗时：{} 毫秒", beanName, methodName, times);
       }
   
       @Override
       public boolean equals(Object o) {
           if (this == o){
               return true;
           }
           if (o == null || getClass() != o.getClass()) {
               return false;
           }
           SchedulingRunnable that = (SchedulingRunnable) o;
           return beanName.equals(that.beanName) && methodName.equals(that.methodName);
       }
   
       @Override
       public int hashCode() {
           return Objects.hash(beanName, methodName);
       }
   
   }
   ```

4. 添加定时任务注册类，用来增加、删除定时任务

   ```java
   @Slf4j
   @RequiredArgsConstructor
   @Configuration
   public class CronTaskRegistrar implements DisposableBean {
   
       private final TaskScheduler taskScheduler;
   
       private final Map<Runnable,ScheduledTask> scheduledTasks = new ConcurrentHashMap<>(16);
   
   
       public TaskScheduler getScheduler() {
           return this.taskScheduler;
       }
   
       public void addCronTask(Runnable task, String cronExpression) {
           addCronTask(new CronTask(task, cronExpression));
       }
   
       public void addCronTask(CronTask cronTask) {
           if (cronTask != null) {
           Runnable task=cronTask.getRunnable();
           if (this.scheduledTasks.containsKey(task)) {
               removeCronTask(task);
           }
           this.scheduledTasks.put(task, scheduleCronTask(cronTask));
   
       }
   
       }
       public void removeCronTask(Runnable task) {
   
           ScheduledTask scheduledTask= this.scheduledTasks.remove(task);
           if (scheduledTask != null) {
               scheduledTask.cancel();
           }
   
       }
       public ScheduledTask scheduleCronTask(CronTask cronTask) {
           ScheduledTask scheduledTask= new ScheduledTask();
           scheduledTask.future= this.taskScheduler.schedule(cronTask.getRunnable(), cronTask.getTrigger());
           return scheduledTask;
   
       }
   
       @Override
       public void destroy() {
           for (ScheduledTask task : this.scheduledTasks.values()) {
               task.cancel();
           }this.scheduledTasks.clear();
       }
   }
   ```

5. 定时任务数据库表设计

   ```sql
   CREATE TABLE `sys_job` (
     `job_id` bigint(200) NOT NULL AUTO_INCREMENT COMMENT '任务ID',
     `code` varchar(50) DEFAULT NULL COMMENT '编码',
     `bean_name` varchar(50) NOT NULL COMMENT ' bean名称',
     `method_name` varchar(50) NOT NULL COMMENT '方法名称',
     `cron_expression` varchar(100) DEFAULT NULL COMMENT 'cron表达式',
     `status` tinyint(4) DEFAULT NULL COMMENT '状态  0：暂停   1：正常',
     `ctime` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
     `utime` datetime DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '最后更新时间',
     `remark` varchar(255) DEFAULT NULL COMMENT '备注',
     PRIMARY KEY (`job_id`) USING BTREE
   ) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8mb4 ROW_FORMAT=DYNAMIC COMMENT='定时任务表';
   ```
   
6. 添加实现了CommandLineRunner接口的SysJobRunner类，当spring boot项目启动完成后，加载数据库里状态为正常的定时任务。

   ```java
   @Slf4j
   @Service
   public class SysJobRunner implements CommandLineRunner {
   
       @Autowired
       private SysJobService sysJobService;
   
       @Autowired
       private CronTaskRegistrar cronTaskRegistrar;
   
       @Override
       public void run(String... args) {
           // 初始加载数据库里状态为正常的定时任务
           List<SysJob> jobList = sysJobService.getBaseMapper().selectAllByStatus(true);
           if (CollectionUtils.isNotEmpty(jobList)) {
               for (SysJob job : jobList) {
                   SchedulingRunnable task = new SchedulingRunnable(job.getBeanName(), job.getMethodName());
                   cronTaskRegistrar.addCronTask(task, job.getCronExpression());
               }
               log.info("定时任务已加载完毕...");
           }
       }
   }
   
   ```

7. 测试任务

   ```java
@Slf4j
   @Component("PopulationTableTask")
   @RequiredArgsConstructor
   public class PopulationTableTask {
       public void task() throws Exception {
           log.info("开始执行人口专题库统计任务");
       }
   }
   ```

   

8. 测试代码

   因为我这边只需要对任务进行状态切换，所以我只写了一个方法。

   controller实现定时任务切换

   ```java
   /**
    * 定时任务启动/停止状态切换
    * @param id-任务id
    **/
   @GetMapping("updateStatus")
   public R<String> updateStatus(@RequestParam Integer id){
         sysJobService.updateStatus(modelId);
         return R.ok();
   }
   ```

   service方法

   ```java
   		private final CronTaskRegistrar cronTaskRegistrar;
   
       /**
        * 定时任务启动/停止状态切换
        */
       public void updateStatus(Integer id) {
           SysJob job = baseMapper.selectOneById(id);
           if (job == null) {
               throw new BusinessException("该任务不存在");
           }
           SchedulingRunnable task = new SchedulingRunnable(job.getBeanName(), job.getMethodName());
           //现在是关闭状态，改成开启
           if (!job.getStatus()) {
               cronTaskRegistrar.addCronTask(task, job.getCronExpression());
           } else {
               //现在是开启状态，改为关闭状态
               cronTaskRegistrar.removeCronTask(task);
           }
           job.setStatus(!job.getStatus());
           job.setUtime(new Date());
           updateById(job);
       }
   ```

8. 其他：工具类SpringContextUtils，用来从spring容器里获取bean

   ```java
   @Component  
   public class SpringContextUtils implements ApplicationContextAware {  
     
       private static ApplicationContext applicationContext;  
     
       @Override  
       public void setApplicationContext(ApplicationContext applicationContext)  
               throws BeansException {  
           SpringContextUtils.applicationContext = applicationContext;  
       }  
     
       public static Object getBean(String name) {  
           return applicationContext.getBean(name);  
       }  
     
       public static <T> T getBean(Class<T> requiredType) {  
           return applicationContext.getBean(requiredType);  
       }  
     
       public static <T> T getBean(String name, Class<T> requiredType) {  
           return applicationContext.getBean(name, requiredType);  
       }  
     
       public static boolean containsBean(String name) {  
           return applicationContext.containsBean(name);  
       }  
     
       public static boolean isSingleton(String name) {  
           return applicationContext.isSingleton(name);  
       }  
     
       public static Class<? extends Object> getType(String name) {  
           return applicationContext.getType(name);  
       }  
   }
   
   ```

   



​	