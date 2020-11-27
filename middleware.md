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

Middaleware is used to filter requests before reaching your application. For example, we might have a middleware which is used to check if the user is autherized to use the application or not. If he is not, then the middleware may redirect the user to login or send a response to tell him that he is not authorized. 

## The Class [`AbstractMiddleware`](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware)
Middleware represented by the class [`AbstractMiddleware`](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware). The class has abstract methods at which the developer must implement to have a functional middleware. The methods are:

* [AbstractMiddleware::before()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#before)
* [AbstractMiddleware::after()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#after)
* [AbstractMiddleware::beforeTerminate()](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware#beforeTerminate)

Middleware must be created in the folder `app/middleware` in order for the framework to auto-register it.

## Implementing Custom Middleware

## Registering Middleware

## Assigning Middleware to Routes

## Middleware Groups

## Priority
