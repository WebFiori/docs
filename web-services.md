# Web Services

<meta name="description" content="Web services play important role when it comes to sending and receiving data from any web server. Learn how to use the library RESTEasy to create web services and to integrate the library with WebFiori Framework.">

In this page:

* [Introduction](#introduction)
* [Main Classes](#main-classes)
* [Creating Services with Annotations](#creating-services-with-annotations)
  * [Basic Annotation-Based Service](#basic-annotation-based-service)
  * [Request Parameters](#request-parameters)
  * [Object Mapping](#object-mapping)
* [Authentication and Authorization](#authentication-and-authorization)
  * [Using RequiresAuth and AllowAnonymous](#using-requiresauth-and-allowanonymous)
  * [Accessing Auth Headers](#accessing-auth-headers)
  * [Security Context](#security-context)
* [Creating Services Traditionally](#creating-services-traditionally)
  * [Extending AbstractWebService](#extending-abstractwebservice)
  * [Adding to WebServicesManager](#adding-to-webservicesmanager)
* [Response Handling](#response-handling)
* [OpenAPI Documentation](#openapi-documentation)
* [Testing Web Services](#testing-web-services)
* [Calling Services](#calling-services)

## Introduction

Web services play important role when it comes to sending and receiving data in the web. They can be used to perform CRUD operations on a database server without having to look at implementation details from the front end side. 

WebFiori framework provides the very basic level of utilities at which it can be used to implement web services that supports data filtering and validation. The library [WebFiori HTTP](https://github.com/WebFiori/http) is one of the core libraries of the framework that can be used to create RESTful APIs in simple manner. In addition to that, this library provides abstraction when it comes to collecting server output and sending it back to web clients.

> **Note:** This library can be used as a stand alone library using composer by including this entry in the `require` part of the `composer.json` file: `"webfiori/http":"*"`.


## Main Classes

| Class | Description |
|-------|-------------|
| `WebService` | Modern service class with annotation support |
| `AbstractWebService` | Traditional base class for services |
| `WebServicesManager` | Manages and routes requests to services |
| `RequestParameter` | Represents a request parameter |
| `SecurityContext` | Manages authentication state |
| `APITestCase` | PHPUnit helper for testing APIs |

## Creating Services with Annotations

The recommended approach for creating web services uses PHP 8 attributes for clean, declarative API definitions.

### Basic Annotation-Based Service

``` php
namespace App\Apis;

use WebFiori\Http\Annotations\AllowAnonymous;
use WebFiori\Http\Annotations\GetMapping;
use WebFiori\Http\Annotations\ResponseBody;
use WebFiori\Http\Annotations\RestController;
use WebFiori\Http\WebService;

#[RestController('users', 'User management API')]
class UserService extends WebService {
    
    #[GetMapping]
    #[ResponseBody]
    #[AllowAnonymous]
    public function getUsers(): array {
        return [
            'users' => [
                ['id' => 1, 'name' => 'John'],
                ['id' => 2, 'name' => 'Jane']
            ]
        ];
    }
}
```

Key annotations:

| Annotation | Target | Description |
|------------|--------|-------------|
| `#[RestController]` | Class | Defines service name and description |
| `#[GetMapping]` | Method | Maps to GET requests |
| `#[PostMapping]` | Method | Maps to POST requests |
| `#[PutMapping]` | Method | Maps to PUT requests |
| `#[DeleteMapping]` | Method | Maps to DELETE requests |
| `#[ResponseBody]` | Method | Auto-converts return value to JSON |
| `#[AllowAnonymous]` | Method | Skips authentication check |
| `#[RequiresAuth]` | Method | Requires authentication |

### Request Parameters

Define parameters using the `#[RequestParam]` annotation:

``` php
use WebFiori\Http\Annotations\GetMapping;
use WebFiori\Http\Annotations\RequestParam;
use WebFiori\Http\Annotations\ResponseBody;
use WebFiori\Http\Annotations\RestController;
use WebFiori\Http\WebService;

#[RestController('products', 'Product API')]
class ProductService extends WebService {
    
    #[GetMapping]
    #[ResponseBody]
    #[RequestParam(name: 'id', type: 'int')]
    #[RequestParam(name: 'include_details', type: 'bool', optional: true, default: false)]
    public function getProduct(): array {
        $id = $this->getParamVal('id');
        $includeDetails = $this->getParamVal('include_details');
        
        return [
            'product' => [
                'id' => $id,
                'name' => 'Sample Product',
                'details' => $includeDetails ? ['weight' => '1kg'] : null
            ]
        ];
    }
}
```

Parameter options:

| Option | Type | Description |
|--------|------|-------------|
| `name` | string | Parameter name (required) |
| `type` | string | Data type: `string`, `int`, `float`, `bool`, `email`, `url`, `array` |
| `optional` | bool | Whether parameter is optional (default: false) |
| `default` | mixed | Default value if not provided |
| `description` | string | Parameter description for documentation |

### Object Mapping

The `#[MapEntity]` annotation automatically maps request parameters to an object:

``` php
use WebFiori\Http\Annotations\MapEntity;
use WebFiori\Http\Annotations\PostMapping;
use WebFiori\Http\Annotations\ResponseBody;
use WebFiori\Http\Annotations\RestController;
use WebFiori\Http\WebService;

#[RestController('users', 'User API')]
class UserService extends WebService {
    
    #[PostMapping]
    #[ResponseBody]
    #[MapEntity(User::class)]
    public function createUser(User $user): array {
        // $user is automatically populated from request body
        // Request: {"name": "John", "email": "john@example.com", "age": 30}
        
        return [
            'message' => 'User created',
            'user' => $user->toArray()
        ];
    }
}
```

The mapper matches request parameters to setter methods:
- `name` → `setName()`
- `email` → `setEmail()`
- `user_age` → `setUserAge()`

For custom parameter-to-setter mapping:

``` php
#[MapEntity(User::class, setters: [
    'full-name' => 'setName',
    'email-address' => 'setEmail'
])]
public function createUser(User $user): array {
    // Maps 'full-name' param to setName(), 'email-address' to setEmail()
}
```

Alternative: Use `getObject()` method:

``` php
public function createUser(): array {
    $user = $this->getObject(User::class);
    // ...
}
```

## Authentication and Authorization

### Using RequiresAuth and AllowAnonymous

``` php
#[RestController('admin', 'Admin API')]
class AdminService extends WebService {
    
    #[GetMapping]
    #[ResponseBody]
    #[AllowAnonymous]  // No authentication required
    public function getPublicInfo(): array {
        return ['status' => 'online'];
    }
    
    #[GetMapping]
    #[ResponseBody]
    #[RequiresAuth]  // Authentication required
    public function getSecretData(): array {
        return ['secret' => 'classified'];
    }
}
```

### Accessing Auth Headers

``` php
public function isAuthorized(): bool {
    $authHeader = $this->getAuthHeader();
    
    if ($authHeader === null) {
        return false;
    }
    
    $scheme = $authHeader->getScheme();      // 'basic', 'bearer', etc.
    $credentials = $authHeader->getCredentials();  // Token or encoded credentials
    
    if ($scheme === 'bearer') {
        return $this->validateToken($credentials);
    }
    
    if ($scheme === 'basic') {
        $decoded = base64_decode($credentials);
        [$username, $password] = explode(':', $decoded, 2);
        return $this->validateUser($username, $password);
    }
    
    return false;
}
```

### Security Context

Use `SecurityContext` for managing authentication state:

``` php
use WebFiori\Http\SecurityContext;
use WebFiori\Http\SecurityPrincipal;

// Set authenticated user (using a class that implements SecurityPrincipal)
$user = new AppUser(); // Must implement SecurityPrincipal interface
SecurityContext::setCurrentUser($user);

// Check authentication
if (SecurityContext::isAuthenticated()) {
    $currentUser = SecurityContext::getCurrentUser();
}

// Evaluate security expressions
SecurityContext::evaluateExpression("hasRole('ADMIN')");
SecurityContext::evaluateExpression("hasAnyRole('USER', 'ADMIN')");
SecurityContext::evaluateExpression("isAuthenticated()");
SecurityContext::evaluateExpression("hasRole('ADMIN') && hasAuthority('DELETE_USER')");
```

## Creating Services Traditionally

For more control, use the traditional approach by extending `AbstractWebService`.

### Extending AbstractWebService

``` php
use WebFiori\Http\AbstractWebService;
use WebFiori\Http\RequestMethod;
use WebFiori\Http\ParamOption;
use WebFiori\Http\ParamType;

class GetRandomService extends AbstractWebService {
    public function __construct() {
        parent::__construct('get-random-number');
        $this->addRequestMethod(RequestMethod::GET);
        
        $this->addParameters([
            'min' => [
                ParamOption::TYPE => ParamType::INT,
                ParamOption::OPTIONAL => true
            ],
            'max' => [
                ParamOption::TYPE => ParamType::INT,
                ParamOption::OPTIONAL => true
            ]
        ]);
    }

    public function isAuthorized(): bool {
        return true;
    }

    public function processRequest() {
        $min = $this->getParamVal('min') ?? 0;
        $max = $this->getParamVal('max') ?? 100;
        
        $this->sendResponse(rand($min, $max));
    }
}
```

### Adding to WebServicesManager

``` php
use WebFiori\Http\WebServicesManager;

class RandomAPI extends WebServicesManager {
    public function __construct() {
        parent::__construct();
        $this->addService(new GetRandomService());
    }
}

// Process request
$manager = new RandomAPI();
$manager->process();
```

Auto-discover services in a directory:

``` php
$manager = new WebServicesManager();
$manager->autoDiscoverServices();  // Discovers services in current directory
$manager->process();
```

## Response Handling

### JSON Response (Default)

``` php
#[GetMapping]
#[ResponseBody]
public function getData(): array {
    return ['key' => 'value'];  // Automatically converted to JSON
}
```

### Custom Response Formats

``` php
public function processRequest() {
    // JSON response
    $this->send('application/json', ['data' => 'value']);
    
    // Plain text
    $this->send('text/plain', 'Hello World');
    
    // XML response
    $xml = '<?xml version="1.0"?><root><item>value</item></root>';
    $this->send('application/xml', $xml);
    
    // With status code
    $this->sendResponse('Resource created', 201, self::I);
    
    // Error response
    $this->sendResponse('Not found', 404, self::E);
}
```

Response type constants:
- `self::I` - Info message
- `self::S` - Success message
- `self::E` - Error message

## OpenAPI Documentation

Generate OpenAPI 3.1.0 specification automatically:

``` php
#[RestController('openapi', 'API Documentation')]
class OpenAPIService extends WebService {
    
    #[GetMapping]
    #[ResponseBody]
    #[AllowAnonymous]
    public function getSpec(): array {
        $openApi = $this->getManager()->toOpenAPI();
        
        // Customize info
        $info = $openApi->getInfo();
        $info->setTitle('My API');
        $info->setVersion('1.0.0');
        $info->setDescription('API documentation');
        
        return $openApi->toArray();
    }
}
```

The generated spec can be used with:
- Swagger UI
- Postman (import as OpenAPI)
- Other OpenAPI-compatible tools

## Testing Web Services

Use `APITestCase` for unit testing:

``` php
use WebFiori\Http\APITestCase;

class UserServiceTest extends APITestCase {
    
    public function testGetUsers() {
        $manager = new UserAPI();
        
        $output = $this->callEndpoint(
            $manager,
            'GET',
            'get-users',
            ['page' => 1, 'limit' => 10]
        );
        
        $response = json_decode($output, true);
        $this->assertArrayHasKey('users', $response);
    }
    
    public function testCreateUser() {
        $manager = new UserAPI();
        
        $output = $this->callEndpoint(
            $manager,
            'POST',
            'create-user',
            ['name' => 'John', 'email' => 'john@example.com']
        );
        
        $response = json_decode($output, true);
        $this->assertEquals('User created', $response['message']);
    }
    
    public function testWithAuthentication() {
        $manager = new UserAPI();
        
        $output = $this->callEndpoint(
            $manager,
            'GET',
            'get-profile',
            [],
            ['Authorization' => 'Bearer test-token']
        );
        
        $this->assertStringContainsString('profile', $output);
    }
    
    public function testFileUpload() {
        $manager = new FileAPI();
        
        $this->addFile('document', '/path/to/test.pdf');
        
        $output = $this->callEndpoint(
            $manager,
            'POST',
            'upload-file',
            ['description' => 'Test file']
        );
        
        $response = json_decode($output, true);
        $this->assertEquals('File uploaded', $response['message']);
    }
}
```

## Calling Services

Services are called via HTTP with the `service` parameter:

```
GET https://example.com/api?service=get-users
POST https://example.com/api?service=create-user
```

In WebFiori Framework, create a route to your services manager:

``` php
Router::api([
    'path' => '/api',
    'route-to' => UserAPI::class
]);
```

Then call: `https://example.com/api?service=get-users`

## Related Articles

* [MVC Architecture](learn/mvc) - Build APIs with Controllers, Repositories, and Entities
* [Database Management](learn/database) - Use database in web services
* [Middleware](learn/middleware) - Protect API endpoints with middleware
* [Routing](learn/routing) - Create API routes
* [The Library WebFiori JSON](learn/webfiori-json) - Format API responses as JSON
* [The Class Response](learn/class-response) - Send API responses
