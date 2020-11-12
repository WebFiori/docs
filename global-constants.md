
# Global Constants

<meta name="description" content="A reference of all constants which are global in WebFiori Framework.">

## Introduction

The framework comes with a class which is only used to initialize global constants. They can be accessed anywhere within the scope of the framework. The class [GlobalConstants](https://webfiori.com/docs/webfiori/ini/GlobalConstants) is used to initialize the values of the constants. The developer can modify the values of the constants in the class as needed to configure some of the settings of the framework. Also, the developer can modify the content of the class and create his own constants.

## Constants

The following table lists all constants and for what each one is used.

|Constant Name|Description  |Default Value|
|:-----------|:-----------|:-----------|
|`SCRIPT_MEMORY_LIMIT`|Memory limit per script. This constant represents the maximum amount of memory each script will consume before showing a fatal error. The developer can change this value as needed.|'2GB' (`string`)|
|`LOAD_COMPOSER_PACKAGES`|This constant is used to tell the core if the application uses composer packages or not. If set to true, then composer packages will be loaded.|true (`boolean`)|
|`CRON_THROUGH_HTTP`|A constant which is used to enable or disable HTTP access to cron. If the constant value is set to true, the framework will add routes to the components which is used to allow access to cron control panel. The control panel is used to execute jobs and check execution status.|false `boolean`|
|`NO_WWW`|This constant is used to redirect a URI with www to `non-www`. If this constant is defined and is set to true and a user tried to access a resource using a URI that contains `www` in the host part, the router will send a `301 - permanent redirect` HTTP response code and send the user to `non-www` host. For example, if a request is sent to `https://www.example.com/my-page`, it will be redirected to `https://example.com/my-page`. |false (`boolean`)|
|`DATE_TIMEZONE`|Define the timezone at which the system will operate in. The value of this constant is passed to the function `date_default_timezone_set()`. This one is used to fix some date and time related issues when the application is deployed in multiple servers. See http://php.net/manual/en/timezones.php for supported time zones. Change this as needed.|'Asia/Riyadh' (`string`)|
|`PHP_INT_MIN`|Fallback for older php versions that does not support the constant `PHP_INT_MIN`.|(`double`)|
|`WF_VERBOSE`|This constant is used to tell the framework if more information should be displayed if an exception is thrown or an error happens. The main aim of this constant is to hide some sensitive information from users if the system is in production environment. Note that the constant will have effect only if the framework is accessed through HTTP protocol. If used in CLI environment, everything will appear.|false (`boolean`)|
|`THEMES_PATH`|This constant represents the directory at which themes exist.|'themes' (`string`)|
|`MAX_BOX_MESSAGES`|The maximum number of message boxes to show in one page. A message box is a box which will be shown in a web page that contains some information. The box can be created manually by using the method [`Util::print_r()`](https://webfiori.com/docs/webfiori/framework/Util#pring_r)' or it can be as a result of an error during execution. Default value is 15. The developer can change the value as needed. Note that if the constant is not defined, the number of boxes will be **almost unlimited**.|15 (`int`)|
|`CLI_HTTP_HOST`|Host name to use in case the system is executed through CLI. When the application is running throgh CLI, there is no actual host name. For this reason, the host is set to `127.0.0.1` by default. If this constant is defined, the host will be changed to the value of the constant. Default value of the constant is 'example.com'.|'example.com' (`string`)|
|`DS`|Directory separator. This one is is used as a shorthand instead of using PHP constant `DIRECTORY_SEPARATOR`. The two will have the same value.|(`string`)|
|`USE_HTTP`|Sets the framework to use `http://` or `https://` for base URIs. The default behaviour of the framework is to use `https://`. But in some cases, there is a need for using `http://`. If this constant is set to true, the framework will use `http://` for base URI of the system. Default value is false.|false (`boolean`)|
