# Quartz Notes

official doc: https://www.quartz-scheduler.org/documentation/2.4.0-SNAPSHOT/ 

## Using Quartz 

To use a **Scheduler**, it must be instantiated using **org.quartz.SchedulerFactory**. 

```
SchedulerFactory schedulerFact = new org.quartz.impl.StdSchedulerFactory();
Scheduler scheduler = schedulerFact.getScheduler();
```

In Spring-Boot, the **Scheduler** is automatically populated by **QuartzAutoConfiguration**, the implementation is shown below. The **SchedulerFactoryBean** is a Spring's implementation that replace the **org.quartz.SchedulerFactory**, it doesn't implement the **SchedulerFactory**, but it works similarly. The **SchedulerFactoryBean** relies on the lifecycle methods that are invoked by the Spring container, which then creates, starts, and stops the **Scheduler** with the container.

```
@Bean
@ConditionalOnMissingBean
public SchedulerFactoryBean quartzScheduler(
            QuartzProperties properties,
            ObjectProvider<SchedulerFactoryBeanCustomizer> customizers, 
            ObjectProvider<JobDetail> jobDetails,
            Map<String, Calendar> calendars, 
            ObjectProvider<Trigger> triggers, 
            ApplicationContext applicationContext) {

    // .. do some configuration and customization
}
```

We may configure the **SchedulerFactoryBean** by providing the **SchedulerFactoryBeanCustomizer**. The **SchedulerFactoryBeanCustomizer** provides a method to access to the **SchedulerFactoryBean** that has not yet being populated, we can, for example, replace its **JobFactory** with our implementation.

```
public interface SchedulerFactoryBeanCustomizer {

	/**
	 * Customize the {@link SchedulerFactoryBean}.
	 * @param schedulerFactoryBean the scheduler to customize
	 */
	void customize(SchedulerFactoryBean schedulerFactoryBean);

}
```

E.g., Replace **JobFactory** inside **SchedulerFactoryBean** using **SchedulerFactoryBeanCustomizer**.

```
@Slf4j
@Configuration
public class SchedulerFactoryBeanConfig implements SchedulerFactoryBeanCustomizer {

    @Autowired
    private ManagedBeanJobFactory managedBeanJobFactory;

    @Override
    public void customize(SchedulerFactoryBean schedulerFactoryBean) {
        schedulerFactoryBean.setJobFactory(managedBeanJobFactory);
        log.info("Configured JobFactory to use: {}", managedBeanJobFactory.getClass());
    }
}
```

***Once a scheduler is instantiated, it can be started, placed in stand-by mode, and shutdown. But once a scheduler is shutdown, it cannot be restarted.***

**Trigger** do not fire, i.e., execute the job, if the *Scheduler* is **not started** or in **paused** state. We start a **Scheduler** by calling **Scheduler.start()**.

**JobDetail** is a template object that describes a **Job**, when using Quartz, we don't create the **Job** instance directly, we provide a class through the **JobDetail**, then Quartz or Spring will attempt to create a new **Job** instance based on the class. If it's in Spring-Boot, the bean's dependencies are autowired using **SpringBeanJobFactory**, which is configured by the **QuartzAutoConfiguration** as well.

The snippet below describes how **JobDetail**, **Trigger** and **Job** work together.

```
// define the job and tie it to our HelloJob class
JobDetail job = newJob(HelloJob.class)
    .withIdentity("myJob", "group1")
    .build();

// Trigger the job to run now, and then every 40 seconds
Trigger trigger = newTrigger()
    .withIdentity("myTrigger", "group1")
    .startNow()
    .withSchedule(simpleSchedule()
        .withIntervalInSeconds(40)
        .repeatForever())
    .build();

// Tell quartz to schedule the job using our trigger
scheduler.scheduleJob(job, trigger);
```

## Quartz API 

Key interfaces in Quartz:

- **Scheduler**
    - API for scheduler
- **Job**
    - an interface to be implemented by a job
- **JobDetail** 
    - used to define instances of Jobs.
- **Trigger**
    - used to define schedule of a job
- **JobBuilder**
    - used to build **JobDetail** 
- **TriggerBuilder**
    - used to build **Trigger** 
- **ScheduleBuilder**
    - used to describe schedule for **Trigger**

As shown in the snippet below, we created a schedule using **SimpleScheduleBuilder**. 

```
Trigger trigger = newTrigger()
    .withIdentity("myTrigger", "group1")
    .startNow()
    .withSchedule(SimpleScheduleBuilder.simpleSchedule()
        .withIntervalInSeconds(40)
        .repeatForever())
    .build();
```

There are four types of **SchedulerBuilder**:

- CalendarIntervalScheduleBuilder
    - used to define calendar time (day, week, month, year) interval-based schedules
- CronScheduleBuilder
    - used to define cron expression based schedules
- DailyTimeIntervalScheduleBuilder
    - used to define time-specific interval-based schedules
        - e.g., endTimeOfDay, startTimeOfDay
- SimpleSchedulerBuilder
    - used to define literal based schedules

## Jobs and Triggers 

A job must implements the **Job** interface, that is shown below:

```
public interface Job {

    public void execute(JobExecutionContext context)
        throws JobExecutionException;
}
```

The **JobExecutionContext** represents the runtime environment of the job, which provides access to **Scheduler**, **Trigger**, **Calendar**, **JobDetail**, and **JobDataMap**. Notice that the **JobDataMap** is read-only.

**Trigger** is responsible for firing the job, the job is run by one of the scheduler thread. **CronTrigger** and **SimpleTrigger** are the commonly used trigger types.

- CronTrigger
- SimpleTrigger 
- DailyTimeIntervalTrigger 

## Identities

Both **Job**s and **Trigger**s are given identifying keys. The **Job**'s identify being the **JobKey** and the **Trigger**'s identity being the **TriggerKey**. The keys consists of an **id** and **group**, within the same **group**, the **id** must be unique.

## Nature of Job and JobDetail 

When we provide **Scheduler** a **JobDetail**, it contains the class of the **Job**, every time **Scheduler** executes the job, it creates a new instance of the class using **default constructor**. Once the job is finished, the instance is GC by the JVM, and thus normally won't have any states in it. To provide properties for the job, we can simply use **JobDataMap** in **JobDetail**. 

## JobDataMap

**JobDataMap** is simply a key-value map. 

```
JobDetail job = newJob(DumbJob.class)
    .withIdentity("myJob", "group1") // name "myJob", group "group1"
    .usingJobData("jobSays", "Hello World!")
    .usingJobData("myFloatValue", 3.141f)
    .build();
```

**Trigger** can also have **JobDataMap** associated with it, it's useful when a **Job** is associated with multiple **Trigger**s, then each **Trigger** may add some extra parameters in the **JobDataMap**. Notice that the **JobDataMap** from **JobDetail** and **Trigger** are merged into a single one in **JobExecutionContext** during job execution.

```
public void execute(JobExecutionContext context) throws JobExecutionException {

    JobDataMap dataMap = context.getMergedJobDataMap();  

}
```

## TriggerBuilder and Common Trigger Attributes

Properties of a **Trigger** can be set using **TriggerBuilder**.

- job_key
    - identity of the job that should execute when this trigger fires
- start_time
    - the time when the trigger will start to have effect
- end_time
    - the time when the trigger will no longer be in effect
- priority
    - the priority of a trigger, when there are only N scheduler threads, and we want to trigger M (M > N) jobs at the **same time**, then we will use **priority** of triggers to determine which N triggers fire first. ***The priority value is only used to compare between triggers that fires at the same time.***
- misfire_instruction
    - **Misfire instruction** specifies what should the trigger do when the fire is missed (e.g., no available threads). Each **Trigger** type will have its own instruction policy defined on the interface.
- calendar
    - Quartz provides the **Calendar** class that we can use to exclude dates and times that we don't want the **Trigger** to fire. For example, we may create a **CronTrigger** and add it into **Scheduler**, then we may create a **Calendar** to exclude the business days. The **Calendar** can also use cron expression as well.

When we want to create a special **Trigger**, for example, one with cron expression, we will use the **ScheduleBuilder** as well. For example, the snippet below shows how to create a **CronTrigger** using **TriggerBuilder** and **CronScheduleBuilder**, notice that the **CronScheduleBuilder** is implementation of **ScheduleBuilder**.

```
Trigger t = TriggerBuilder.newTrigger()
                .withSchedule(
                        CronScheduleBuilder
                                .cronSchedule(tv.getCronExpr())
                                .withMisfireHandlingInstructionFireAndProceed()
                )
                .withIdentity(tv.getJobName())
                .startAt(new Date())
                .build();
```

Spring also provides a factory (same as builder) for **CronTrigger**, it's **CronTriggerFactoryBean**, the code using **CronTriggerFactoryBean** that is equivalent to above snippet is shown below.

```
CronTriggerFactoryBean factoryBean = new CronTriggerFactoryBean();
factoryBean.setName(tv.getJobName());
factoryBean.setStartTime(new Date());
factoryBean.setCronExpression(tv.getCronExpr());
factoryBean.setMisfireInstruction(SimpleTrigger.MISFIRE_INSTRUCTION_FIRE_NOW);
factoryBean.afterPropertiesSet();
return factoryBean.getObject();
```

## Calendar

**Calendar** objects are used to exclude dates and times for the trigger. A **Trigger** is associated with a **Calendar** using **Calendar**'s name. The example below shows how **Calendar** object and **Trigger** are registered. 

```
scheduler.addCalendar("week_day_only", new WeeklyCalendar(), true, true);

TriggerBuilder.newTrigger()
        .modifiedByCalendar("week_day_only");
        ...
        .build();
```

In this example, we use **WeeklyCalendar** to exclude saturday and sunday. We may also use other **Calendar** implementation as well.

Some of the implementation of **Calendar**:

- AnnualCalendar
- CronCalendar
- DailyCalendar 
- HolidayCalendar
- MonthlyCalendar
- WeeklyCalendar

## SimpleTrigger

Misfire Instruction:

- MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY
- MISFIRE_INSTRUCTION_FIRE_NOW
- MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_EXISTING_REPEAT_COUNT
- MISFIRE_INSTRUCTION_RESCHEDULE_NOW_WITH_REMAINING_REPEAT_COUNT
- MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_REMAINING_COUNT
- MISFIRE_INSTRUCTION_RESCHEDULE_NEXT_WITH_EXISTING_COUNT

## CronTrigger

Sub-expressions delimited by space in a Cron Expression: 

- Seconds
    - 0-59
- Minutes
    - 0-59
- Hours
    - 0-23
- Day-of-Month
    - 1-31, but need to be careful about how many days are in the given month
- Month
    - 0-11
    - JAN, FEB, MAR, APR, MAY, JUN, JUL, AUG, SEP, OCT, NOV, DEC
- Day-of-Week
    - 1-7 (1 for saturday)
    - SUN, MON, TUE, WED, THU, FRI, SAT
    - e.g., 'WED' for wednesday
    - e.g., 'MON-FRI' for monday to friday
    - e.g., 'MON-FRI,SAT' for monday to friday, including saturday 
- Year (optional field)

Special characters:

- '**\***' mean '**every**' 
- '**/**' means **incrementing**, the format is '`[starting_at]/[every_N_th]`', e.g., '0/10' in minutes means every 10th minutes of the hour, starting at 0.
- '**?**' is allowed for day-of-month and day-of-week, it means '**no specific value**', it's useful when one of the field in these two fields is specified, and the remaining doesn't have any value to specify
- '**L**' is allowed for day-of-month and day-of-week, it means '**last**', if 'L' is used in day-of-month field, it means the last day of the month. However, if it is used in day-of-week, it means '7', which is not saturday. A **offset** may be specified, e.g., 'L-3' in day-of-month means the third to last day of the month.
- '**W**' is used in day-of-month field, it means the weekday that is nearest to the given day, e.g., '15W' meaning the nearest day to the 15th of the month.
- '**#**' is used in day-of-week field, it means the nth weekday of the month, e.g., 'FRI#3' means the 3rd friday of the month.

Misfire instruction:

- MISFIRE_INSTRUCTION_IGNORE_MISFIRE_POLICY
- MISFIRE_INSTRUCTION_DO_NOTHING
- MISFIRE_INSTRUCTION_FIRE_NOW

## TriggerListener and JobListener

**TriggerListener** receives events related **Trigger**s. 

- Trigger firing
- Trigger mis-firing
- Trigger completion

```
public interface TriggerListener {

    public void triggerFired(Trigger trigger, JobExecutionContext context);

    public boolean vetoJobExecution(Trigger trigger, JobExecutionContext context);

    public void triggerMisfired(Trigger trigger);

    public void triggerComplete(Trigger trigger, JobExecutionContext context,
            int triggerInstructionCode);
}
```

**JobListener** receives events related to **Job**s.

- Job to be executed
- Job was executed 

```
public interface JobListener {

    public void jobToBeExecuted(JobExecutionContext context);

    public void jobExecutionVetoed(JobExecutionContext context);

    public void jobWasExecuted(JobExecutionContext context,
            JobExecutionException jobException);

}
```

To implement these listeners we can extend **JobListenerSupport** and **TriggerListenerSupport**. Notice that for both the **JobListener** and **TriggerListener** (except`triggerMisfired()`), their events are published by the **JobRunShell**. 

A new **JobRunShell** is created every time a **Job** is to be executed, and it's basically an environment that calls the listeners, runs the **Job**, and catch exceptions. Similarly, a **JobRunShell** is created by the **JobRunShellFactory**, so we may also implement **JobRunShellFactory** for more customization of the **JobRunShell**.

The listeners are able to **veto/reject** the job execution, if the listener choose to **veto** the job by returning true in `vetoJobExecution()` or `jobExecutionVetoed()`, the job won't execute.

We can register a listener using **ListenerManager** and a matcher.

```
scheduler
    .getListenerManager()
    .addJobListener(myJobListener, jobKeyEquals(jobKey("myJobName", "myJobGroup")));
```

## Scheduler Listener

**SchedulerListener** receives events related the **Scheduler**.

- Addition of job or trigger 
- Removal of job or trigger
- Errors within the Scheduler
- Scheduler being shutdown
- and so on 

```
public interface SchedulerListener {

    public void jobScheduled(Trigger trigger);

    public void jobUnscheduled(String triggerName, String triggerGroup);

    public void triggerFinalized(Trigger trigger);

    public void triggersPaused(String triggerName, String triggerGroup);

    public void triggersResumed(String triggerName, String triggerGroup);

    public void jobsPaused(String jobName, String jobGroup);

    public void jobsResumed(String jobName, String jobGroup);

    public void schedulerError(String msg, SchedulerException cause);

    public void schedulerStarted();

    public void schedulerInStandbyMode();

    public void schedulerShutdown();

    public void schedulingDataCleared();
}
```

To add a **SchedulerListener**, we still use the **ListenerManager**:

```
scheduler
    .getListenerManager()
    .addSchedulerListener(myListener);
```

## Job Store

- RAMJobStore
    - Simplest and performant implementation that use RAM 
    - Quartz configuration:
        - 'org.quartz.jobStore.class=org.quartz.simpl.RAMJobStore'
    - Spring-boot configuration:
        - see **org.springframework.boot.autoconfigure.quartz.QuartzProperties**
- JDBCJobStore
    - Schema scripts are provided that create a bunch of qrtz_* tables, and the jobs are persisted there
    - For Spring-Boot, the `LocalDataSourceJobStore` class is used instead for Spring-managed data source
    - Quartz configuration
        - 'org.quartz.jobStore.class=org.quartz.impl.jdbcjobstore.JobStoreTX'
        - 'org.quartz.jobStore.class=org.quartz.impl.jdbcjobstore.JobStoreCMT'
        - 'org.quartz.jobStore.tablePrefix=QRTZ_'
        - 'org.quartz.jobStore.driverDelegateClass=org.quartz.impl.jdbcjobstore.StdJDBCDelegate'
    - Spring-boot configuration:
        - see **org.springframework.boot.autoconfigure.quartz.QuartzProperties**
- TerracottaJobStore
    - Without using Database
    - Quartz configuration 
        - 'org.quartz.jobStore.class = org.terracotta.quartz.TerracottaJobStore'
        - 'org.quartz.jobStore.tcConfigUrl = localhost:9510'
     
## Quartz Configuration 

Major components that can be configured:

- ThreadPool 
- JobStore
- DataSources (if necessary)
- Scheduler 

***All the above, including listeners, are configured in SchedulerFactory.***

### ThreadPool

The default thread pool provided is **org.quartz.simpl.SimpleThreadPool**, the number of threads are fixed, never grow or shrink. This thread pool implementation is loaded by a **SchedulerFactory**, and by default, it's **StdSchedulerFactory**. 

The **StdSchedulerFactory** looks for a property **org.quartz.threadPool.class**, and use this property to load the class of the **ThreadPool**. Other properties with prefix **org.quartz.threadPool** are considered as properties of this selected implementation of **ThreadPool**, and are loaded into the instance using reflect. Properties shown below are the default configuration.

```
org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
org.quartz.threadPool.threadCount: 10
org.quartz.threadPool.threadPriority: 5
org.quartz.threadPool.threadsInheritContextClassLoaderOfInitializingThread: true
```

The way it loads the properties for the **ThreadPool** instance is by looking for a method with signature '`set${property_suffix}`', for example, for 'threadCount', it looks for method 'setThreadCount', and then it simply calls the method and pass in the property value.

Part of the implementation in **StdSchedulerFactory**.

```
String tpClass = cfg.getStringProperty(PROP_THREAD_POOL_CLASS, SimpleThreadPool.class.getName());

if (tpClass == null) {
    initException = new SchedulerException(
            "ThreadPool class not specified. ");
    throw initException;
}

try {
    tp = (ThreadPool) loadHelper.loadClass(tpClass).newInstance();
} catch (Exception e) {
    initException = new SchedulerException("ThreadPool class '"
            + tpClass + "' could not be instantiated.", e);
    throw initException;
}
tProps = cfg.getPropertyGroup(PROP_THREAD_POOL_PREFIX, true);
try {
    setBeanProps(tp, tProps);
} catch (Exception e) {
    initException = new SchedulerException("ThreadPool class '"
            + tpClass + "' props could not be configured.", e);
    throw initException;
}
```

### JobStore

The way the **JobStore** is configured is similar to **ThreadPool**, except that there isn't much to configure really. We may implement the **JobStore**, and replace the default configuration by setting value for property **org.quartz.jobStore.class**.

Below are the default configuration.

```
org.quartz.jobStore.misfireThreshold: 60000
org.quartz.jobStore.class: org.quartz.simpl.RAMJobStore
```
