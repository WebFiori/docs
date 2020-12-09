# Background Tasks

<meta name="description" content="Background tasks are used to run even if there is no one is using the application. Learn about how to implement and use them here.">

In this page:

* [Introduction](#introduction)
* [Main Classes](#main-classes)
  * [The Class `AbstractJob`](#the-class-abstractjob)
  * [The Class `CronJob`](#the-class-cronjob)
  * [The Class `Cron`](#the-class-cron)
  * [The Class `InitCron`](#the-class-initcron)
* [Scheduling Jobs](#scheduling-jobs)
  * [Scheduling Job as a Closure](#scheduling-job-as-a-closure)
  * [Using the Class `AbstractJob`](#using-the-class-abstractjob)
* [Jobs Execution](#jobs-execution)
* []()
* []()

## Introduction

One of the features of the framework is the ability to schedule PHP code to run at specific time in the background. For example, it is possible to make your web application send reports at specific time using email or any comunication way. The framework give the developers all needed tools to schedule background tasks in a very simple way.

## Main Classes

The sub-system which is responsible for scheduling jobs consist of 4 main classes. In order to schedule jobs, we have to use at least the first one. The classes are:

* The class [`AbstractJob`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob)
* The class [`CronJob`](https://webfiori.com/docs/webfiori/framework/cron/CronJob)
* The class [`Cron`](https://webfiori.com/docs/webfiori/framework/cron/Cron)
* The class [`InitCron`](https://webfiori.com/docs/webfiori/framework/cron/InitCron)

There are other classes that makes the system for scheduling jobs works but they are mainly utility class. In order to schedule a job, mostly we have to work with the 4 given classes.

### The Class `AbstractJob`

This class has all needed methods for created a custom job. The developer can extend this class and implement the abstract methods to create the job. The class has 4 abstract methods which can be implemented:

* [`AbstractJob::execute()`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob#execute)
* [`AbstractJob::afterExec()`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob#afterExec)
* [`AbstractJob::onFail()`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob#onFail)
* [`AbstractJob::onSuccess()`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob#onSuccess)

### The Class `CronJob`

This class provides basic implementation for the class `AbstractJob`. The developer can create an instance of this class and schedule it directly.

### The Class `Cron`

This class is the controller for jobs scheduling system. This class performs many operations including jobs management, executing jobs, trigger job execution and so on.

### The Class `InitCron`

This class is one of the framework initialization classes. The class is used to initialize the scheduled jobs. The developer can use it to create an instance of his job and schedule it there. The class has one ststic method which is [`InitCron::init()`](https://webfiori.com/docs/webfiori/ini/InitCron#init). The developer can modify the body of the method as he needs.

## Scheduling Jobs

A job can be created using one of two ways, As a closure, or by extending the class [`AbstractJob`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob).

### Scheduling Job as a Closure

The easiest and simplest way is to schedule a job as a closure. To schedule a job as a closure, we have to use the class [`Cron`](https://webfiori.com/docs/webfiori/framework/cron/Cron). The class has many pre-made methods which can be used to schedule jobs as closures in diffrent tomes. These methods include the following ones:

* [`Cron::createJob()`](https://webfiori.com/docs/webfiori/framework/cron/Cron#createJob): Schedule a job using specific cron expression.
* [`Cron::dailyJob()`](https://webfiori.com/docs/webfiori/framework/cron/Cron#dailyJob): Schedule a job to run every day at specific time.
* [`Cron::monthlyJob()`](https://webfiori.com/docs/webfiori/framework/cron/Cron#monthlyJob): Schedule a job to run once every month in a specific day and time.
* [`Cron::weeklyJob()`](https://webfiori.com/docs/webfiori/framework/cron/Cron#weeklyJob): Schedule a job to run once every week in a specific day and time.

The following code sample shows how to use each one of the methods to schedule jobs.

``` php
namespace webfiori\ini;

use webfiori\framework\cron\Cron;

class InitCron {
    /**
     * A method that can be used to initialize cron jobs.
     * The developer can use this method to create cron jobs.
     * @since 1.0
     */
    public static function init() {

        Cron::createJob('*/10 * * * *', 'Every Minute', function () {
            echo \"Will execute every 10 minutes\";
        });
        
        Cron::dailyJob(\"13:00\", \"Test Job\", function () {
            echo \"Will execute every day at 1:00 PM\";
        });
        
        Cron::weeklyJob('sun-15:30', 'Another Job', function () {
            echo \"Will execute every sunday at 3:30 PM\";
        });
        
        Cron::monthlyJob(1, '00:00', 'First Day Of Month', function () {
            echo \"Will execute at first day of every month at 12:00 AM\";
        });
    }
}
```

### Using the Class `AbstractJob`

If the job that will be scheduled is simple, it is recomended to schedule it as closure. But if the job is complex, it is recomended to implement it as a class to make it easier when managing it. In order to implement a job as a class, the class [`AbstractJob`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob) must be extended.

As we have said before, this class has 4 abstract methods at which the developer can implement. The method [`AbstractJob::execute()`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob#execute) will contain the actual job code that will be executed when it is time to run the job. The method must return a boolean value or null. If the job successfully executed, it must return `true` or `null`. If the job fails, the method must return `false`. Note that if the method throws an exception or error while it executes, the job will be considered as failed.

The method [`AbstractJob::onFail()`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob#onFail) can have a code that will be executed if the job has failed to complete successfully. For example, it can have a code that will notify system admins that a background job has failed. A job will be considered as a failed job in two cases. If the method [`AbstractJob::execute()`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob#execute) return `false` or when it throws an exception.

The method [`AbstractJob::onSuccess()`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob#onSuccess) can have a code that will be executed if the job finished without any issues. A job is considered as a success job when the method [`AbstractJob::execute()`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob#execute) returns `true` or `null` (simply return nothing).

The method [`AbstractJob::afterExec()`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob#afterExec) can have a code which will be executed after the job finished to execute. The code will get executed in both cases, success or fail.

#### A Sample Job

The developer can place the class that represents the job in any directory as long as it is in autoload directories. But if the developer would like from the framework to auto-register the job, he can place it in the folder `app/jobs`. The following code shows a sample job that writes a text to a file.

``` php
namespace webfiori\framework\cron;

use webfiori\framework\cron\AbstractJob;
use webfiori\framework\File;

class WriteFileJob extends AbstractJob{
    public function __construct() {
        parent::__construct('Write File Job');
        //Execute job every 1 hour
        $this->everyHour();
    }
    public function afterExec() {
        // A code that will get executed after job compleition.
    }

    public function execute() {
        // This is the code that will be get executed.
        $file = new File('Background Job File.txt', ROOT_DIR);
        $file->setRawData('The job "'.$this->getJobName().'" was executed at '.date('H:i')."\n");
        $file->write();
    }

    public function onFail() {
        // A code that will get executed only when the job fails.
    }

    public function onSuccess() {
        // A code that will get executed only when the job successfully run.
    }

}
```

After implementing the job, it must be registered. If the class is placed in the folder `app/jobs`, then the registration process will be automatic. If the job class is somewhere else, then it must be registered manualy. The developer can use the method [`Cron::scheduleJob()`](https://webfiori.com/docs/webfiori/framework/cron/Cron#scheduleJob) to register jobs. The code which can be used to register a job can be placed in the class `InitCron`. The following code shows how it is done.

``` php
namespace webfiori\ini;

use webfiori\framework\cron\Cron;
use webfiori\framework\cron\WriteFileJob;

class InitCron {
    /**
     * A method that can be used to initialize cron jobs.
     * The developer can use this method to create cron jobs.
     * @since 1.0
     */
    public static function init() {

        Cron::scheduleJob(new WriteFileJob());
    }
}

```

## Jobs Execution
