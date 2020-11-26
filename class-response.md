
# The Class Response

<meta name="description" description="The class response is used to send back server output to the client.">

In this page:

* [Introduction](#introduction)
* [Collecting Server Output](#collecting-server-output)
* [Sending Output](#sending-the-output)
* [Sending Custom Headers](#sending-custom-headers)
* [Custom Response Code](#custom-response-code)
* [Clearing Output](#clearing-output)

## Introduction

Normally, sending output to HTTP client in php is performed using [`echo`](https://www.php.net/manual/en/function.echo.php) or [`print`](https://www.php.net/manual/en/function.print.php). The framework has its own way for sending output to HTTP clients which can be achived using the class [`Response`](https://webfiori.com/docs/webfiori/http/Response). The main aim of this class is to collect server output and send it back to the client after the server compeletly finished processing client request. For this reason, the developer must never use `echo`, `print` or `die` to show server output.  

## Collecting Server Output

Server output at the end must be a string. Collecting server output can be achived using the method [`Response::write()`](https://webfiori.com/docs/webfiori/http/Response#write). From its name, we can infer that it is used to write a string to collected server output.

> **Note:** In most cases, the developer will not have to collect the output him self. If the class [`Page`](https://webfiori.com/docs/webfiori/framework/Page) is used for rendering HTML, no need to use the method `Response::write()`. This also applies to web services if the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/http/WebServicesManager) or [`ExtendedWebServicesManager`](https://webfiori.com/docs/webfiori/framework/ExtendedWebServicesManager) is used.

``` php
namespace webfiori\framework\router;

use webfiori\framework\Util;
use webfiori\http\Response;

class ClosureRoutes {
    /**
     * Create all closure routes. Include your own here.
     * @since 1.0
     */
    public static function create() {
        Router::closure([
            'path' => '/closure',
            'route-to' => function() {
                Response::write('Hello,<br/>');
                Response::write('Welcome to my website!<br/>');
                Response::write('Current Time is: '.date('H:i:s'));
            }
        ]);
    }
}
```

## Sending The Output

It is possible to terminate code execution on server before the whole code is executed. This can be achived by manually invoking the static method [`Response::send()`](https://webfiori.com/docs/webfiori/http/Response#send). This can happen if the developer would like to debug his code and checking the value of specific variable.

``` php
namespace webfiori\framework\router;

use webfiori\framework\Util;
use webfiori\http\Response;

class ClosureRoutes {
    /**
     * Create all closure routes. Include your own here.
     * @since 1.0
     */
    public static function create() {
        Router::closure([
            'path' => '/closure',
            'route-to' => function() {
                $myVar = null;
                Util::print_r($myVar);
                $myVar = 'Hello';
                Util::print_r($myVar);
                
                // This will terminate code execution
                Response::send();
                
                // This will show the value of $myVal twice.
            }
        ]);
    }
}
```

## Sending Custom Headers

The developer can use the method [`Response::addHeader()`](https://webfiori.com/docs/webfiori/http/Response#addHeader) to add custom headers to the response. For example, we can send a JSON string and set its content type to `application/json`.

``` php
namespace webfiori\framework\router;

use webfiori\http\Response;

class ClosureRoutes {
    /**
     * Create all closure routes. Include your own here.
     * @since 1.0
     */
    public static function create() {
        Router::closure([
            'path' => '/closure',
            'route-to' => function() {
                $jsonData = '{"username":"WarriorVx", "email":"my-email@example.com", "age":33}';
                Response::write($jsonData);
                Response::addHeader('content-type', 'application/json');
            }
            
        ]);
    }
}
```

## Custom Response Code

By default, the class `Response` will send any output with code `200 - Ok`. But it is possible to send the response using different response code. The method the method [`Response::setCode()`](https://webfiori.com/docs/webfiori/http/Response#setCode) can be used to achive this.

``` php
namespace webfiori\framework\router;

use webfiori\http\Response;

class ClosureRoutes {
    /**
     * Create all closure routes. Include your own here.
     * @since 1.0
     */
    public static function create() {
        Router::closure([
            'path' => '/closure',
            'route-to' => function() {
                Response::write('Resource Was Not Found!');
                Response::setCode(404);
            }
            
        ]);
    }
}
```

## Clearing Output

For some reson or another, the developer may want to clear the collected server output and replace it with another. Also, he might want to clear the headers. Clearing all output can be performed using the method [`Response::clear()`](https://webfiori.com/docs/webfiori/http/Response#clear). To clear response body only, the method [`Response::clearBody()`](https://webfiori.com/docs/webfiori/http/Response#clearBody) can be used. To clear headers only, the method [`Response::clearHeaders()`](https://webfiori.com/docs/webfiori/http/Response#clearHeaders) can be used.

``` php
namespace webfiori\framework\router;

use webfiori\http\Response;

class ClosureRoutes {
    /**
     * Create all closure routes. Include your own here.
     * @since 1.0
     */
    public static function create() {
        Router::closure([
            'path' => '/closure',
            'route-to' => function() {
                Response::append('Resource Was Not Found!');
                Response::setCode(404);
                
                Response::clear();
                Response::write('Resource Was Found!');
                Response::setCode(200);
                //This will send back "Resource Was Found!"
            }
        ]);
    }
}
```

**Next: [UI Package](learn/ui-package)**

**Previous: [Routing](learn/routing)**
