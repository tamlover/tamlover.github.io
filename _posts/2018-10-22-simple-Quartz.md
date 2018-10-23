---
layout: post
title: Quartz简单入门
tags: [Quartz,schedule,trigger,job]
---

## Quartz是什么？能干什么？

​	Quartz是一个完全使用Java开发，开源的任务调度框架。它的核心功能是定时、定期的执行任务，比如x年x月的每个星期五上午8点到9点，每隔10分钟执行一次任务。它有3个核心要素：调度器(Scheduler)、任务(Job)、触发器(Trigger)。它能够承载成千上万的任务调度，并且支持集群。它支持将数据存储到数据库中以实现持久化，并支持绝大多数的数据库。它将任务与触发设计为松耦合，即一个任务可以对应多个触发器，这样能够轻松构造出极为复杂的触发策略

## 先跑起来

1、导入Quratz Jar包（你可以在官网下载，可以用mvn库里导入）

​	我用的是Gradle导入：

```
compile group: 'org.quartz-scheduler', name: 'quartz', version: '2.3.0'
```



2、添加quartz.properties配置文件

​	配置里面可以配置很多东西，所以quartz可以很灵活的使用，这里先配置几个需要的

```
org.quartz.scheduler.instanceName = MyScheduler
org.quartz.threadPool.threadCount = 3
org.quartz.jobStore.class = org.quartz.simpl.RAMJobStore
```



3、示例程序

```java
package com.example.quartzdemo;

import org.quartz.JobDetail;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.Trigger;
import org.quartz.impl.StdSchedulerFactory;

import static org.quartz.JobBuilder.newJob;
import static org.quartz.SimpleScheduleBuilder.simpleSchedule;
import static org.quartz.TriggerBuilder.newTrigger;

/**
 * @author li.lu
 * @date 2018/10/23 17:01
 */
public class QuartzDemo {
    public static void main(String[] args) throws SchedulerException {

        //获取调度中心的实例
        Scheduler scheduler = StdSchedulerFactory.getDefaultScheduler();

        //start
        scheduler.start();

        //定义job
        JobDetail job = newJob(MyJob.class)
                .withIdentity("myJob", "myGroup")
                .build();

        //定义trigger，每隔1秒执行一次，执行10次
        Trigger trigger = newTrigger()
                .withIdentity("myTrigger", "myGroup")
                .startNow()
                .withSchedule(simpleSchedule()
                        .withIntervalInSeconds(1)
                        .withRepeatCount(10))
                .build();

        //注册到调度中心
        scheduler.scheduleJob(job, trigger);

        //shutdown
//        scheduler.shutdown();

    }
}


```

```java
package com.example.quartzdemo;

import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;

/**
 * @author li.lu
 * @date 2018/10/23 17:18
 */
public class MyJob implements Job {
    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        System.out.println("执行任务:"+System.currentTimeMillis());
    }
}

```

这个时候如果你能看到你的控制窗每隔1秒输出 “执行任务:XXXXX”，就说一个简单任务调度例子明成功了。好，现在我们来研究它是怎么跑起来的。

## 怎么跑起来的

Quartz的主要元素是scheduler（调度器），trigger（触发器），job（任务），其中trigger和job是元数据，scheduler是任务调度的管理器。scheduler是由schedulerFactory创建，scheduler创建好后，分别将trigger和job注册进scheduler，最后scheduler负责调度任务。



### scheduler

scheduler是任务调度的管理器，其中有俩个重要的组件：ThreadPool（线程池），JobStore（任务存储）。

ThreadPool线程池，任务都交给线程池来做。

JobStore做任务存储，在Quartz中，trigger和job都需要储存下来才可以使用。

Schedule有三种：

1. StdScheduler

2. RemoteMBeanScheduler

3. RemoteScheduler

其中StdScheduler最常用。

scheduler.start（）方法就是启动scheduler，scheduler.scheduleJob(job, trigger)将job和trigger注册进来。

当我们scheduler.start（）后，在调用.shutdown()之前，scheduler会一直在跑。

#### schedulerFactory

schedulerFactory负责创建scheduler，有DirectSchedulerFactory或者StdSchedulerFactory俩类。前者直接构建，基于代码初始化配置，而后者基于配置文件（quartz.properties）构建。  StdSchedulerFactory使用比较多。



### Job

Job是一个接口类，这个接口只有一个方法

```java
public interface Job {
    void execute(JobExecutionContext context)
        throws JobExecutionException;

}
```

当和Job绑定的Trigger被触发时就会调用到这个execute（）方法。

#### JobDetail

jobDetail 负责实例化一个job，并且设置这个job的属性

而MyJob这个类就是你要做的任务了，在上面的代码中你可以看到，我实现了Job这个接口后，重写execute（）方法，输出一串内容。

这里提一个点，给Job传输数据用JobDataMap而不是构造器。



### Trigger

Trigger用于定义任务调度时间规则。

#### 属性

1. startTime和endTime 
   所有的Trigger都包含startTime、endTime这两个属性

2. 优先级(Priority) 
   触发器的优先级值默认为5，不过注意优先级是针对同一时刻来说的，在同一时刻优先级高的先触发。假如一个触发器被执行时间为3:00，另外一个为3:01，那么肯定是先执行时间为3:00的触发器。

3. 错失触发(Misfire)策略

   在任务调度中，并不能保证所有的触发器都会在指定时间被触发，假如Scheduler资源不足或者服务器重启的情况，就好发生错失触发的情况。

#### 类型

在任务调度Quartz中，Trigger主要的触发器有：SimpleTrigger，CalendarIntervelTrigger DailyTimeIntervalTrigger，CronTrigger。

上面的例子中用的就是SimplTrigger，一种最基本的触发器。

这种DSL风是不是看起来特别清爽，希望我们以后都做这样的类初始化！

其中几种trigger类型这里就不讲了，看名字也大概能猜到他们的功能吧！



## 总结

用一个生活中的例子来总结吧。scheduler就像一座车站，trigger比如是一张时刻表，job就是一辆要等待出发的车。将时刻表和车都放入车站中（注册），并将时刻表和车绑定。车站在轮询时刻表时，发现一辆车该出发了，就让那辆车出发（线程池去执行），它要去哪开多快车站不管（MyJob中excute()方法），车站让它出发就行（调用excute（））。

这里注意一下，一张时刻表（trigger）只能绑定一辆车（job），但是一辆车（job）却可以绑定多张时刻表（trigger）。



