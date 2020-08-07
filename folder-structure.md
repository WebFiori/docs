# Folder Structure

## The `app` Folder
The folder `app` will contain all your application files. Anything that created by you must exist inside this folder. The folder can be kept under source control because of this reason. This folder can have extra sub-folders for maintaining your code. The folders are:
* `app/apis`
* `app/commands`
* `app/jobs`
* `app/langs`
* `app/pages`
* `app/routes`

### The `app/apis` Folder
This folder can have all classes that represents web service set. A web service set is a class that extends the class <a href="https://webfiori.com/docs/restEasy/WebServices">`WebServices`</a> or the class <a href="https://webfiori.com/docs/webfiori/entity/ExtendedWebServices">`ExtendedWebServices`</a>. The routes to web services added using the method `Router::api()`.

### The `app/commands` Folder
This folder can have any custom CLI commands which are created by the developer. A CLI command can be created by extending the class <a href="https://webfiori.com/docs/webfiori/entity/cli/CLICommand">`CLICommand`</a> and implementing the abstract methods of the class. Any command inside this folder will be auto-registered by the framework.

### The `app/jobs` Folder
This folder can hold background jobs. Background jobs contains a code which is set to execute at specific type. They are implemented by extending the class <a href="https://webfiori.com/docs/webfiori/entity/cron/AbstractJob">`AbstractJob`</a>. Any class that represents a background job and is added to this folder, will be auto-registered.

### The `app/langs` Folder 
This folder contains files which adds support for i18n. The developer can create his own language classes or add labels to existing one. A language class must extend the class <a href="https://webfiori.com/docs/webfiori/entity/langs/Language">`Language`</a>.

### The `app/pages` Folder
This folder must contains the classes which represents the pages of your website or web application. The routes to classes which exist in this folder can be added using the method <a  href="https://webfiori.com/docs/webfiori/entity/router/Router#view">`Router::view()`</a>.

### The `app/routes` Folder
This folder contains classes which can be used to create different types of routes to any resources which exist in the system. The folder already has 4 classes. Each class has one static method at which the developer can modify its body to add new routes.

## The `conf` Folder
The folder `conf` contains 3 configuration classes which might hold sensitive information about your web application. They can have database credentials or SMTP connection information. Also, they hold main information about your website such as Its name, general description and main language.

## The `entity` Folder
This folder contains core entities of the framework. The developer should not modify this folder or add any code which is related to his web application. The folder should be kept outside source control.

## The `ini` Folder
This folder contains initialization classes. The developer can use the classes to initialize system privileges, custom jobs, custom CLI commands, custom autoload directories and global constants. The folder can be kept under source control as the developer might modify the code in the initialization classes.

## The `public` Folder
The public folder is the root directory of the web application. The developer should include any public resources inside this folder as it will be the entry point of any request. 

## The `logic` Folder
This folder contains basic controller classes which are used by the framework. The controllers are used to control basic settings of the framework in addition to managing database connections and sesstion management.

## The `themes` Folder
The folder `themes` is a place that contains components that gives a unified look and feel for the website. Each theme is contained in its sub-folder.
