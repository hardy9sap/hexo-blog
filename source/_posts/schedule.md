title: Spring Boot 集成 Quartz 实现动态任务调度
categories: Coding
tags: []
toc: false
date: 2017-12-20 20:54:41
---


最近使用 spring boot、 quartz、H2(内存数据库) 以及 RabbitMQ 等实现了一个动态的任务管理系统，可以动态的进行任务的创建、修改、暂停、运行以及删除操作，并且使用了 RabbitMQ 消息队列实现了定时任务系统与具体业务系统的解耦，再也不需要每次加个定时任务都上线一次了。<!-- more -->

## Java 实现定时任务的几种方式对比

目前 Java 系统中实现调度任务的方式大体有一下三种：

- 使用 JDK 自带的 `java.util.Timer` 及 `java.util.TimerTask` 类实现
- 使用 Spring 定时任务
- 使用第三方插件 Quartz

如果是在纯粹的 Java 环境需要实现定时任务毫无疑问就使用 JDK 自带的`java.util.concurrent.ScheduledExecutorService` 替代 `Timer & TimerTask` 实现即可，这种场景一般比较简单，也不存在集群的问题。

如果是集成 Spring 框架开发应用，则使用 Spring 的 `@Scheduled` 注解实现，简洁方便省事。但是此类应用很可能是集群部署，因此需要通过一定的途径避免集群环境下任务被多次调用的现象发生。常见的方法有使用 Redis 存储一个会过期的常量锁，每台容器执行器先读取锁变量值判断任务是否已被执行；另一种常见的方法是只让指定IP的容器执行定时任务（存在单点的问题）。

如果在集群环境下，想实现定时任务的可视化管理，或者想做一个统一的定时任务应用，亦或者定时任务的场景非常复杂，则建议使用企业级应用系统常用的 Quartz，而且现在 Spring 或者 Springboot 集成Quartz 也非常方便。

## Spring Boot + Quartz 任务调度系统预览

源码：https://github.com/jiwenxing/springboot-quartz

预览：    
![](//
jverson.oss-cn-beijing.aliyuncs.com/206751c9ac95c7860f087a02e5f2fd9f.jpg)


这里也将常规的 Quartz 与 Spring 的整合过程记录如下。

## 实现步骤

### 添加依赖

```xml
<!-- Includes spring's support classes for quartz -->
<dependency>
	<groupId>org.springframework</groupId>
	<artifactId>spring-context-support</artifactId>
</dependency>
<dependency>
	<groupId>org.quartz-scheduler</groupId>
	<artifactId>quartz</artifactId>
	<version>2.2.1</version>
</dependency>
```

### 创建 job

创建一个 Job 类，该类需要继承 QuartzJobBean 或者实现 Job 方法

```java
public class SampleJob implements Job {
	@Autowired
	private SampleService sampleService;
	@Override
	public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
    	// execute task inner quartz system
    	// spring bean can be @Autowired
    	sampleService.hello(jobName);
	}

}
```

### 配置 jobDetail、jobTrigger 及 schedule 实例

```java
@Bean
public JobDetailFactoryBean sampleJobDetail() {
    return createJobDetail(SampleJob.class);
}

@Bean(name = "sampleJobTrigger")
public SimpleTriggerFactoryBean sampleJobTrigger(@Qualifier("sampleJobDetail") JobDetail jobDetail,
                                                 @Value("${samplejob.frequency}") long frequency) {
    return createTrigger(jobDetail, frequency);
}

@Bean
public Scheduler schedulerFactoryBean(DataSource dataSource, JobFactory jobFactory,
                                      @Qualifier("sampleJobTrigger") Trigger sampleJobTrigger) throws Exception {
    SchedulerFactoryBean factory = new SchedulerFactoryBean();
    // this allows to update triggers in DB when updating settings in config file:
    factory.setOverwriteExistingJobs(true);
    factory.setDataSource(dataSource);
    factory.setJobFactory(jobFactory);

    factory.setQuartzProperties(quartzProperties());
    factory.afterPropertiesSet();

    Scheduler scheduler = factory.getScheduler();
    scheduler.setJobFactory(jobFactory);
    scheduler.scheduleJob((JobDetail) sampleJobTrigger.getJobDataMap().get("jobDetail"), sampleJobTrigger);

    scheduler.start();
    return scheduler;
}
```


## 注意点

### 引入 spring-context-support 依赖（非必须）

在使用 Spring 集成 Quartz 的时候，一定不要忘记引入 spring-context-support 这个包，该依赖包含支持UI模版（Velocity，FreeMarker，JasperReports），邮件服务，脚本服务(JRuby)，缓存Cache（EHCache），任务计划Scheduling（uartz）等方面的类。

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context-support</artifactId>
    <version>4.2.6.RELEASE</version>
</dependency>
```

### job 中无法注入业务类

Job 类中，注入 XXService 的时候，会报空指针异常，因为 Job 对象是 Quartz 通过反射创建的，而业务 service 是通过 Spring 创建的，所以在 Job 类中使用由 Spring 管理的对象就会报空指针异常。

Quartz 中有一个 JobFactory 接口，负责生成 Job 类的实例。那我们是不是可以通过自定义实现这个接口在创建 Job 的时候给予依赖注入的特性，实现如下所示，最后在 SchedulerFactoryBean 中设置新的 JobFactory 即可。

```java
public final class SpringJobFactory extends SpringBeanJobFactory implements
        ApplicationContextAware {

    private transient AutowireCapableBeanFactory beanFactory;

    @Override
    public void setApplicationContext(final ApplicationContext context) {
        beanFactory = context.getAutowireCapableBeanFactory();
    }

    @Override
    protected Object createJobInstance(final TriggerFiredBundle bundle) throws Exception {
        final Object job = super.createJobInstance(bundle);
        beanFactory.autowireBean(job);
        return job;
    }
}

@Configuration
public class SchedulerConfig {
    @Autowired
    private SpringJobFactory springJobFactory;
    @Bean
    public SchedulerFactoryBean schedulerFactoryBean() {
        SchedulerFactoryBean schedulerFactoryBean = new SchedulerFactoryBean();
        schedulerFactoryBean.setJobFactory(springJobFactory);
        return schedulerFactoryBean;
    }
}
```

## REFERENCES

- [https://github.com/davidkiss/spring-boot-quartz-demo](https://github.com/davidkiss/spring-boot-quartz-demo)
- [quartz与spring实现任务动态管理](//lixuguang.iteye.com/blog/2256478)
- [Spring 3整合Quartz 2实现定时任务](https://www.dexcoder.com/selfly/article/308)
- [Spring 中使用 Quartz 时，Job 无法注入spring bean的问题](https://blog.seveniu.com/post/spring-quartz-autowired/)
