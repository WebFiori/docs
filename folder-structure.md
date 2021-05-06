# Folder Structure
<meta name="description" content="Learn about the folders at which the framework uses to keep your code and the content of each folder.">

In this page:
* [The `app` Folder](#the-app-folder)
  * [The `app/apis` Folder](#the-appapis-folder)
  * [The `app/commands` Folder](#the-appcommands-folder)
  * [The `app/jobs` Folder](#the-appjobs-folder)
  * [The `app/langs` Folder ](#the-applangs-folder)
  * [The `app/pages` Folder](#the-apppages-folder)
  * [The `app/ini` Folder](#the-appini-folder)
  * [The `app/ini/routes` Folder](#the-appiniroutes-folder)
  * [The `app/middleware` Folder](#the-appmiddleware-folder)
* [The `vendor` Folder](#the-vendor-folder)
* [The `public` Folder](#the-public-folder)
* [The `themes` Folder](#the-themes-folder)

## The `app` Folder

The folder `app` will contain all your application files. Anything that created by you must exist inside this folder. The folder can be kept under source control because of this reason. This folder can have extra sub-folders for maintaining your code. The folders are:
* `app/apis`: Used to hold web services and services managers.
* `app/commands`: Used to hold custom CLI commands.
* `app/jobs`: Used to hold cron jobs.
* `app/langs`: Used to hold translation files (for i18n support)
* `app/pages`: Used to hold views and web pages.
* `app/ini`: Used to hold classes which used to initialize parts of the system.
* `app/routes`: A folder that contains classes for creating routes.
* `app/database`: Holds classes which are related to database.
* `app/middleware`: Holds classes which represents a middleware.
* `app/entity` Holds classes which represents system entities.

In addition to the given folders, the directory will have a file called `AppConfig.php`. This file contains a code for a class that holds configuration information for your application. If the file does not exist or was deleted, the framework will re-create it.

Also, you may create your own folders and add files as you like. More details about the folders can be found below.

### The `app/apis` Folder

This folder can have all classes that are related to web services. This folder will mostly holde web services controllers and web services classes. A web service controller is a class that extends the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/http/WebServicesManager) or the class [`ExtendedWebServices`](https://webfiori.com/docs/webfiori/framework/ExtendedWebServicesManager). A web service is a class that extends the class [`AbstractWebService`](https://webfiori.com/docs/webfiori/http/AbstractWebService).

### The `app/commands` Folder

This folder can have any custom CLI commands which are created by you. A CLI command can be created by extending the class [`CLICommand`](https://webfiori.com/docs/webfiori/framework/cli/CLICommand) and implementing the abstract methods of the class. Any command inside this folder will be auto-registered by the framework.

### The `app/jobs` Folder

This folder can hold background jobs. Background jobs contains a code which is set to execute at specific type. They are implemented by extending the class [`AbstractJob`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob). Any class that represents a background job and is added to this folder, will be auto-registered.

### The `app/langs` Folder 

This folder contains files which adds support for i18n. You can create your own language classes or add labels to existing one. A language class must extend the class [`Language`](https://webfiori.com/docs/webfiori/framework/i18n/Language).

### The `app/pages` Folder

This folder must contains the classes which represents the pages of your application. Also, it can contain HTML template files which represent views. The routes to classes which exist in this folder can be added using the method <a  href="https://webfiori.com/docs/webfiori/framework/router/Router#view">`Router::view()`</a>.

## The `app/ini` Folder

This folder contains initialization classes. You can use the classes to initialize system privileges, custom jobs, custom CLI commands, custom autoload directories, middleware and global constants. The folder can be kept under source control as you might modify the code in the initialization classes.


### The `app/ini/routes` Folder

This folder contains classes which are used to create different types of routes to any resources which exist in the application. The folder already has 4 classes. Each class has one static method at which you can modify its body to add new routes.

## The `app/middleware` Folder

This folder can have the classes which provides implementation for the class [`AbstractMiddleware`](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware). The classes in this folder represents every middleware you will create.

## The `vendor` Folder

This folder contains core entities of the framework in addition to any libraries you install using composer. You should not modify this folder or add any code which is related to his web application. The folder should be kept outside source control.

## The `public` Folder

The public folder is the root directory of your web application. You should include any public resources inside this folder as it will be the entry point of any request. The folder must have the `assets` folder inside it. The `assets` folder will usesally contain all resource files. Also, the folder will contain themes assets.

## The `themes` Folder
The folder `themes` is a place that contains components that gives a unified look and feel for the website. Each theme is contained in its sub-folder. CSS files and JavaScript files of each theme must exist inside the folder `public/assets`.

**Previous: [Installation](learn/installation)**

**Next: [Basic Usage](learn/basic-usage)**
