# Background Tasks

<meta name="description" content="Background tasks are used to run even if there is no one is using the application. Learn about how to implement and use them here.">

In this page:

* [Introduction](#introduction)
* [Main Classes](#main-classes)
  * [The Class `AbstractTask`](#the-class-abstracttask)
  * [The Class `BaseTask`](#the-class-BaseTask)
  * [The Class `TasksManager`](#the-class-tasksmanager)
  * [The Class `InitTasks`](#the-class-inittasks)
* [Scheduling Jobs](#scheduling-jobs)
  * [Scheduling Job as a Closure](#scheduling-job-as-a-closure)
  * [Using the Class `AbstractTask`](#using-the-class-abstracttask)
* [Jobs Execution](#jobs-execution)
  * [Using `crontab` Entry to Trigger Execution](#using-crontab-entry-to-trigger-execution)
  * [Using Command Line Interface to Trigger Execution](#using-command-line-interface-to-trigger-execution)
  * [Forcing a Job to Execute](#forcing-a-job-to-execute)
    * [Force a Job to Execute Using Cron Web Interface](#force-a-job-to-execute-using-cron-web-interface)
    * [Force a Job to Execute Using Command Line Interface](#force-a-job-to-execute-using-command-line-interface)
* [Using Arguments With Forced Jobs](#using-aguments-with-forced-jobs)
  * [Sending Arguments Through CRON Web Interface](#sending-arguments-through-cron-web-interface)
  * [Sending Arguments Through Terminal](#sending-arguments-through-terminal)
* [Sending CRON Notifications](#sending-cron-notifications)
  * [The Class `CronEmail`](#the-class-cronemail)

## Introduction

One of the features of the framework is the ability to schedule PHP code to run at specific time in background. For example, it is possible to make your web application send reports at specific time using email. The framework give the developers all needed tools to schedule background tasks in a simple way.

## Main Classes

The sub-system which is responsible for scheduling jobs consist of 3 classes. The classes are:

* [`AbstractTask`](https://webfiori.com/docs/webfiori/framework/cron/AbstractTask)
* [`BaseTask`](https://webfiori.com/docs/webfiori/framework/scheduler/BaseTask)
* [`Cron`](https://webfiori.com/docs/webfiori/framework/scheduler/Cron)

There are other classes that makes the system for scheduling jobs works, but they are mainly utility class.

### The Class `AbstractTask`

Developer can extend this class and implement the abstract methods to create the job. The class has 4 abstract methods which must be implemented:

* [`AbstractTask::execute()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#execute)
* [`AbstractTask::afterExec()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#afterExec)
* [`AbstractTask::onFail()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#onFail)
* [`AbstractTask::onSuccess()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#onSuccess)

### The Class `BaseTask`

This class provides basic implementation for the class `AbstractTask`. The developer can create an instance of this class and schedule it directly.

### The Class `Cron`

This class is the controller for jobs scheduling system. This class performs many operations including jobs management, executing jobs, trigger job execution and so on.

## Scheduling Jobs

A job can be implemented using one of two ways, As a closure, or by extending the class [`AbstractTask`](https://webfiori.com/docs/webfiori/framework/cron/AbstractTask).

### Scheduling Job as a Closure

This way is straightforward and does not require lots of coding. To schedule a job as a closure, the class [`Cron`](https://webfiori.com/docs/webfiori/framework/cron/scheduler) is used directly. The class has many pre-made methods which can be used to schedule jobs as closures in diffrent times. These methods include the following ones:

* [`Cron::createJob()`](https://webfiori.com/docs/webfiori/framework/scheduler/Cron#createJob): Schedule a job using specific cron expression.
* [`Cron::dailyJob()`](https://webfiori.com/docs/webfiori/framework/scheduler/Cron#dailyJob): Schedule a job to run every day at specific time.
* [`Cron::monthlyJob()`](https://webfiori.com/docs/webfiori/framework/scheduler/Cron#monthlyJob): Schedule a job to run once every month in a specific day and time.
* [`Cron::weeklyJob()`](https://webfiori.com/docs/webfiori/framework/scheduler/Cron#weeklyJob): Schedule a job to run once every week in a specific day and time.

The following code sample shows how to use each one of the methods to schedule jobs. Initializing jobs must be performed in the class `[APP_DIR]\ini\InitCron`

``` php
namespace app\ini;

use webfiori\framework\scheduler\Cron;

class InitCron {
    /**
     * A method that can be used to initialize cron jobs.
     * The developer can use this method to create cron jobs.
     * @since 1.0
     */
    public static function init() {

        Cron::createJob('*/10 * * * *', 'Every Minute', function () {
            echo "Will execute every 10 minutes";
        });
        
        Cron::dailyJob("13:00", "Test Job", function () {
            echo "Will execute every day at 1:00 PM";
        });
        
        Cron::weeklyJob('sun-15:30', 'Another Job', function () {
            echo "Will execute every sunday at 3:30 PM";
        });
        
        Cron::monthlyJob(1, '00:00', 'First Day Of Month', function () {
            echo "Will execute at first day of every month at 12:00 AM";
        });
    }
}
```

### Using the Class `AbstractTask`

This method of implementing jobs should be used for complex jobs. In this approach, the class [`AbstractTask`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask) is extended and its abstract methods are implemented.

This class has 4 abstract methods at which the developer must implement. The method [`AbstractTask::execute()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#execute) will contain the actual job logic that will be executed. The method must return a boolean value or null. If the job successfully executed, it must return `true` or `null`. If the job fails, the method must return `false`. Note that if the method throws an exception or error while it executes, the job will be considered as failed.

The method [`AbstractTask::onFail()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#onFail) can have a code that will be executed if the job has failed to complete successfully. For example, it can have a code that will notify system admins that a background job has failed. A job will be considered as a failed job in two cases. If the method [`AbstractTask::execute()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#execute) return `false` or when it throws an exception.

The method [`AbstractTask::onSuccess()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#onSuccess) can have a code that will be executed if the job finished without any issues. A job is considered as a success job when the method [`AbstractTask::execute()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#execute) returns `true` or `null` (simply return nothing).

The method [`AbstractTask::afterExec()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#afterExec) can have a code which will be executed after the job finished to execute. The code will get executed in both cases, success or fail.

#### A Sample Job

It is recommended to place jobs at the folder `[APP_DIR]/jobs` as they will be auto-registered if they are placed in this folder. The following code shows a sample job that writes a text to a file.

``` php
namespace app\jobs;

use webfiori\framework\scheduler\AbstractTask;
use webfiori\framework\File;

class WriteFileJob extends AbstractTask{
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

After implementing the job, it must be registered. If the class is placed in the folder `[APP_DIR]/jobs`, then the registration process will be automatic. If the job class is somewhere else, then it must be registered manually. The developer can use the method [`Cron::scheduleJob()`](https://webfiori.com/docs/webfiori/framework/scheduler/Cron#scheduleJob) to register jobs. The code which can be used to register a job can be placed in the class `[APP_DIR]\InitCron`. The following code shows how it is done.

``` php
namespace app\ini;

use webfiori\framework\scheduler\Cron;
use app\jobs\WriteFileJob;

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

When a job is scheduled, it will not get executed by itself even if it is time to run it. A job can be only executed if there was a trigger that caused it to execute. The trigger can be a forced trigger or automatic trigger.

### Using `crontab` Entry to Trigger Execution

The recommended way is to add a CRON entry on your server which looks like this one: `* * * * * php webfiori cron p="pass" --check`. This will execute the command `cron` of the framework with the option `--check` every minute. The option `pass` must be included if a password is set to protect jobs from unauthorized execution. It will simply check all scheduled jobs and to check if it is time to execute them. 

If the server is running on windows, it is possible to use "Task Scheduler" to achieve the same result.

>> Note: The path to PHP interpreter and the framework my differ. Because of this, the format of the entry may differ.


### Using Command Line Interface to Trigger Execution

Another way to trigger jobs execution is to use command line interface (or terminal) to run the command `cron`. If the server supports SSH access, it is possible to run the following command to trigger execution:

``` 
php webfiori cron p="pass" --check
```

Executing jobs through terminal can be useful if the developer would like to inspect the output of jobs or would like to check what causes a job to fail.

### Forcing a Job to Execute

One of the features of the framework is the ability to force a scheduled job to execute even if it is not the time to run it. A job can be forced to run using one of two methods:

* Cron web interface.
* Command line interface.

#### Force a Job to Execute Using Cron Web Interface

If the environment variable `CRON_THROUGH_HTTP` is defined and is set to `true`, it will be possible to access cron web interface and use it to force a job to execute. The control panel can be accessed using any web browser by navigating to the URL `https://yoursite.com/cron`. If a password is set to protect the jobs, it will open a login page to enter protection password.

<img src="assets/images/cron01.png" alt="Cron web interface.">

#### Force a Job to Execute Using Command Line Interface

Forcing a job to execute through terminal is useful in case of debugging. The terminal can be used to show the full output of executing a job. To force execution of a specific job, simply we have to run the following command:

```
php webfiori cron p="pass" --force --show-log
```

Once this command is executed, the terminal will ask the user to select one of the scheduled jobs to force. The following image shows the full terminal output when using this way to force a job.

<img src="assets/images/cron02.png" alt="Cron command line interface">

## Using Arguments With Forced Jobs

One of the things that a developer might want from a job to do is when it is forced to execute is to behave in a different way based on some inputs given by the one who will execute it. One of the features that the framework provides is the ability to add custom execution arguments to a job. The arguments can be sent to the job when it is forced to execute through the control panel of the jobs or through command line interface.

### Adding Arguments

Job arguments are associated with an instance of the class `AbstractTask`. In order to add arguments to a job, simply use the method [`AbstractTask::addExecutionArg()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#addExecutionArg). Also, it is possible to add multiple arguments at once using the method [`AbstractTask::addExecutionArgs()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#addExecutionArgs). Usually, arguments are added to the job when initialized (in the constructor). But it is possible to add them after the job has been scheduled.

The following code sample shows how to add arguments to job.
``` php
namespace app\jobs;

use webfiori\framework\scheduler\AbstractTask;
use webfiori\frameworl\scheduler\Cron;

class GenerateAttendanceReportJob extends AbstractTask {
    public function __construct() {
        parent::__construct('');
        
        //Generate attendance report once the new month start.
        $this->everyMonthOn(1, '00:00');
        
        // Add two arguments
        $this->addExecutionArgs([
            'start-date' => [
                'default' => date('Y-m-d', strtotime(date('Y-m-d').' -30 days'))
            ],
            'end-date' => [
                'default' => date('Y-m-d')
            ],
        ]);
    }
    public function afterExec() {
        //TODO:
    }

    public function execute() {
        
        $startDate = $this->getArgValue('start-date');
        $endDate = $this->getArgValue('end-date');
        
        Cron::log('Start date = "'.\$startDate.'"');
        Cron::log('End date = "'.\$endDate.'"');
        //TODO: Generate the report.
    }

    public function onFail() {
        //TODO: Notify using email that an error stopped the job.
    }

    public function onSuccess() {
        //TODO: Store generated reports and send them using email.
    }

}
```

### Sending Arguments Through CRON Web Interface

If the ability to execute cron jobs through HTTP is enabled, it will be possible to access cron control panel to force execution of a job. The access to the control panel can be enabled by defining the constant <code>CRON_THROUGH_HTTP</code> and setting its value to <code>true</code>. Assuming that the server has the URL "https://example.com", the cron control panel can be accessed through "https://example.com/cron".

To supply arguments to a job when forcing it to execute, simply navigate to the page that shows job information by clicking on its name in the page that lists all scheduled jobs. In the lower side of the page, there will be a table at which the user can see the names of the arguments. The next image shows the whole page that shows the details of the job.

<img src="assets/images/cron03.png" alt="Cron Job arguments">

### Sending Arguments Through Terminal

Another way to force jobs to execute is to use command line interface. The command `cron` is used to force the execution of a job. To force a job, simply supply the argument `--force` 

The following terminal output image shows how to force the job that was created using the code at the start of this page. Notice that if the job has extra arguments, it asks to supply them.

<img src="assets/images/cron04.png" alt="Cron job arguments">

## Sending CRON Notifications

One of the things that the framework supports is the ability to send email notifications about jobs execution status. The notifications are useful and can be used to follow up with jobs execution and can be used to diagnose the cause of failure of a job when it fails.

> **Note**: Note that to use mailing service of the framework, SMTP account must be setup. To learn more about sending emails and how to do the setup, [click here](learn/sending-emails).

### The Class `CronEmail`

In order to send email notifications, the class [`CronEmail`](https://webfiori.com/docs/webfiori/framework/scheduler/CronEmail) must be used. This class can be used to send an email regarding the status of background job execution (failed or success). This class must be only used in one of the abstract methods of a background job since using it while no job is active will have no effect. It is recommended to create an instance of this class in the body of the method [`AbstractTask::afterExec()`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask#afterExec).

The following code sample shows how to use the class to send notifications to one email address. It assumes that there exist SMTP account which has the name `cron-notifications` is added to the class `[APP_DIR]\config\AppConfig` and it will be used to send the notifications.

``` php
namespace app/jobs;

use webfiori\framework\scheduler\AbstractTask;
use webfiori\framework\scheduler\Cron;
use webfiori\framework\scheduler\CronEmail;

class GenerateAttendaceReportJob extends AbstractTask {
    public function __construct() {
        parent::__construct('Generate Attendance Report');
        
        //Generate attendance report once the new month start.
        $this->everyMonthOn(1, '00:00');
        
        // Add two arguments
        $this->addExecutionArgs([
            'start-date',
            'end-date',
        ]);
    }
    public function afterExec() {
        //Send email that shows the status of job execution
        new CronEmail('cron-notifications', [
            'ibrahim@example.com' => 'Ibrahim' 
        ]);
    }

    public function execute() {
        //Access argument value.
        $startDate = \$this->getArgValue('start-date');
        if (\$startDate === null) {
            Cron::log('Start date is missing. Using default value.');
            //TODO: Set a default start date which should be the start of prev month
            $startDate = '2020-06-01';
        }
        
        $endDate = \$this->getArgValue('end-date');
        if ($endDate === null) {
            Cron::log('End date is missing. Using default value.');
            //TODO: Set a default start date which should be the start of prev month
            $endDate = '2020-06-30';
        }
        Cron::log('Start date = \"'.\$startDate.'\"');
        Cron::log('End date = \"'.\$endDate.'\"');
        //TODO: Generate the report.
    }

    public function onFail() {
        //TODO: Notify using email that an error stopped the job.
    }

    public function onSuccess() {
        //TODO: Store generated reports and send them using email.
    }

}
```

**Next: [Internationalization](learn/i18n)**

**Previous: [Middleware](learn/middleware)**
