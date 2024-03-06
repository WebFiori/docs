
# The Class Response

<meta name="description" content="The class response is used to send back server output to the client.">

In this page:

* [Introduction](#introduction)
* [Collecting Server Output](#collecting-server-output)
* [Sending Output](#sending-the-output)
* [Sending Custom Headers](#sending-custom-headers)
* [Working With HTTP Cookies](#working-with-http-cookies)
* [Custom Response Code](#custom-response-code)
* [Clearing Output](#clearing-output)

## Introduction

While PHP offers `echo` and `print` for sending output, WebFiori's `webfiori\http\Response` class provides a centralized approach. It gathers server output for streamlined delivery to clients after request processing. Additionally, it allows for sending custom HTTP headers. This empowers developers to construct well-structured and efficient responses within their WebFiori applications. 

> **Note**: This class is part of the library [`webfiori\http`](https://github.com/WebFiori/http).

## Collecting Server Output

Collecting server output can be archived using the static method [`Response::write()`](https://webfiori.com/docs/webfiori/http/Response#write). From its name, we can infer that it is used to write a string to response body.

> **Note:** In most cases, the developer will not have to collect the output him self. If the class [`WebPage`](https://webfiori.com/docs/webfiori/framework/ui/WebPage) is used for rendering HTML, no need to use it. This also applies to web services if the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/http/WebServicesManager) or [`ExtendedWebServicesManager`](https://webfiori.com/docs/webfiori/framework/ExtendedWebServicesManager) is used to manage your web APIs.

``` php
namespace app\ini\routes;

use webfiori\http\Response;
use webfiori\framework\router\Router;
use webfiori\framework\router\RouteOption;

class ClosureRoutes {
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/closure',
            RouteOption::TO => function() {
                Response::write('Hello,<br/>');
                Response::write('Welcome to my website!<br/>');
                Response::write('Current Time is: '.date('H:i:s'));
            }
        ]);
    }
}
```

## Sending The Output

It is possible to terminate code execution on server before the whole code is executed. This can be archived by manually invoking the static method [`Response::send()`](https://webfiori.com/docs/webfiori/http/Response#send). This can happen if the developer would like to debug his code and checking the value of specific variable.

``` php
namespace app\ini\routes;

use webfiori\http\Response;
use webfiori\framework\router\Router;
use webfiori\framework\router\RouteOption;

class ClosureRoutes {
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/closure',
            RouteOption::TO => function() {
                $myVar = 'Good;
                Response::write($myVar);
                $myVar = 'Hello';
                Response::write($myVar);
                
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
namespace app\ini\routes;

use webfiori\http\Response;
use webfiori\framework\router\Router;
use webfiori\framework\router\RouteOption;

class ClosureRoutes {
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/closure',
            RouteOption::TO => function() {
                $jsonData = '{"username":"WarriorVx", "email":"my-email@example.com", "age":33}';
                Response::write($jsonData);
                Response::addHeader('content-type', 'application/json');
            }
        ]);
    }
}
```

## Working With HTTP Cookies

In simple terms, [HTTP Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) are small piece of text which is set by the server for session management, personalizing content or tracking. Cookies are set by sending HTTP header [`set-cookie`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie). At minimum, a cookie must have a name and value.

Cookies have multiple attributes that must be set on order to have it valid like duration, path and others. Cookies in WebFiori are represented by the class [`HttpCookie`](https://webfiori.com/docs/webfiori/http/HttpCookie)

In order to send a new cookie, an object of type [`HttpCookie`](https://webfiori.com/docs/webfiori/http/HttpCookie) is created and then added to the response using the method [`Response::addCookie(HttpCookie)`](https://webfiori.com/docs/webfiori/http/Response#addCookie) as follows:

``` php
namespace app\ini\routes;

use webfiori\http\Response;
use webfiori\http\HttpCookie;
use webfiori\framework\router\Router;
use webfiori\framework\router\RouteOption;

class ClosureRoutes {
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/closure',
            RouteOption::TO => function() {
                $cookie = new HttpCookie();
                $cookie->setName('hello-cookie');
                $cookie->setValue('hello-world');
                Response::addCookie($cookie);
            }
        ]);
    }
}
```

## Custom Response Code

By default, the class `Response` will send any output with code `200 - Ok`. But it is possible to send the response using different response code. The method [`Response::setCode()`](https://webfiori.com/docs/webfiori/http/Response#setCode) can be used to achieve this.

``` php
namespace app\ini\routes;

use webfiori\http\Response;
use webfiori\framework\router\Router;
use webfiori\framework\router\RouteOption;

class ClosureRoutes {
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/closure',
            RouteOption::TO => function() {
                Response::write('Resource Was Not Found!');
                Response::setCode(404);
            }
        ]);
    }
}
```

## Clearing Output

For some reason or another, the developer may want to clear the collected server output and replace it with another. Also, he might want to clear the headers. Clearing all output can be performed using the method [`Response::clear()`](https://webfiori.com/docs/webfiori/http/Response#clear). To clear response body only, the method [`Response::clearBody()`](https://webfiori.com/docs/webfiori/http/Response#clearBody) can be used. To clear headers only, the method [`Response::clearHeaders()`](https://webfiori.com/docs/webfiori/http/Response#clearHeaders) can be used.

``` php
namespace app/ini/routes;

use webfiori\http\Response;
use webfiori\framework\router\Router;
use webfiori\framework\router\RouteOption;

class ClosureRoutes {
    /**
     * Create all closure routes. Include your own here.
     * @since 1.0
     */
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/closure',
            RouteOption::TO => function() {
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
