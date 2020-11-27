# Middleware
<meta name="description" content="Middleware is a way to filter HTTP request before it actually reach your application. You can think of it as a protection layer.">

In this page:
* [Introduction](#introduction)
* [The Class `AbstractMiddleware`](#the-class-abstractmiddleware)
* [Implementing Custom Middleware](#implementing-custom-middleware)
* [Registering Middleware](#registering-middleware)
* [Assigning Middleware to Routes](#assigning-middleware-to-routes)
* [Middleware Groups](#middleware-groups)
* [Priority](#priority)

## Introduction

Middaleware can be used to filter requests before reaching your application. For example, we might have a middleware which is used to check if the user is autherized to use the application or not. If he is not, then the middleware may redirect the user to login or send a response to tell him that he is not authorized. In addition to that, middleware can be used to modify the response before sending it back. For example, they can be used to add extra headers to the response depending on the content of the response.

WebFiori framework provides simple straight way for implementing middleware.

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

There are 3 methods in the middleware that the developer can implement. The method [AbstractMiddleware::before()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#before) can have code that will be executed before routing and sending back response. The method [AbstractMiddleware::after()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#after) can have code that will be executed before sending the response but after routing and processing the request. The method [AbstractMiddleware::beforeTerminate()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#beforeTerminate) can have code that will be executed after sending the request.

The developer can access the request properties using the class [`Request`](https://webfiori.com/docs/webfiori/http/Request). 

## Registering Middleware

## Assigning Middleware to Routes

## Middleware Groups

## Priority
