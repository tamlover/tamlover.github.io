---
layout: post
title: Quartz简单入门（三）
tags: [Quartz,springboot,bean]
---

Quartz集成springboot主要是集成spring，将scheduler注入容器管理，向job中注入bean。

这部分比较简单，就直接上代码了。

1、导入依赖包

```
compile('org.springframework.boot:spring-boot-starter-quartz')
```

2、注入scheduler，job

```java
package com.example.quartzdemo;

import org.quartz.Scheduler;
import org.quartz.spi.TriggerFiredBundle;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.config.AutowireCapableBeanFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;
import org.springframework.scheduling.quartz.SchedulerFactoryBean;
import org.springframework.scheduling.quartz.SpringBeanJobFactory;
import org.springframework.stereotype.Component;


/**
 * @author li.lu
 * @date 2018/10/15 13:24
 */
@Configuration
public class QuartzConfig {

    @Bean(name = "scheduler")
    public Scheduler scheduler(QuartzJobFactory quartzJobFactory) throws Exception {

        SchedulerFactoryBean factoryBean = new SchedulerFactoryBean();
        factoryBean.setJobFactory(quartzJobFactory);
        factoryBean.setConfigLocation(new ClassPathResource("quartz.properties"));
        factoryBean.afterPropertiesSet();

        Scheduler scheduler = factoryBean.getScheduler();
        scheduler.start();
        return scheduler;
    }

    /**
     * solve inject job spring bean is null
     */
    @Component("quartzJobFactory")
    class QuartzJobFactory extends SpringBeanJobFactory {

        @Autowired
        private AutowireCapableBeanFactory beanFactory;

        /**
         * override super createJobInstance
         */
        @Override
        protected Object createJobInstance(TriggerFiredBundle bundle) throws Exception {

            Object jobInstance = super.createJobInstance(bundle);

            beanFactory.autowireBean(jobInstance);

            return jobInstance;

        }

    }
}
```

3、job实现类

```java
package com.example.quartzdemo;

import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.quartz.QuartzJobBean;

/**
 * @author li.lu
 * @date 2018/10/24 10:47
 */
public class MyJob2 extends QuartzJobBean {

    @Autowired
    private QuartzTestInjectBean quartzTestInjectBean;

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        System.out.println("job execute");

        String str  = (String) context.getJobDetail().getJobDataMap().get("str");
        quartzTestInjectBean.print(str);
    }
}
```

接下来再写俩个测试类

```java
package com.example.quartzdemo;

import org.quartz.JobDetail;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.Trigger;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import static org.quartz.JobBuilder.newJob;
import static org.quartz.SimpleScheduleBuilder.simpleSchedule;
import static org.quartz.TriggerBuilder.newTrigger;

/**
 * @author li.lu
 * @date 2018/10/24 11:29
 */
@Service
public class QuartzTestService {

    @Autowired
    private Scheduler scheduler;

    public void test(String str) throws SchedulerException {

        JobDetail jobDetail = newJob(MyJob2.class)
                .withIdentity("myJob","group")
                .usingJobData("str", str)
                .build();

        Trigger trigger = newTrigger()
                .withIdentity("myTrigger", "myGroup")
                .startNow()
                .withSchedule(simpleSchedule()
                        .withIntervalInSeconds(1)
                        .withRepeatCount(10))
                .build();

        scheduler.scheduleJob(jobDetail, trigger);

    }
}

```

```java
    @Autowired
    private QuartzTestService quartzTestService;

    @RequestMapping(value = "test")
    public void test() throws SchedulerException {
        quartzTestService.test("hello");
    }
```

然后你起一下服务，调用一下test这个接口，如果有正常输出就说明ok了。这只是其中一种方式，如果你有其他方式也可以。

还有一点 @QuartzDatasource 这个注解我不知道该如何使用，和flyway的数据源配置好像不太一样。如果你知道的话，可不可以给我发一封邮件 ?我的邮箱是benniao1996@gmail.com，感谢！