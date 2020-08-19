
# Basic Usage

In this page:
* [Introduction](#introduction)
* [Serving Static Pages](#serving-static-pages)
* [Serving Dynamic Pages](#serving-dynamic-pages)

## Introduction

The most basic use case of the framework is to use it in serving web pages. The pages can be static or dynamic. We will be looking at both cases. The pages will be served using a concept which is called [routing](routing). Routing is simply a way to serve the correct requested resource. Pages in WebFiori Framework can be created inside the folder `app/pages`. It is possible to create pages in different place but leave this for later.

## Serving Static Pages

Lets assume that we have HTML page which hase the name `index.html`. Also, let's assume tha the page has the following code in it:

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
In order to view the page, we have to create a route for it. Assuming that this page is in the folder `app/pages`, we can use the method [`Router::view()`](https://webfiori.com/docs/webfiori/entity/router/Router#view) to create its route. The class [`ViewRoutes`](https://webfiori.com/docs/webfiori/entity/router/ViewRoutes) has one static method which the developer can use to create routes to views. The following code shows how the route is created.

``` php
namespace webfiori\entity\router;

class ViewRoutes {
    public static function create() {
        Router::view([
            'path' => '/home', 
            'route-to' => '/index.html'
        ]);
    }
}
```

Assuming that the base URL of the website is `https//example.com`, if we navigate to `https//example.com/home` the page that we have created should appear in the web browser.

## Serving Dynamic Pages

Dynamic pages are files that can have executable PHP code. It is recommeded to always place PHP code inside classes. Let's assume that we have a page that has the name `SayHi.php` inside the folder `app/pages`. Also, assume that the page has the following code:

``` php 
<?php

use webfiori\entity\Response;

class SayHi {
    public function __construct() {
        Response::append('Hi Visitor. Welcome to my website!');
    }
}
```

You will notice that we have used something new here which is the class [`Response`](https://webfiori.com/docs/webfiori/entity/Response). This class represents server response and the main functionality of the class is to collect server output and send it back. The method [`Response::append()`](https://webfiori.com/docs/webfiori/entity/Response#append) is used to append server output. 

Let's assume that route to the page is created as follows:
``` php
namespace webfiori\entity\router;

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

If we navigate to `https://localhost/say-hi`, The output in the browser will be the string `Hi Visitor. Welcome to my website!`. This can be improved further. Instead of making the page says `Hi Visitor`, we can make it say `Hi Ibrahim` or `Hi Jon`.
