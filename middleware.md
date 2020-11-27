# Middleware
<meta name="description" content="Middleware is a way to filter HTTP request before it actually reach your application. You can think of it as a protection layer.">

In this page:
* [Introduction](#introduction)
* [The Class `AbstractMiddleware`](#the-class-abstractmiddleware)
* [Implementing Custom Middleware](#implementing-custom-middleware)
* [Assigning Middleware to Routes](#assigning-middleware-to-routes)
* [Middleware Groups](#middleware-groups)
* [Priority](#priority)

## Introduction

Middaleware can be used to filter requests before reaching your application. For example, we might have a middleware which is used to check if the user is autherized to use the application or not. If he is not, then the middleware may redirect the user to login or send a response to tell him that he is not authorized. In addition to that, middleware can be used to modify the response before sending it back. For example, they can be used to add extra headers to the response depending on the content of the response.

WebFiori framework provides simple straight way for implementing middleware.

The following image shows how middleware works in general. The green request represents a request which has passed all middleware. The red one represents a request which reached the `Middleware 1` and failed some criteria and rejected by the middleware.

<img src="assets/img/middleware00.PNG" alt="Middleware in WebFiori Framework.">

## The Class [`AbstractMiddleware`](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware)
Middleware represented by the class [`AbstractMiddleware`](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware). The class has abstract methods at which the developer must implement to have a functional middleware. The methods are:

* [AbstractMiddleware::before()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#before)
* [AbstractMiddleware::after()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#after)
* [AbstractMiddleware::beforeTerminate()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#beforeTerminate)

Middleware must be created in the folder `app/middleware` in order for the framework to auto-register it.

## Implementing Custom Middleware

The first step in creating new middleware is to create new class in the folder `app/middleware` which extends the class [`AbstractMiddleware`](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware).

``` php 
namespace app\examples;

use webfiori\framework\middleware\AbstractMiddleware;

class MyMiddleware extends AbstractMiddleware {
    
    public function __construct() {
        parent::__construct('my-middleware');
    }
    
    public function after() {
        
    }

    public function afterTerminate() {
        
    }

    public function before() {
        
    }

}
// Return namespace if the middleware belongs to one.
return __NAMESPACE__;
```

Each middleware must have a unique name. The name is used to associate the middleware with routes. In addition to the name, there are 3 methods in the middleware that the developer can  or leave empty. The method [AbstractMiddleware::before()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#before) can have code that will be executed before routing and sending back response. The method [AbstractMiddleware::after()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#after) can have code that will be executed before sending the response but after routing and processing the request. The method [AbstractMiddleware::beforeTerminate()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#beforeTerminate) can have code that will be executed after sending the request.

The developer can access the request properties using the class [`Request`](https://webfiori.com/docs/webfiori/http/Request). Also, response can be accessed using the class [`Response`](https://webfiori.com/docs/webfiori/http/Response). To send back a response before the request reaches your application, simply call the method [Response::send()](https://webfiori.com/docs/webfiori/http/Response#send)

For example, the following middleware will stop processing the request if content type is `application/xml`.

``` php 
namespace app\examples;

use webfiori\http\Response;
use webfiori\http\Request;

use webfiori\framework\middleware\AbstractMiddleware;

class MyMiddleware extends AbstractMiddleware {
    
    public function __construct() {
        parent::__construct('my-middleware');
    }
    
    public function after() {
        
    }

    public function afterTerminate() {
        
    }

    public function before() {
        $contentType = Request::getContentType();
        if ($contentType == 'application/xml') {
            Response::setCode(422);
            Response::write('Content type not supported.');
            Response::send();
        }
    }

}
// Return namespace if the middleware belongs to one.
return __NAMESPACE__;
```

## Assigning Middleware to Routes

It is possible to assign a single middleware to a route or assign a group of middleware to a route. To achive this, simply use the option `middleware` while creating the route. This option accepts an array. The array can hold middleware names or middleware groups names.

``` php
Router::api([
    'path' => 'apis/private/private-image',
    'route-to' => 'some/private/resource.php',
    'middleware' => [
        'check-is-authorized'
    ]
]);
```

## Middleware Groups

Middleware groups is a way which is used to group multiple middleware with each other. Then the name of the group is used to assign the middleware to routes. To add a middleware to a group, simply use the method [AbstractMiddleware::addToGroup()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#addToGroup). A middleware can be added to more than one group.

One special middleware group is the group `global`. Any middleware that belongs to this group will be assigned to all routes.

``` php
namespace app\examples;

use webfiori\framework\middleware\AbstractMiddleware;

class MyMiddleware extends AbstractMiddleware {
    
    public function __construct() {
        parent::__construct('my-middleware');
        
        //Add the middleware to the global middleware group
        $this->addToGroup('global');
    }
    
    public function after() {
        
    }

    public function afterTerminate() {
        
    }

    public function before() {

    }

}
// Return namespace if the middleware belongs to one.
return __NAMESPACE__;
```

## Priority

Middleware acts as a layer in top of the application. In addition to that, a middleware can be used as a protection layer before reaching another middleware. For this reason, execution of middleware matters. The developrt can specifiy the priority of the middleware using the method [AbstractMiddleware::setPriority()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#setPriority). The higher the priority, the earlier the middleware will be reached. For example, a middleware with priority 100 will be reached before a middleware with priority 99.

``` php 
<?php

namespace app\examples;

use webfiori\framework\middleware\AbstractMiddleware;

class MyMiddleware extends AbstractMiddleware {
    
    public function __construct() {
        parent::__construct('my-middleware');
      
        $this->addToGroup('global');
        $this->setPriority(300);
    }
    
    public function after() {
        
    }

    public function afterTerminate() {
        
    }

    public function before() {

    }

}
// Return namespace if the middleware belongs to one.
return __NAMESPACE__;

```

If two middleware having same priority, the name of the middle is used as indicator of which one will get executed. For example, a middleware with name `compress-file` will be reached before a one with name `start-sesstion` in case of entering the application. After processing the request, the middleware `start-sesstion` will be reached first.
