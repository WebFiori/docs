
# Global Constants

<meta name="description" content="A reference of all constants which are global in WebFiori Framework.">

## Introduction

The framework comes with a class which is only used to initialize global constants. They can be accessed anywhere within the scope of the framework. The class [GlobalConstants](https://webfiori.com/webfiori/ini/GlobalConstants) is used to initialize the values of the constants. The developer can modify the values of the constants in the class as needed to configure some of the settings of the framework. Also, the developer can modify the content of the class and create his own constants.

## Constants

The following table lists all constants and for what each one is used.
|Constant Name|Description  |Default Value|
|:-----------|:-----------|:-----------|
|`SCRIPT_MEMORY_LIMIT`|Memory limit per script. This constant represents the maximum amount of memory each script will consume before showing a fatal error. The developer can change this value as needed.|'2GB' (`string`)|
