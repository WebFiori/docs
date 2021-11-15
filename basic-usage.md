
# Basic Usage

<meta name="description" content="The most basic use case of the framework is to use it in serving web pages. The pages can be static or dynamic.">

In this page:
* [Introduction](#introduction)
* [Serving Static Pages](#serving-static-pages)
* [Serving Dynamic Pages](#serving-dynamic-pages)

## Introduction

The most basic use case of the framework is to use it in serving web pages. The pages can be static or dynamic. The pages will be served using a concept which is called [routing](learn/routing). Routing is simply a way to serve the correct requested resource. Pages in WebFiori Framework can be created inside the folder `app/pages`. It is possible to create pages in different place. The folder is only used to organize your application classes.

## Serving Static Pages

Assuming that there is an HTML page which has the name `index.html`. Also, let's assume tha the page has the following code in it:

``` html
<!DOCTYPE html>
<html>
    <head>
        <title>Hello</title>
        <meta charset="UTF-8">
    </head>
    <body>
        <div>
            <p>
                Welcome to My Home Page
            </p>
            <p>
                This page is served using 
                <a href="https://webfiori.com/webfiori">WebFiori Framework</a>
            </p>
        </div>
    </body>
</html>
```

In order to view the page, A route must be created that points to it. Assuming that this page is in the folder `app/pages`, the method [`Router::page()`](https://webfiori.com/docs/webfiori/framework/router/Router#page) to create its route. The class [`ViewRoutes`](https://webfiori.com/docs/app/ini/routes/ViewRoutes) has one static method which the developer can use to create routes to pages (or views). The following code shows how the route is created.

``` php
namespace webfiori\framework\router;

class ViewRoutes {
    public static function create() {
        Router::page([
            'path' => '/home', 
            'route-to' => '/index.html'
        ]);
    }
}
```

Assuming that the base URL of the website is `https//example.com`, if navigated to `https//example.com/home` the page that was created should appear in web browser.

## Serving Dynamic Pages

Dynamic pages are files that can have executable PHP code. Also, it is possible to represent dynamic page using a class and create a route to that class.

### Routing to PHP Files

Assuming that a page which has the name `SayHi.php` inside the folder `app/pages`. Also, assuming that the page has the following code:

``` php 
use webfiori\framework\Response;

class SayHi {
    public function __construct() {
        Response::append('Hi Visitor. Welcome to my website!');
    }
}

// This statement must be included at the end of the class 
// file for pages if the route will be created using file name
return __NAMESPACE__;
```

Notice that in the code a new class is used which is the class [`Response`](https://webfiori.com/docs/webfiori/http/Response). This class represents server response and the main functionality of it is to collect server output and send it back to the client. The method [`Response::append()`](https://webfiori.com/docs/webfiori/http/Response#append) is used to append server output. You can learn more about this class [here](learn/class-response).

Assuming that a route to the page is created as follows:

``` php
namespace webfiori\framework\router;

class ViewRoutes {
    /**
     * Create all views routes. Include your own here.
     * @since 1.0
     */
    public static function create() {
        Router::view([
            'path' => '/say-hi', 
            'route-to' => '/SayHi.php'
        ]);
    }
}
```

If navigated to `https://localhost/say-hi`, The output in the browser will be the string `Hi Visitor. Welcome to my website!`. This can be improved further. Instead of making the page says `Hi Visitor`, it is possible to make it say `Hi Ibrahim` or `Hi Jon`. Achiving this goal can be done by using variables when creating the route. A variable is a part of the path in the URL which can have any value. Its name is enclosed between two curly braces (e.g. `{user-name}`). The code for creating a route with a variable will be similar to the following:

``` php
namespace webfiori\framework\router;

class ViewRoutes {
    /**
     * Create all views routes. Include your own here.
     * @since 1.0
     */
    public static function create() {
        Router::view([
            'path' => '/say-hi/{person-name}', 
            'route-to' => '/SayHi.php'
        ]);
    }
}
```

Accessing the value of the variable inside the class `SayHi` can be perfomed by using the method [`Router::getVarValue()`](https://webfiori.com/docs/webfiori/framework/router/Router#getVarValue) as follows:

``` php
use webfiori\framework\router\Router;
use webfiori\http\Response;

class SayHi {
    public function __construct() {
        $personName = Router::getVarValue('person-name');
        Response::append('Hi '.$personName.'. Welcome to my website!');
    }
}

return __NAMESPACE__;
```

Now when navigating to `https://example.com/say-hi/Ibrahim BinAlshikh`, the message `Hi Ibrahim BinAlshikh. Welcome to my website!` will appear. The string `Ibrahim BinAlshikh` can be replaced by anything and it will say "hi" to it.

### Routing to PHP Classes

Assuming that the following PHP class represent a dynamic web page:

``` php
use webfiori\framework\router\Router;
use webfiori\framework\Response;

class SayHi {
    public function __construct() {
        $personName = Router::getVarValue('person-name');
        Response::append('Hi '.$personName.'. Welcome to my website!');
    }
}
```

It is possible to have a route which points to the class directly as follows:

``` php
namespace webfiori\framework\router;

use SayHi;

class ViewRoutes {
    /**
     * Create all views routes. Include your own here.
     * @since 1.0
     */
    public static function create() {
        Router::addRoute([
            'path' => '/say-hi/{person-name}', 
            'route-to' => SayHi::class
        ]);
    }
}
```

This syntax is much better when creating routes but it has one issue. It may not help in identifying the exact location of a bug in your class if it has one.

**Next: [Routing](learn/routing)**

**Previous: [Folder Structure](learn/folder-structure)**
