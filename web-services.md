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
  * [Specify Request Method of The Service](#specify-request-method-of-the-service)
  * [Adding Request Parameters](#adding-request-parameters)
  * [Implementing The Abstract Methods of the Class `AbstractWebService`](#implementing-the-abstract-methods-of-the-class-abstractwebservice)
  * [Adding The Service to The Class `WebServicesManager`](#adding-the-service-to-the-class-webservicesmanager)
  * [Processing The Request](#processing-the-request)
  * [Calling The Service](#calling-the-service)
  * [Calling The Service using WebFiori Framework](#calling-the-service-using-webfiori-framework)

## Introduction

Web services play important role when it comes to sending and receiving data in the web. They can be used to perform CRUD operations on a database server without having to look at implementation details from the front end side. 

WebFiori framework provides the very basic level of utilities at which it can be used to implement web services that supports data filtering and validation. The library [WebFiori HTTP](https://github.com/WebFiori/http) is one of the core libraries of the framework that can be used to create RESTful APIs in simple manner. In addition to that, this library provides abstraction when it comes to collecting server output and sending it back to web clients.

> **Note:** This library can be used as a stand alone library using composer by including this entry in the `require` part of the `composer.json` file: `"webfiori/http":"*"`.


## Main Classes


### The Class `RequestParameter`

The class [`RequestParameter`](https://webfiori.com/docs/webfiori/http/RequestParameter) simply represents a `GET`, `POST` body parameter. Also, this class represents the name of object property if request content type is `application/json`.

### The Class `AbstractWebService`

The class [`AbstractWebService`](https://webfiori.com/docs/webfiori/http/AbstractWebService) represents the actual web service. The class has two abstract methods that must be implemented. The first one is the method [`AbstractWebService::isAuthorized()`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService#isAuthorized) and the second method is [`AbstractWebService::processRequest()`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService#processRequest). The developer must extend this class to implement the code that will get executed when the service is called.

### The Class `WebServicesManager`

The class [`WebServicesManager`](https://webfiori.com/docs/webfiori/restEasy/WebServicesManager) is used to manage a set of related web services. For example, the developer might have 4 services for performing CRUD operations on a resource. The 4 services must be added to an instance of this class. When creating a route, the route should point to a class which is a child of this class.

## Creating a Simple Web Service

Assuming that a developer wants to implement a service that sends back a random number, then he must follow the following steps to have that service:

* Create new class that extends the class [`AbstractWebService`](https://webfiori.com/docs/webfiori/http/AbstractWebService). 
* Specify a name for the web service.
* Specify request methods of the service.
* Add parameters to the service (optionally).
* Implement the abstract methods of the class [`AbstractWebService`](https://webfiori.com/docs/webfiori/http/AbstractWebService).
* Add the service to an instance of the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/http/WebServicesManager).

> **Note:** If the service is created inside WebFiori framework, then there is a need for a final step which is to create a route to services manager.

### Extending The Class `AbstractWebService`

The class [`AbstractWebService`](https://webfiori.com/docs/webfiori/http/AbstractWebService) will basically represent a web service (or API end point) that will be get executed when called. The [constructure](https://webfiori.com/docs/webfiori/http/AbstractWebService#__construct) of the class accepts one parameter which is the name of the service. Each service must have a unique name as it will be used to call it. Assuming that the name of the service that will be create is `get-random-number`.

``` php
use webfiori\http\AbstractWebService;

class GetRandomService extends AbstractWebService {
    public function __construct() {
        parent::__construct('get-random-number');
    }
}
```

### Specify Request Method of The Service

For every new service, the developer must specify allowed request methods at which the service can be called with. Request method can be something like `GET`, `POST` or `PUT`. To set allowed request method(s), the method [`AbstractWebService::addRequestMethod()`](https://webfiori.com/docs/webfiori/http/AbstractWebService#addRequestMethod) can be used. It is possible to have more than one request method for a service.

``` php
use webfiori\http\AbstractWebService;

class GetRandomService extends AbstractWebService {
    public function __construct() {
        parent::__construct('get-random-number');
        $this->addRequestMethod('get');
        $this->addRequestMethod('post');
    }
}
```

### Adding Request Parameters

Request parameters are represented by the class [`RequestParameter`](https://webfiori.com/docs/webfiori/http/RequestParameter). To add request parameter to a service, the method [`AbstractWebService::addParameter()`](https://webfiori.com/docs/webfiori/http/AbstractWebService#addParameter). Each parameter must have a name and a type. The two can be specified in the [constructor](https://webfiori.com/docs/webfiori/http/RequestParameter#__construct) of the class.

Assuming that the developer will create the API in a way it can accepts two values and the random number will be between the two. 

``` php
use webfiori\http\AbstractWebService;
use webfiori\http\RequestParameter;

class GetRandomService extends AbstractWebService {
    public function __construct() {
        parent::__construct('get-random-number');
        $this->addRequestMethod('get');
        $this->addRequestMethod('post');
        
        $this->addParameter(new RequestParameter('min', 'integer', true));
        $this->addParameter(new RequestParameter('max', 'integer', true));
    }
}
```

### Implementing The Abstract Methods of the Class `AbstractWebService`

The class [`AbstractWebService`](https://webfiori.com/docs/webfiori/http/AbstractWebService) has two abstract methods at which they must be implemented. The first one is the method [`AbstractWebService::isAuthorized()`](https://webfiori.com/docs/webfiori/http/AbstractWebService#isAuthorized). This method is used to check if the one who is calling the service is allowed to call it or not. The second method is [`AbstractWebService::processRequest()`](https://webfiori.com/docs/webfiori/http/AbstractWebService#processRequest). This method is simply used to process client's request and send back a response.

``` php
use webfiori\http\AbstractWebService;
use webfiori\http\RequestParameter;

class GetRandomService extends AbstractWebService {
    public function __construct() {
        parent::__construct('get-random-number');
        $this->addRequestMethod('get');
        $this->addRequestMethod('post');
        
        $this->addParameter(new RequestParameter('min', 'integer', true));
        $this->addParameter(new RequestParameter('max', 'integer', true));
    }

    public function isAuthorized() {
        //Possibly, check if client is authorized to call the API or not.
        //If he is not authorized, return false.
    }

    public function processRequest() {
        $max = $this->getParamVal('max');
        $min = $this->getParamVal('min');
        
        // Since the two are optional, they can be null
        if ($max !== null && $min !== null) {
            $random = rand($min, $max);
        } else {
            $random = rand();
        }
        $this->sendResponse($random);
    }

}
```

### Adding The Service to The Class `WebServicesManager`

The final step is to add the service to a services manager. The main aim of services manager is to group related services in one place. Also, it is used to manage the incoming requests and send them to correct service. It is always recommended to have a class which extends the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/http/WebServicesManager) and use it to group the services. To add a service to a services manager, the method 
[`WebServicesManager::addService()`](https://webfiori.com/docs/webfiori/http/WebServicesManager#addService) can be used.

``` php
use webfiori\http\WebServicesManager;
use GetRandomService;

class RandomGenerator extends WebServicesManager {
    public function __construct() {
        parent::__construct();
        $this->addService(new GetRandomService());
    }
}

```

### Processing The Request

In background, processing the request is performed by the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/http/WebServicesManager), to be specific, the method [`WebServicesManager::process()`](https://webfiori.com/docs/webfiori/http/WebServicesManager#process). For the service that was created, the developer need to create an instance of the class `RandomGenerator`. After that, he have to call the method [`WebServicesManager::process()`](https://webfiori.com/docs/webfiori/http/WebServicesManager#process)

``` php
use webfiori\http\WebServicesManager;
use GetRandomService;

class RandomGenerator extends WebServicesManager {
    public function __construct() {
        parent::__construct();
        $this->addService(new GetRandomService());
    }
}

$manager = new RandomGenerator();
$manager->process();
```

> **Note:** If the library is used in WebFiori Framework, then no need to perform the last step as the router will handle it. The code assumes that the library is used outside the scope of WebFiori Framework.

### Calling The Service

Assuming that the base URL of the website is `https://example.com` and the class `RandomGenerator` is at the root, then the service can be called using the link `https://example.com/RandomGenerator.php`. If navigated to the given URL in any web browser, the output should be similar to the following:

``` json
{
    "message":"Service name is not set.", 
    "type":"error", 
    "http-code":404
}
```

This message means that service name is not included in the request and the manager does not know which service to call. To fix this issue, the name of the service that will be called must be included as `GET` parameter with the name `service`. In case the developer would like to call the service `GetRandomService`, he can use the value `get-random-number` for the parameter. So, the URL for calling the service would be `https://example.com/RandomGenerator.php?service=get-random-number`. Calling the service that way would result in the following output.

``` json
{
    "message":1480407584, 
    "http-code":200
}
```

It is also possible to can set a value for the parameter `min` or `max` using same way. For example, sending a request to `https://example.com/RandomGenerator.php?service=get-random-number` would result in getting a random number between 1 and 5.

### Calling The Service using WebFiori Framework

The library is fully integrated with WebFiori Framework. In order to call a service, the developer have to create a route to a class which extends the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/http/WebServicesManager). For more information about how to create routes to web services, [check here](learn/routing#api-route)

**Next: [Midddleware](learn/middleware)**

**Previous: [Database Management](learn/database)**
