# Installation
In this page:
* [Requirements](#requirements)
  * [General Requirements](#general-requirements)
* [Setup Local Development Environment In Windows](#setup-local-development-environment-in-windows)
* [Installing The Framework](#installing-the-framework)
    * [Create Project Using Composer](#create-project-using-composer)
    * [Manual Installation](#manual-installation)
* [Running The Project](#running-the-project)
    * [Using The Built-in Web Server](#using-the-built-in-web-server) 

## Requirements

The following set of requirements must be met in order for the framework to work accordingly.

### General Requirements

* PHP >= 7.0
* mbstring PHP Extension
* mysqli PHP Extension (to connect to MySQL)
* [sqlsrv PHP Extension](https://learn.microsoft.com/en-us/iis/application-frameworks/install-and-configure-php-on-iis/install-the-sql-server-driver-for-php) (to connect SQL Server)
* fileinfo PHP Extension
* json PHP Extension
* [Composer Dependency Manager](https://getcomposer.org/download/)

In addition to that, it would be great to have an IDE or a text editor that can help in writing code and debugging. We recommend to use [Apache Netbeans IDE](https://netbeans.apache.org/). 

## Setup Local Development Environment In Windows
For simple testing and development, it is possible to use the built-in server which comes with PHP. For production, it is always recommended to use server application such as Apache, Nginx or IIS to run PHP.

PHP binaries for windows can be found [here](https://windows.php.net/download). The binaries include only PHP interpurture without MySQL or Apache.

## Installing The Framework

There are 2 ways to install the framework:
* [Create Project Using Composer](#create-project-using-composer)
* [Manual Installation](#manual-installation)

### Create Project Using Composer

Assuming that composer is installed as executable, the framework can be installed by creating a composer project which is based on the framework inside the same folder. To do that, simply open command promote in your working directory and run the following command:
``` bash
composer create-project webfiori/app my-site --prefer-dist
```
This command will create new folder which has the name `my-site` and install WebFiori framework with all its dependencies. The name of the folder can be changed as needed.

> **Note:** It is possible to rename application source code folder by changing the value of the constant `APP_DIR` in the top of the file `index.php`. The default value for `APP_DIR` is `app`.

### Manual Installation

The other installation option is to download the framework from the official website and install it manually. Simply, go to [https://webfiori.com/download](https://webfiori.com/download) and download the latest release of the framework. Once downloaded, the files should be extracted to a sub-folder. Assuming that the name of the folder is `my-site`, your workspace will be the folder `\my-site\app`. Also, document root will be `\my-site\public`.

### Running The Project

There are two ways to start your website. The first one is to use PHP's built-in web server and the second one is to use actual web server application. The recommended way is to use web server application as this will make sure that your application is running in an environment which is similar to production.


#### Using The Built-in Web Server
For simplicity, this section shows how to run the project using the built-in PHP server. This approach must only be used for testing. To run your application using built-in server, open command prompt and change directory to your application root (e.g. `my-site`). After that, run the following command:
``` bash
php -S localhost:8080 -t public
```
This will start the server at `localhost` port `8080`. Now if we open the web browser and navigate to `localhost:8080`, the application will open.


**Previous: [Introduction](learn/introduction)**

**Next: [Folder Structure](learn/folder-structure)**


