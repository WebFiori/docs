# Web Services

<meta name="description" content="Web services play important role when it comes to sending and receiving data from any web server. Learn how to use the library RESTEasy to create web services and to integrate the library with WebFiori Framework.">

In this page:

* [Introduction](#introduction)
* [Main Classes](#main-classes)
  * [The Class `RequestParameter`](#the-class-requestparameter)
  * [The Class `AbstractWebService`](#the-class-abstractwebservice)
  * [The Class `WebServicesManager`](#the-class-webservicesmanager)
* [Creating a Simple Web Service](#creating-a-simple-web-service)
  * [Extending The Class `AbstractWebService`](#extending-the-class-abstractwebservice)

## Introduction

Web services play important role when it comes to sending and receiving data in the web. They can be used to perform CRUD operations on a database server without having to look at implementation details from the front end side. 

WebFiori framework provides the very basic level of utilities at which it can be used to implement web services that supports data filtering and validation. The library [RESTEasy](https://github.com/usernane/restEasy) is one of the core libraries of the framework that can be used to create RESTful APIs in simple manner.

> **Note:** This library can be used as a stand alone library using composer by including this entry in the `require` part of the `composer.json` file: `"webfiori/rest-easy":"*"`.


## Main Classes

In this section, we introduce only the classes at which the developer will have to use to create web services.

### The Class `RequestParameter`

The class [`RequestParameter`](https://webfiori.com/docs/webfiori/restEasy/RequestParameter) simply represents a `GET`, `POST` body parameter. Also, this class represents the name of object property if request content type is `application/json`.

### The Class `AbstractWebService`

The class [`AbstractWebService`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService) represents the actual web service. The class has two abstract methods that must be implemented. The first one is the method [`AbstractWebService::isAuthorized()`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService#isAuthorized) and the second method is [`AbstractWebService::processRequest()`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService#processRequest). The developer must extend this class to implement the code that will get executed when the service is called.

### The Class `WebServicesManager`

The class [`WebServicesManager`](https://webfiori.com/docs/webfiori/restEasy/WebServicesManager) is used to manage a set of related web services. For example, the developer might have 4 services for performing CRUD operations on a resource. The 4 services must be added to an instance of this class. When creating a route, the route should point to a class which is a child of this class.

## Creating a Simple Web Service

In order to have a functional web service, we have to do two steps. First one is to create new class that extends the class [`AbstractWebService`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService) and implement its abstract methods. The second step is to have the newly created service added to an instance of the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/restEasy/WebServicesManager). If the service is created inside WebFiori framework, then we need a final step which is to create a route to the manager.

### Extending The Class `AbstractWebService`
