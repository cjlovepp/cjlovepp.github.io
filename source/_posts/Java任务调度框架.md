---
title: Java任务调度框架
date: 2018-08-09 11:14:32
tags: 
    - Java
categories:
    - Java
---

## 前言
在进行Web开发时，我们经常会遇到任务调度的需求，要求你在某个特定的时间段执行某个任务。这里对JAVA的几种任务调度的实现做下学习记录：Timer、ScheduleExecutor、Quartz、JCronTab。

<!--more-->

## Timer
`java.util.Timer`提供了一种简单的任务调度实现方法：
```java
    package com.ibm.scheduler; 
    import java.util.Timer; 
    import java.util.TimerTask; 

    public class TimerTest extends TimerTask {

        private String jobName = ""; 

        public TimerTest(String jobName) { 
            super(); 
            this.jobName = jobName; 
        } 

        @Override 
        public void run() { 
            System.out.println("execute " + jobName); 
        } 

        public static void main(String[] args) {
            Timer timer = new Timer(); 
            long delay1 = 1 * 1000; 
            long period1 = 1000; 
            // 从现在开始 1 秒钟之后，每隔 1 秒钟执行一次 job1 
            timer.schedule(new TimerTest("job1"), delay1, period1); 
            long delay2 = 2 * 1000; 
            long period2 = 2000; 
            // 从现在开始 2 秒钟之后，每隔 2 秒钟执行一次 job2 
            timer.schedule(new TimerTest("job2"), delay2, period2); 
        }
    }
```
输出结果：
```console
    execute job1 
    execute job1 
    execute job2 
    execute job1 
    execute job1 
    execute job2 
```
使用 `Timer` 实现任务调度的核心类是 `Timer` 和 `TimerTask`。其中Timer负责设定TimerTask的起始与间隔执行时间。使用者只需要创建一个 TimerTask 的继承类，实现自己的 run 方法，然后将其丢给 Timer 去执行即可。

Timer 的设计核心是一个 TaskList 和一个 TaskThread。Timer 将接收到的任务丢到自己的 TaskList 中，TaskList 按照 Task 的最初执行时间进行排序。TimerThread 在创建 Timer 时会启动成为一个守护线程。这个线程会轮询所有任务，找到一个最近要执行的任务，然后休眠，当到达最近要执行任务的开始时间点，TimerThread 被唤醒并执行该任务。之后 TimerThread 更新最近一个要执行的任务，继续休眠。

> Timer 的优点在于简单易用，但由于 **所有任务都是由同一个线程来调度**,因此所有任务都是串行执行的，同一时间只能有一个任务在执行，前一个任务的延迟或异常都将会影响到之后的任务。

## ScheduledExecutor
鉴于 Timer 的上述缺陷，Java 5 推出了基于线程池设计的ScheduledExecutor。其设计思想是，每一个被调度的任务都会由线程池中一个线程去执行，因此任务是并发执行的，相互之间不会受到干扰。需要注意的是，只有当任务的执行时间到来时，ScheduedExecutor才会真正启动一个线程，其余时间 ScheduledExecutor 都是在轮询任务的状态。
```java
package com.ibm.scheduler;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

public class ScheduledExecutorTest implements Runnable {
    private String jobName = "";

    public ScheduledExecutorTest(String jobName) {
        super();
        this.jobName = jobName;
    }

    @Override
    public void run() {
        System.out.println("execute " + jobName);
    }

    public static void main(String[] args) {
        ScheduledExecutorService service = Executors.newScheduledThreadPool(10);

        long initialDelay1 = 1;
        long period1 = 1;
        // 从现在开始1秒钟之后，每隔1秒钟执行一次job1
        service.scheduleAtFixedRate(
                new ScheduledExecutorTest("job1"), initialDelay1,
                period1, TimeUnit.SECONDS);

        long initialDelay2 = 1;
        long delay2 = 1;
        // 从现在开始2秒钟之后，每隔2秒钟执行一次job2
        service.scheduleWithFixedDelay(
                new ScheduledExecutorTest("job2"), initialDelay2,
                delay2, TimeUnit.SECONDS);
    }
}
```
输出：
```console
execute job1
execute job1
execute job2
execute job1
execute job1
execute job2
```
ScheduledExecutorService的两种常用方法：
- `scheduleAtFixedRate`:每次执行时间为上一次任务开始起向后推一个时间间隔，即每次执行时间为 :`initialDelay`, `initialDelay+period`, `initialDelay+2*period`, `...`；
- `scheduleWithFixedDelay`:每次执行时间为上一次任务结束起向后推一个时间间隔，即每次执行时间为：`initialDelay`, `initialDelay+executeTime+delay`, `initialDelay+2*executeTime+2*delay`。
> 由此可见，ScheduleAtFixedRate是基于固定时间间隔进行任务调度，ScheduleWithFixedDelay取决于每次任务执行的时间长短，是基于不固定时间间隔进行任务调度。

## Quartz
Quartz可以说是使用相当广泛的任务调度框架了，它够支持更多更负责的调度需求，同时Quartz在设计上将`Job`和`Trigger`分离实现了松耦合，这大大增强了任务配置的灵活性
### 简单的例子
用 Quartz 实现每星期二 16:38 的调度安排：
```java
package com.ibm.scheduler;
import java.util.Date;

import org.quartz.Job;
import org.quartz.JobDetail;
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.quartz.Scheduler;
import org.quartz.SchedulerFactory;
import org.quartz.Trigger;
import org.quartz.helpers.TriggerUtils;

public class QuartzTest implements Job {

    @Override
    //该方法实现需要执行的任务
    public void execute(JobExecutionContext arg0) throws JobExecutionException {
        System.out.println("Generating report - "
                + arg0.getJobDetail().getFullName() + ", type ="
                + arg0.getJobDetail().getJobDataMap().get("type"));
        System.out.println(new Date().toString());
    }
    public static void main(String[] args) {
        try {
            // 创建一个Scheduler
            SchedulerFactory schedFact = new org.quartz.impl.StdSchedulerFactory();
            Scheduler sched = schedFact.getScheduler();
            sched.start();
            // 创建一个JobDetail，指明name，groupname，以及具体的Job类名，
            //该Job负责定义需要执行任务
            JobDetail jobDetail = new JobDetail("myJob", "myJobGroup",
                    QuartzTest.class);
            jobDetail.getJobDataMap().put("type", "FULL");
            // 创建一个每周触发的Trigger，指明星期几几点几分执行
            Trigger trigger = TriggerUtils.makeWeeklyTrigger(3, 16, 38);
            trigger.setGroup("myTriggerGroup");
            // 从当前时间的下一秒开始执行
            trigger.setStartTime(TriggerUtils.getEvenSecondDate(new Date()));
            // 指明trigger的name
            trigger.setName("myTrigger");
            // 用scheduler将JobDetail与Trigger关联在一起，开始调度任务
            sched.scheduleJob(jobDetail, trigger);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```
结果：
```console
Generating report - myJobGroup.myJob, type =FULL
Tue Feb 8 16:38:00 CST 2011
Generating report - myJobGroup.myJob, type =FUL
Tue Feb 15 16:38:00 CST 2011
```
非常简洁地实现了一个复杂的任务调度。Quartz 设计的核心类包括 Scheduler, Job 以及Trigger。其中，Job负责定义需要执行的任务，Trigger 负责设置调度策略，Scheduler 将二者组装在一起，并触发任务开始执行。
### Job
只需要创建一个 Job 的继承类，实现 execute 方法。JobDetail 负责封装 Job 以及 Job 的属性，并将其提供给 Scheduler 作为参数。每次 Scheduler 执行任务时，首先会创建一个 Job 的实例，然后再调用 execute 方法执行。
Quartz 没有为 Job 设计带参数的构造函数，因此需要通过额外的 JobDataMap 来存储 Job 的属性。JobDataMap 可以存储任意数量的 Key，Value 对，例如：
```java
    jobDetail.getJobDataMap().put("myDescription", "my job description"); 
    jobDetail.getJobDataMap().put("myValue", 1998); 
    ArrayList<String> list = new ArrayList<String>(); 
    list.add("item1"); 
    jobDetail.getJobDataMap().put("myArray", list); 
```
JobDataMap 中的数据可以通过下面的方式获取：
```java
public class JobDataMapTest implements Job {

    @Override
    public void execute(JobExecutionContext context)
            throws JobExecutionException {
        //从context中获取instName，groupName以及dataMap
        String instName = context.getJobDetail().getName();
        String groupName = context.getJobDetail().getGroup();
        JobDataMap dataMap = context.getJobDetail().getJobDataMap();
        //从dataMap中获取myDescription，myValue以及myArray
        String myDescription = dataMap.getString("myDescription");
        int myValue = dataMap.getInt("myValue");
        ArrayList<String> myArray = (ArrayListlt;Strin>) dataMap.get("myArray");
        System.out.println("
                Instance =" + instName + ", group = " + groupName
                + ", description = " + myDescription + ", value =" + myValue
                + ", array item0 = " + myArray.get(0));
    }
}
```

### Trigger
Trigger 的作用是设置调度策略。Quartz 设计了多种类型的 Trigger，其中最常用的是 `SimpleTrigger` 和 `CronTrigger`。
#### SimpleTrigger
SimpleTrigger 适用于在某一特定的时间执行一次，或者在某一特定的时间以某一特定时间间隔执行多次。上述功能决定了 SimpleTrigger 的参数包括 `start-time`, `end-time`, `repeat count`, 以及 `repeat interval`。

- `Repeat count`: 重复次数取值为大于或等于零的整数，或者常量 SimpleTrigger.REPEAT_INDEFINITELY。

- `Repeat interval`: 执行间隔取值为大于或等于零的长整型。当 `Repeat interval` 取值为0并且 `Repeat count`取值大于零时，将会触发任务的并发执行。

- `Start-time` 与 e`nd-time` 取值为 java.util.Date。当同时指定 `end-time` 与 `repeat count` 时，优先考虑 `end-time`。一般地，可以指定 `end-time`，并设定 `repeat count` 为 REPEAT_INDEFINITELY。

以下是 SimpleTrigger 的构造方法：
```java
 public SimpleTrigger(String name, 
                       String group, 
                       Date startTime, 
                       Date endTime, 
                       int repeatCount, 
                       long repeatInterval) 
```
#### CronTrigger
`CronTrigger` 的用途更广，相比基于特定时间间隔进行调度安排的 SimpleTrigger，CronTrigger 主要适用于基于日历的调度安排。例如：每星期二的 16:38:10 执行，每月一号执行，以及更复杂的调度安排等。
CronTrigger 同样需要指定 start-time 和 end-time，其核心在于 Cron 表达式,由七个字段组成（补充中有关于Cron表达式的说明）
```
 Seconds 
 Minutes 
 Hours 
 Day-of-Month 
 Month 
 Day-of-Week 
 Year (Optional field) 
```
Job 与 Trigger 的松耦合设计是 Quartz 的一大特点，其优点在于同一个 Job 可以绑定多个不同的 Trigger，同一个 Trigger 也可以调度多个 Job，灵活性很强。
### Listener
除了上述基本的调度功能，Quartz 还提供了 listener 的功能。主要包含三种 listener：JobListener，TriggerListener 以及 SchedulerListener。当系统发生故障，相关人员需要被通知时，Listener 便能发挥它的作用。最常见的情况是，当任务被执行时，系统发生故障，Listener 监听到错误，立即发送邮件给管理员。下面给出 JobListener 的实例：
```java
import org.quartz.JobExecutionContext;
import org.quartz.JobExecutionException;
import org.quartz.JobListener;
import org.quartz.SchedulerException;


public class MyListener implements JobListener{

    @Override
    public String getName() {
        return "My Listener";
    }
    @Override
    public void jobWasExecuted(JobExecutionContext context,
            JobExecutionException jobException) {
        if(jobException != null){
            try {
                //停止Scheduler
                context.getScheduler().shutdown();
                System.out.println("Error occurs when executing jobs, shut down the scheduler ");
                // 给管理员发送邮件…
            } catch (SchedulerException e) {
                e.printStackTrace();
            }
        }
    }
}
```
使用者只需要创建一个 JobListener 的继承类，重载需要触发的方法即可。当然，需要将 listener 的实现类注册到 Scheduler 和 JobDetail 中：
```java
 sched.addJobListener(new MyListener()); 
 jobDetail.addJobListener("My Listener"); // listener 的名字
```
也可以将 listener 注册为全局 listener，这样便可以监听 scheduler 中注册的所有任务 :
```java
sched.addGlobalJobListener(new MyListener()); 
```
### JobStores
Quartz的另一显著优点在于持久化，即将任务调度的相关数据保存下来。这样，当系统重启后，任务被调度的状态依然存在于系统中，不会丢失。默认情况 下，Quartz采用的是org.quartz.simpl.RAMJobStore，在这种情况下，数据仅能保存在内存中，系统重启后会全部丢失。若想持久化数据，需要采用 org.quartz.simpl.JDBCJobStoreTX。
实现持久化的第一步，是要创建 Quartz 持久化所需要的表格。在 Quartz 的发布包 docs/dbTables 中可以找到相应的表格创建脚本。Quartz 支持目前大部分流行的数据库。本文以 DB2 为例，所需要的脚本为 tables_db2.sql。首先需要对脚本做一点小的修改，即在开头指明 Schema：
```sql
 SET CURRENT SCHEMA quartz; 
```
然后创建数据库 sched，执行 tables_db2.sql 创建持久化所需要的表格。

第二步，配置数据源。数据源与其它所有配置，例如 ThreadPool，均放在 quartz.properties 里：
```properties
# Configure ThreadPool 
 org.quartz.threadPool.class = org.quartz.simpl.SimpleThreadPool 
 org.quartz.threadPool.threadCount =  5 
 org.quartz.threadPool.threadPriority = 4 

 # Configure Datasources 
 org.quartz.jobStore.class = org.quartz.impl.jdbcjobstore.JobStoreTX 
 org.quartz.jobStore.driverDelegateClass = org.quartz.impl.jdbcjobstore.StdJDBCDelegate 
 org.quartz.jobStore.dataSource = db2DS 
 org.quartz.jobStore.tablePrefix = QRTZ_ 

 org.quartz.dataSource.db2DS.driver = com.ibm.db2.jcc.DB2Driver 
 org.quartz.dataSource.db2DS.URL = jdbc:db2://localhost:50001/sched 
 org.quartz.dataSource.db2DS.user = quartz 
 org.quartz.dataSource.db2DS.password = passw0rd 
 org.quartz.dataSource.db2DS.maxConnections = 5 
```
使用时只需要将 quatz.properties 放在 classpath 下面，再次运行之前的任务调度实例，trigger、job 等信息便会被记录在数据库中。重启服务后将数据库中记录的任务调度数据重新导入程序运行：
```java
package com.ibm.scheduler;
import org.quartz.Scheduler;
import org.quartz.SchedulerException;
import org.quartz.SchedulerFactory;
import org.quartz.Trigger;
import org.quartz.impl.StdSchedulerFactory;
 
public class QuartzReschedulerTest{ 
    public static void main(String[]args) throws SchedulerException{ 
    //初始化一个 Schedule Factory 
    SchedulerFactory schedulerFactory = new StdSchedulerFactory();
    //从 schedule factory 中获取 scheduler 
    Scheduler scheduler = schedulerFactory.getScheduler(); 
    // 从 schedule factory 中获取 trigger 
    Trigger trigger = scheduler.getTrigger("myTrigger","myTriggerGroup");
    // 重新开启调度任务
    scheduler.rescheduleJob("myTrigger",
    "myTriggerGroup",
     trigger); 
    scheduler.start();
    }
}
```
上面代码中，schedulerFactory.getScheduler() 将 quartz.properties 的内容加载到内存，然后根据数据源的属性初始化数据库的链接，并将数据库中存储的数据加载到内存。之后，便可以在内存中查询某一具体的 trigger，并将其重新启动。这时候重新查询 qrtz_simple_triggers 中的数据，发现 times_triggered 值比原来增长了。

## JCronTab
Crontab 是一个非常方便的用于 unix/linux 系统的任务调度命令。JCronTab 则是一款完全按照 crontab 语法编写的 java 任务调度工具。

## 总结
对于简单的基于起始时间点与时间间隔的任务调度，使用 Timer 就足够了；如果需要同时调度多个任务，基于线程池的 ScheduledTimer 是更为合适的选择；当任务调度的策略复杂到难以凭借起始时间点与时间间隔来描述时，Quartz 与 JCronTab 则体现出它们的优势。熟悉 Unix/Linux 的开发人员更倾向于 JCronTab，且 JCronTab 更适合与 Web 应用服务器相结合。Quartz 的 Trigger 与 Job 松耦合设计使其更适用于 Job 与 Trigger 的多对多应用场景。
## 补充
### Spring Boot集成Quartz

### cronExpression表达式介绍
|    字段    |      允许值       | 允许的特殊字符  |
|------------|-------------------|-----------------|
| 秒         | 0-59              | , - * /         |
| 分         | 0-59              | , - * /         |
| 小时       | 0-23              | , - * /         |
| 日期       | 1-31              | , - *   / L W C |
| 月份       | 1-12 或者 JAN-DEC | , - * /         |
| 星期       | 1-7 或者 SUN-SAT  | , - *   / L C # |
| 年（可选） | 留空, 1970-2099   | , - * /         |

如上面的表达式所示: 

“\*”字符被用来指定所有的值。如：”*“在分钟的字段域里表示“每分钟”。 

“-”字符被用来指定一个范围。如：“10-12”在小时域意味着“10点、11点、12点”。
 
“,”字符被用来指定另外的值。如：“MON,WED,FRI”在星期域里表示”星期一、星期三、星期五”. 

“?”字符只在日期域和星期域中使用。它被用来指定“非明确的值”。当你需要通过在这两个域中的一个来指定一些东西的时候，它是有用的。看下面的例子你就会明白。 


“L”字符指定在月或者星期中的某天（最后一天）。即“Last ”的缩写。但是在星期和月中“Ｌ”表示不同的意思，如：在月子段中“L”指月份的最后一天-1月31日，2月28日，如果在星期字段中则简单的表示为“7”或者“SAT”。如果在星期字段中在某个value值得后面，则表示“某月的最后一个星期value”,如“6L”表示某月的最后一个星期五。

“W”字符只能用在月份字段中，该字段指定了离指定日期最近的那个星期日。

“#”字符只能用在星期字段，该字段指定了第几个星期value在某月中



每一个元素都可以显式地规定一个值（如6），一个区间（如9-12），一个列表（如9，11，13）或一个通配符（如*）。“月份中的日期”和“星期中的日期”这两个元素是互斥的，因此应该通过设置一个问号（？）来表明你不想设置的那个字段。表7.1中显示了一些cron表达式的例子和它们的意义：

|           表达式           |                            意义                           |
|----------------------------|-----------------------------------------------------------|
| "0 0 12 * * ?"             |  每天中午12点触发                                         |
| "0 15 10 ? * *"            |  每天上午10:15触发                                        |
| "0 15 10 * * ?"            |  每天上午10:15触发                                        |
| "0 15 10 * * ? *"          |    每天上午10:15触发                                      |
| "0 15 10 * * ? 2005"       |    2005年的每天上午10:15触发                              |
| "0 * 14 * * ?"             |   在每天下午2点到下午2:59期间的每1分钟触发                |
| "0 0/5 14 * * ?"           |    在每天下午2点到下午2:55期间的每5分钟触发               |
| "0 0/5 14,18 * * ?"        |   在每天下午2点到2:55期间和下午6点到6:55期间的每5分钟触发 |
| "0 0-5 14 * * ?"           |   在每天下午2点到下午2:05期间的每1分钟触发                |
| "0 10,44 14 ? 3 WED"       |    每年三月的星期三的下午2:10和2:44触发                   |
| "0 15 10 ? * MON-FRI"      |     周一至周五的上午10:15触发                             |
| "0 15 10 15 * ?"           |    每月15日上午10:15触发                                  |
| "0 15 10 L * ?"            |   每月最后一日的上午10:15触发                             |
| "0 15 10 ? * 6L"           |     每月的最后一个星期五上午10:15触发                     |
| "0 15 10 ? * 6L 2002-2005" |    2002年至2005年的每月的最后一个星期五上午10:15触发      |
| "0 15 10 ? * 6#3"          |    每月的第三个星期五上午10:15触发                        |
