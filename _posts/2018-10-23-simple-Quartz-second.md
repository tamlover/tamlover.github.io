---
layout: post
title: Quartz简单入门（二）
tags: [Quartz,misfire,jobDataMap,persistence]
---

## 知识点补充

上篇是仅仅是一个简单入门，只是希望做一个抛砖引玉的作用。下面我提一下几个知识点，如果有需求，可以再找找其他资料。

scheduler作为一个独立的容器，它不仅仅可以开启，也可以终止，或者添加，删除任务等等功能。

trigger有几种不同的类型，其中CronTrigger可以实现极为复杂的定时任务。trigger的misfire机制，可以设置不同misfire策略，告诉quartz如何如果发生misfire该如何处理。

misfire策略

```
MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY
MISFIRE_INSTRUCTION_FIRE_NOW
MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT
MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT
MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT
MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_EXISTING_COUNT
```

misfire配置

```java
 
Trigger trigger = newTrigger()
        .withIdentity("trigger", "group")
        .withSchedule(simpleSchedule()
            .withIntervalInMinutes(1)
            .repeatForever()
            .withMisfireHandlingInstructionNextWithExistingCount())
        .build();
```



job中使用jobdataMap注入数据

```java
        JobDataMap jobDataMap = new JobDataMap();
        jobDataMap.put("hello", "world");
        jobDataMap.put("love", "china");

        JobDetail jobDetail = newJob(MyJob.class)
                .usingJobData(jobDataMap)
                .build();
```

数据注入job，job如何拿出来呢

```
    @Override
    public void execute(JobExecutionContext context)throws JobExecutionException {
        JobDataMap dataMap = context.getJobDetail().getJobDataMap();
        String hello = dataMap.getString("hello");
        String = dataMap.getString("love");
        //then...
    }
```



该篇是接上篇的一个扩展，再讲一下如何将job和trigger持久化操作。持久化有什么好处呢，假设你是在内存中做定时任务，遇到服务宕机，机房停电，你所有的任务就丢失了，但是如果你存在数据库里，当服务再次启动的时候，Quartz的misfire机制就会启动，触发trigger，执行job。

## Quartz持久化

1、导入需要的jar包

```
    //与数据库连接的包，我这边是postgresql,你是其他的导入其他的即可
    compile group:'org.postgresql',name:'postgresql',version:'42.1.1'
    //c3p0，开源的数据库连接池
    compile group: 'com.mchange', name: 'c3p0', version: '0.9.5.2'
```

2、建表

[下载](http://www.quartz-scheduler.org/downloads/)官方的包，解压一下，在里面你可以找到对应数据库的建表代码。然后执行sql语句将表建到数据库中。

3、修改配置(quartz.propeties)

```
org.quartz.scheduler.instanceName = MyScheduler
org.quartz.threadPool.threadCount = 3
org.quartz.threadPool.class=org.quartz.simpl.SimpleThreadPool

//修改jobsclass,为jobStoreTx,上一篇为RAM，默认也是RAM
org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX
//委托类，执行sql使用
org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.PostgreSQLDelegate
//数据库表的前缀
org.quartz.jobStore.tablePrefix = QRTZ_
//数据源配置
org.quartz.jobStore.dataSource = myDS
org.quartz.dataSource.myDS.driver = org.postgresql.Driver
org.quartz.dataSource.myDS.URL = jdbc:postgresql://localhost/testdb
org.quartz.dataSource.myDS.user = postgres
org.quartz.dataSource.myDS.password = postgres
```

然后你再运行上一篇中例子，执行的过程中，你会在数据库中trigger,jobdetail等表中会看到记录，job执行完后记录也被删除，这就成功了。