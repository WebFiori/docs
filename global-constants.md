
# Global Constants

<meta name="description" content="A reference of all constants which are global in WebFiori Framework.">

## Introduction

The framework comes with a class which is only used to initialize global constants. They can be accessed anywhere within the scope of the framework. The class [GlobalConstants](https://webfiori.com/webfiori/ini/GlobalConstants) is used to initialize the values of the constants. The developer can modify the values of the constants in the class as needed to configure some of the settings of the framework. Also, the developer can modify the content of the class and create his own constants.

## Constants

The following table lists all constants and for what each one is used.
|Constant Name|Description  |Default Value|
|:-----------|:-----------|:-----------|
|`SCRIPT_MEMORY_LIMIT`|Memory limit per script. This constant represents the maximum amount of memory each script will consume before showing a fatal error. The developer can change this value as needed.|'2GB' (`string`)|
|`LOAD_COMPOSER_PACKAGES`|This constant is used to tell the core if the application uses composer packages or not. If set to true, then composer packages will be loaded.|true (`boolean`)|
|`CRON_THROUGH_HTTP`|A constant which is used to enable or disable HTTP access to cron. If the constant value is set to true, the framework will add routes to the components which is used to allow access to cron control panel. The control panel is used to execute jobs and check execution status.|false `boolean`|
|`NO_WWW`|This constant is used to redirect a URI with www to non-www. If this constant is defined and is set to true and a user tried to access a resource using a URI that contains www in the host part, the router will send a 301 - permanent redirect HTTP response code and send the user to non-www host. For example, if a request is sent to 'https://www.example.com/my-page', it will be redirected to 'https://example.com/my-page'. |false (`boolean`)|
