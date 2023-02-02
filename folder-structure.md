# Folder Structure
<meta name="description" content="Learn about the folders at which the framework uses to keep your code and the content of each folder.">

In this page:
* [The `app` Folder](#the-app-folder)
  * [The `app/config` Folder](#the-appconfig-folder)
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
* [Setting Custom Application Folder](#setting-custom-application-folder)

## The `app` Folder

The folder `app` will contain all your application files. Any PHP source code files must exist inside this folder. This folder can have extra sub-folders for maintaining source code of the application. The folders are:

* `app/apis`: Holds web services and services managers (Web APIs).
* `app/commands`: Holds custom-made CLI commands.
* `app/config`: Hold application configuration classes.
* `app/database`: Holds classes which are related to database and database access.
* `app/entity` Holds classes which represents system entities.
* `app/ini`: Holds classes which are used to initialize the application.
* `app/ini/routes`: Holds classes for initializing routes.
* `app/jobs`: Holds cron jobs.
* `app/langs`: Holds translation files (for i18n support)
* `app/middleware`: Holds classes which represents a middleware.
* `app/pages`: Holds anything related to user interface such as web pages, views or UI components.


In addition to given folders, the directory will have a file called `AppConfig.php`. This file contains a code for a class that holds configuration information for your application. If the file does not exist or was deleted, the framework will re-create it.

Also, you may create your own folders and add files as you like. More details about the folders can be found below.

> <b>Note:</b> It is possible to have different name for the folder `app` by changing the value of the constant `APP_DIR`. For more information, [check here](##setting-custom-application-folder).

### The `app/config` Folder
This folder will hold two classes which are used to configure the application. The first class is `AppConfig` and the second one is `Env`. The class `AppConfig` is used to store application related configurations such as its name, default language, database connections and SMTP connections. The class `Env` is used to initialize environment variables only.

### The `app/apis` Folder

This folder can have all classes that are related to web services. This folder will mostly holde web services controllers and web services classes. A web service controller is a class that extends the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/http/WebServicesManager) or the class [`ExtendedWebServices`](https://webfiori.com/docs/webfiori/framework/ExtendedWebServicesManager). A web service is a class that extends the class [`AbstractWebService`](https://webfiori.com/docs/webfiori/http/AbstractWebService).

### The `app/commands` Folder

This folder can have any custom CLI commands which are created by you. A CLI command can be created by extending the class [`CLICommand`](https://webfiori.com/docs/webfiori/cli/CLICommand) and implementing the abstract methods of the class. Any command inside this folder will be auto-registered by the framework.

### The `app/jobs` Folder

This folder can hold background jobs. Background jobs contains a code which is set to execute at specific type. They are implemented by extending the class [`AbstractJob`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob). Any class that represents a background job and is added to this folder, will be auto-registered.

### The `app/langs` Folder 

This folder contains files which adds support for internationalization (i18n). You can create your own language classes or add labels to existing one. A language class must extend the class [`Language`](https://webfiori.com/docs/webfiori/framework/Language).

### The `app/pages` Folder

This folder must contains the classes which represents the pages of the application or the components that a page consist of. Also, it can contain HTML template files which represent views. The routes to classes which exist in this folder can be added using the method <a  href="https://webfiori.com/docs/webfiori/framework/router/Router#page">`Router::page()`</a>.

## The `app/ini` Folder

This folder contains initialization classes. Classes in this folder are used to initialize system privileges, custom jobs, custom CLI commands, custom autoload directories, middleware and global constants.


### The `app/ini/routes` Folder

This folder contains classes which are used to create different types of routes to any resources exist in the application. The folder already has 4 classes. Each class has one static method. The code in each static method should be used to initialize routes.

## The `app/middleware` Folder

This folder can have the classes which provides implementation for the class [`AbstractMiddleware`](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware). The classes in this folder represents every middleware that will be created. A middleware is a class which is used to provide a layer in top of the application for filtering requests before actual routing happens.

## The `vendor` Folder

This folder contains core entities of the framework in addition to any libraries that are installed using composer.

## The `public` Folder

The public folder is the root directory of the application. Any public resources should be includedinside this folder as it will be the entry point of any request. Public resources include JavaScript files, CSS or any images that the application may use. The folder must have the `assets` folder inside it. The `assets` folder will usesally contain all resource files. Also the `assets` folder will contain themes assets.

## The `themes` Folder
The folder `themes` holds themes which are bundeled with the framework. Each theme is contained in its sub-folder. CSS files and JavaScript files of each theme must exist inside the folder `public/assets`. The folde also can hold custom made themes.

> <b>Note:</b> It is possible to have custom-made themes in another place other than the folder `themes`. In this case, custom made themes should use CDN to include thier reasource files.

## Setting Custom Application Folder

The developer have the option to set custom name for the folder that holds application code. This allows the developer to follow PSR-4 without having to add application configuration classes in a folder and application related classes in another folder. To set custom name for application folder, the developer have to define the name of application folder at the top of the file `index.php`. 

When the source code of the file is observed, the `define` can be seen at [the top](https://github.com/WebFiori/app/blob/main/public/index.php#L10) of the file. The developer can specify new name for application folder by specifying a value for the constant `APP_DIR`. As noticed, default value is `app`. When the value of the constant is changed to something else, the framework will attempt to create the same structure as default application structure with all configuration classes.

Note that the process of renaming the folder will change the namespace at which initialization classes belongs to. For example, the class `AppConfig` belongs to the namespace `app\config`. If the value of the constant `APP_DIR` is changed to `coolApp`, the class will be re-created by the framework at the namespace `coolApp\config`.

**Previous: [Installation](learn/installation)**

**Next: [Basic Usage](learn/basic-usage)**
