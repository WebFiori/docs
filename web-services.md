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

WebFiori framework provides the very basic level of utilities at which it can be used to implement web services that supports data filtering and validation. The library [RESTEasy](https://github.com/WebFiori/restEasy) is one of the core libraries of the framework that can be used to create RESTful APIs in simple manner.

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

Assuming that we want to implement a service that sends back a random number, we have to follow the following steps to have that service:

* Create new class that extends the class [`AbstractWebService`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService). 
* Specify a name for the web service.
* Specify request methods of the service.
* Add parameters to the service (optionally).
* Implement the abstract methods of the class [`AbstractWebService`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService).
* Add the service to an instance of the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/restEasy/WebServicesManager).

> **Note:** If the service is created inside WebFiori framework, then we need a final step which is to create a route to the manager.

### Extending The Class `AbstractWebService`

The class [`AbstractWebService`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService) will basically represent your web service that will be get executed when called. The [constructure](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService#__construct) of the class accepts one parameter which is the name of the service. Each service must have a unique name as it will be used to call it later. Let's assume that the name of the service that we will create is `get-random-number`.

``` php
use webfiori\restEasy\AbstractWebService;

class GetRandomService extends AbstractWebService {
    public function __construct() {
        parent::__construct('get-random-number');
    }
}
```

### Specify Request Method of The Service

For every new service, the developer must specify allowed methods at which the service can be called with. Request method can be something like `GET`, `POST` or `PUT`. To set allowed request method(s), the method [`AbstractWebService::addRequestMethod()`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService#addRequestMethod) can be used. It is possible to have more than one request method for a service.

``` php
use webfiori\restEasy\AbstractWebService;

class GetRandomService extends AbstractWebService {
    public function __construct() {
        parent::__construct('get-random-number');
        $this->addRequestMethod('get');
        $this->addRequestMethod('post');
    }
}
```

### Adding Request Parameters

Request parameters represented by the class [`RequestParameter`](https://webfiori.com/docs/webfiori/restEasy/RequestParameter). To add request parameter to a service, the method [`AbstractWebService::addParameter()`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService#addParameter). Each parameter must have a name and a type. The two can be specified in the [constructor](https://webfiori.com/docs/webfiori/restEasy/RequestParameter#__construct) of the class.

What we will do is to add two parameters to specify a minimum and a maximum value at which the random will be taken from. We will make the two parameters optional.

``` php
use webfiori\restEasy\AbstractWebService;
use webfiori\restEasy\RequestParameter;

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

The class [`AbstractWebService`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService) has two abstract methods at which they must be implemented. The first one is the method [`AbstractWebService::isAuthorized()`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService#isAuthorized). This method is used to check if the one who is calling the service is allowed to call it or not. The second method is [`AbstractWebService::processRequest()`](https://webfiori.com/docs/webfiori/restEasy/AbstractWebService#processRequest). This method is simply used to process client's request and send back a response.

For now, we will only process the request. 

``` php
use webfiori\restEasy\AbstractWebService;
use webfiori\restEasy\RequestParameter;

class GetRandomService extends AbstractWebService {
    public function __construct() {
        parent::__construct('get-random-number');
        $this->addRequestMethod('get');
        $this->addRequestMethod('post');
        
        $this->addParameter(new RequestParameter('min', 'integer', true));
        $this->addParameter(new RequestParameter('max', 'integer', true));
    }

    public function isAuthorized() {
        
    }

    public function processRequest() {
        $max = $this->getParamVal('max');
        $min = $this->getParamVal('min');
        
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

The final step is to add the service to a services managers. The main aim of services managers is to group related services in one place. Also, they are used to manage the incoming request and send it to correct service. It is always recommended to have a class which extends the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/restEasy/WebServicesManager) and use it to group the services. To add a service to a services manager, we use the method 
[`WebServicesManager::addService()`](https://webfiori.com/docs/webfiori/restEasy/WebServicesManager#addService)

``` php
use webfiori\restEasy\WebServicesManager;
use GetRandomService;

class RandomGenerator extends WebServicesManager {
    public function __construct() {
        parent::__construct();
        $this->addService(new GetRandomService());
    }
}

```

### Processing The Request

Processing the request is performed by the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/restEasy/WebServicesManager), to be specific, the method [`WebServicesManager::process()`](https://webfiori.com/docs/webfiori/restEasy/WebServicesManager#process). For the service that we have  created, we need to create an instance of the class `RandomGenerator`. After that, we have to call the method [`WebServicesManager::process()`](https://webfiori.com/docs/webfiori/restEasy/WebServicesManager#process)

``` php
use webfiori\restEasy\WebServicesManager;
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

### Calling The Service

Assuming that the base URL of the website is `https://example.com` and the class `RandomGenerator` is at the root, then we can go to the link `https://example.com/RandomGenerator.php`. If we navigate to the given URL in any web browser, the output will be similar to the following:

``` json
{
    "message":"Service name is not set.", 
    "type":"error", 
    "http-code":404
}
```

This message means that service name is not included in the request and the manager does not know which service to call. To fix this issue, we have to include a `GET` parameter with the name `service`. The value of the parameter must be the name of the called service. In case we would like to call the service `GetRandomService`, we use the value `get-random-number` for the parameter. So, the URL for calling the service would be `https://example.com/RandomGenerator.php?service=get-random-number`. If we called the service like that, the output would be similar to the following:

``` json
{
    "message":1480407584, 
    "http-code":200
}
```

The `message` contains the randomly generated number. We can also set a value to the parameter `min` or `max` using same way we used to call the service. For example, if we send a request to `https://example.com/RandomGenerator.php?service=get-random-number`, we would get a random number between 1 and 5.

### Calling The Service using WebFiori Framework

The library is fully integrated with WebFiori Framework. In order to call a service, the developer have to create a route to a class which extends the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/restEasy/WebServicesManager). For more information about how to create routes to web services, [check here](learn/routing#api-route)
