# Web Services

<meta name="description" content="Web services play important role when it comes to sending and receiving data from any web server. Learn how to use the library RESTEasy to create web services and to integrate the library with WebFiori Framework.">

In this page:

* [Introduction](#introduction)
* [Main Classes](#main-classes)
* [Creating Services with Annotations](#creating-services-with-annotations)
  * [Basic Annotation-Based Service](#basic-annotation-based-service)
  * [Request Parameters](#request-parameters)
  * [Object Mapping](#object-mapping)
  * [Cross-Field Validation](#cross-field-validation)
  * [Reusable Parameter Sets](#reusable-parameter-sets)
  * [Content Negotiation](#content-negotiation)
  * [Request Content Type Control](#request-content-type-control)
  * [ResponseEntity](#responseentity)
* [Authentication and Authorization](#authentication-and-authorization)
  * [Using RequiresAuth and AllowAnonymous](#using-requiresauth-and-allowanonymous)
  * [Using PreAuthorize](#using-preauthorize)
  * [Accessing Auth Headers](#accessing-auth-headers)
  * [Security Context](#security-context)
* [Creating Services Traditionally](#creating-services-traditionally)
  * [Extending AbstractWebService](#extending-abstractwebservice)
  * [Adding to WebServicesManager](#adding-to-webservicesmanager)
* [Response Handling](#response-handling)
* [OpenAPI Documentation](#openapi-documentation)
  * [Declarative Response Descriptions](#declarative-response-descriptions)
  * [Custom Route Paths](#custom-route-paths)
  * [Built-in OpenAPI Spec Service](#built-in-openapi-spec-service)
  * [Namespace Scanning](#namespace-scanning)
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
| `WebServicesManager` | Manages and routes requests to services (traditional) |
| `ServiceRouter` | Auto-discovers and registers service routes (recommended) |
| `RequestProcessor` | Processes a single service against a request |
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
| `#[RestController]` | Class | Defines service name, description, and path |
| `#[GetMapping]` | Method | Maps to GET requests |
| `#[PostMapping]` | Method | Maps to POST requests |
| `#[PutMapping]` | Method | Maps to PUT requests |
| `#[DeleteMapping]` | Method | Maps to DELETE requests |
| `#[PatchMapping]` | Method | Maps to PATCH requests |
| `#[ResponseBody]` | Method | Auto-converts return value to JSON |
| `#[RequestParam]` | Method | Declares a request parameter |
| `#[ApiResponse]` | Method | Declares a possible response for OpenAPI spec (repeatable) |
| `#[AllowAnonymous]` | Method | Skips authentication check |
| `#[RequiresAuth]` | Method | Requires authentication |
| `#[PreAuthorize]` | Method/Class | Expression-based authorization |
| `#[Validate]` | Method | Cross-field validation function |
| `#[UseParameterSet]` | Method | Imports a reusable parameter set |
| `#[Produces]` | Method | Content negotiation (Accept header matching) |
| `#[Consumes]` | Method | Per-method content type control (request body types) |

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

### Cross-Field Validation

Use the `#[Validate]` attribute to run custom validation logic after parameters are filtered:

``` php
use WebFiori\Http\Annotations\Validate;

#[PostMapping]
#[ResponseBody]
#[RequestParam(name: 'password', type: 'string')]
#[RequestParam(name: 'password_confirm', type: 'string')]
#[RequestParam(name: 'email', type: 'email')]
#[Validate('validateRegistration')]
public function register(): array {
    // Only reached if validation passes
    return ['message' => 'User registered'];
}

private function validateRegistration(array $inputs): array {
    $errors = [];
    if ($inputs['password'] !== $inputs['password_confirm']) {
        $errors['password_confirm'] = 'Passwords do not match.';
    }
    if (strlen($inputs['password']) < 8) {
        $errors['password'] = 'Password must be at least 8 characters.';
    }
    return $errors; // Empty array = valid
}
```

If the validation method returns a non-empty array, the framework sends a 422 response with the errors.

### Reusable Parameter Sets

Use the `ParameterSet` interface and `#[UseParameterSet]` attribute to share parameter definitions across services:

``` php
use WebFiori\Http\ParameterSet;
use WebFiori\Http\ParamOption;
use WebFiori\Http\ParamType;

class PaginationParams implements ParameterSet {
    public function getParameters(): array {
        return [
            'page' => [
                ParamOption::TYPE => ParamType::INT,
                ParamOption::OPTIONAL => true,
                ParamOption::DEFAULT => 1
            ],
            'per-page' => [
                ParamOption::TYPE => ParamType::INT,
                ParamOption::OPTIONAL => true,
                ParamOption::DEFAULT => 20
            ]
        ];
    }
}
```

Then use it on any method:

``` php
use WebFiori\Http\Annotations\UseParameterSet;

#[GetMapping]
#[ResponseBody]
#[UseParameterSet(PaginationParams::class)]
#[RequestParam(name: 'category', type: 'string', optional: true)]
public function listProducts(): array {
    $page = $this->getParamVal('page');
    $perPage = $this->getParamVal('per-page');
    // ...
}
```

### Content Negotiation

Use `#[Produces]` to declare what content types a method can return. The framework matches against the client's `Accept` header:

``` php
use WebFiori\Http\Annotations\Produces;
use WebFiori\Http\MediaType;

#[GetMapping]
#[Produces(MediaType::JSON, MediaType::XML)]
public function getReport(): ResponseEntity {
    // Framework returns 406 Not Acceptable if client doesn't accept JSON or XML
    return ResponseEntity::ok(['report' => '...']);
}
```

Available `MediaType` constants: `JSON`, `XML`, `HTML`, `PLAIN`, `CSV`, `PDF`, `FORM`, `MULTIPART`, `OCTET_STREAM`.

### Request Content Type Control

Use `#[Consumes]` to declare which content types a method accepts in the request body. This overrides the default allowed types (`application/x-www-form-urlencoded`, `multipart/form-data`, `application/json`) for POST and PUT requests:

``` php
use WebFiori\Http\Annotations\Consumes;
use WebFiori\Http\Annotations\PostMapping;
use WebFiori\Http\Annotations\ResponseBody;
use WebFiori\Http\MediaType;
use WebFiori\Http\ResponseEntity;

#[PostMapping]
#[Consumes(MediaType::OCTET_STREAM)]
#[ResponseBody]
public function uploadBinary(): ResponseEntity {
    // Only application/octet-stream is accepted
    // Default types (form, json) are REJECTED
    $body = file_get_contents('php://input');
    $size = strlen($body);
    
    return ResponseEntity::created(['size' => $size, 'md5' => md5($body)]);
}
```

Key behaviors:

- **No `#[Consumes]`**: The default three types apply (backward compatible).
- **With `#[Consumes]`**: Only the listed types are accepted. The defaults are overridden, not merged.
- **Non-standard types**: When a non-parseable content type is used (not form-encoded, multipart, or JSON), parameter filtering is automatically skipped. The raw body is available via `php://input`.
- **Standard types in `#[Consumes]`**: If you list a parseable type (e.g. `MediaType::FORM`), normal parameter filtering still applies for requests using that type.

Multiple content types on one method:

``` php
#[PostMapping]
#[Consumes(MediaType::OCTET_STREAM, MediaType::XML, 'text/xml')]
#[ResponseBody]
public function acceptMultiple(): ResponseEntity {
    $contentType = $this->getManager()->getRequest()->getContentType();
    $body = file_get_contents('php://input');
    
    // Handle based on actual content type received
    if (str_contains($contentType, 'xml')) {
        return $this->handleXml($body);
    }
    
    return $this->handleBinary($body);
}
```

Mixing standard and non-standard types:

``` php
#[PostMapping]
#[Consumes(MediaType::FORM, MediaType::OCTET_STREAM)]
#[ResponseBody]
public function flexible(): ResponseEntity {
    $contentType = $this->getManager()->getRequest()->getContentType();
    
    if (str_contains($contentType, 'octet-stream')) {
        // Raw binary — no parameter filtering
        $body = file_get_contents('php://input');
        return ResponseEntity::ok(['raw_size' => strlen($body)]);
    }
    
    // Form-encoded — normal parameter filtering applies
    $name = $this->getParamVal('name');
    return ResponseEntity::ok(['name' => $name]);
}
```

> **Note:** `#[Consumes]` only affects POST and PUT requests. GET, DELETE, and PATCH requests bypass content type validation regardless of the annotation.

### ResponseEntity

For more control over HTTP status codes and content types from `#[ResponseBody]` methods, return a `ResponseEntity`:

``` php
use WebFiori\Http\ResponseEntity;

#[PostMapping]
#[ResponseBody]
public function createUser(User $user): ResponseEntity {
    $this->userRepo->save($user);
    return ResponseEntity::created(['user' => $user->toArray()]);
}

#[GetMapping]
#[ResponseBody]
#[RequestParam(name: 'id', type: 'int')]
public function getUser(): ResponseEntity {
    $user = $this->userRepo->findById($this->getParamVal('id'));
    
    if ($user === null) {
        return ResponseEntity::notFound(['error' => 'User not found']);
    }
    
    return ResponseEntity::ok(['user' => $user->toArray()]);
}

#[DeleteMapping]
#[ResponseBody]
#[RequestParam(name: 'id', type: 'int')]
public function deleteUser(): ResponseEntity {
    $this->userRepo->deleteById($this->getParamVal('id'));
    return ResponseEntity::noContent();
}
```

Static factory methods:

| Method | HTTP Status |
|--------|-------------|
| `ResponseEntity::ok($body)` | 200 |
| `ResponseEntity::created($body)` | 201 |
| `ResponseEntity::noContent()` | 204 |
| `ResponseEntity::badRequest($body)` | 400 |
| `ResponseEntity::unauthorized($body)` | 401 |
| `ResponseEntity::forbidden($body)` | 403 |
| `ResponseEntity::notFound($body)` | 404 |
| `ResponseEntity::error($body)` | 500 |

Or construct directly with any status code:

``` php
return new ResponseEntity(['data' => $value], 202, 'application/json');
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

### Using PreAuthorize

For fine-grained authorization based on roles and authorities, use `#[PreAuthorize]` with security expressions:

``` php
use WebFiori\Http\Annotations\PreAuthorize;

#[RestController('admin', 'Admin API')]
class AdminService extends WebService {

    #[GetMapping]
    #[ResponseBody]
    #[PreAuthorize("hasRole('ADMIN')")]
    public function getAdminDashboard(): array {
        return ['dashboard' => '...'];
    }

    #[DeleteMapping]
    #[ResponseBody]
    #[PreAuthorize("hasRole('ADMIN') && hasAuthority('DELETE_USER')")]
    public function deleteUser(): array {
        // Only admins with DELETE_USER authority can reach this
        return ['message' => 'User deleted'];
    }

    #[GetMapping]
    #[ResponseBody]
    #[PreAuthorize("hasAnyRole('USER', 'ADMIN')")]
    public function getProfile(): array {
        return ['profile' => '...'];
    }
}
```

Supported expressions:
- `hasRole('ROLE_NAME')` — user has the specified role
- `hasAnyRole('ROLE1', 'ROLE2')` — user has at least one of the roles
- `hasAuthority('AUTHORITY')` — user has the specified authority
- `isAuthenticated()` — user is authenticated
- Logical operators: `&&`, `||`, `!`

`#[PreAuthorize]` can also be applied at the class level to apply to all methods.

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

The library generates OpenAPI 3.1.0 specifications automatically from your service definitions.

### Declarative Response Descriptions

Use the repeatable `#[ApiResponse]` attribute to declare possible responses on each method:

``` php
use WebFiori\Http\Annotations\ApiResponse;

#[GetMapping]
#[ResponseBody]
#[ApiResponse(status: '200', description: 'List of products')]
#[ApiResponse(status: '404', description: 'Product not found')]
#[RequestParam(name: 'id', type: 'int', optional: true)]
public function getProducts(?int $id): array {
    // ...
}

#[PostMapping]
#[ResponseBody]
#[ApiResponse(status: '201', description: 'Product created')]
#[ApiResponse(status: '400', description: 'Invalid input')]
#[RequestParam(name: 'name', type: 'string')]
public function createProduct(string $name): array {
    // ...
}
```

If no `#[ApiResponse]` is present, the spec defaults to `200 - Successful operation`. Programmatic `addResponse()` calls take priority over annotations.

### Custom Route Paths

Use the `path` property on `#[RestController]` to set a multi-segment URL path independent of the service name:

``` php
#[RestController(name: 'login', path: 'auth/login', description: 'Authentication endpoint')]
class LoginService extends WebService {
    // Route: /apis/auth/login
    // Service name: login (used for lookups)
}
```

- `name` — service identifier (no slashes, used for internal lookups)
- `path` — URL mount point (slashes allowed, used in OpenAPI spec and routing)
- If `path` is not set, the service name is used as the path

### Built-in OpenAPI Spec Service

Use `OpenAPISpecService` to expose your spec as a live endpoint without writing boilerplate:

``` php
use WebFiori\Http\OpenAPI\OpenAPISpecService;
use WebFiori\Http\RequestProcessor;

$specService = new OpenAPISpecService(
    'App\\Apis',       // Namespace to scan for #[RestController] classes
    '/apis',           // Base path prefix in the spec
    'My API',          // API title
    '1.0.0'            // API version
);

$processor = new RequestProcessor();
$processor->process($specService);
```

Point Swagger UI at this endpoint to get auto-generated, always-current documentation.

### Namespace Scanning

Use `OpenAPIGenerator` to discover services and generate specs without manual registration:

``` php
use WebFiori\Http\OpenAPI\OpenAPIGenerator;

$generator = new OpenAPIGenerator();

// Auto-discover all #[RestController] classes in a namespace
$spec = $generator->generateFromNamespace(
    'App\\Apis',
    'My API',
    '1.0.0',
    '/apis'
);

echo $spec->toJSON();
```

Or use the explicit approach for full control:

``` php
$spec = $generator->generate(
    [new UserService(), new ProductService()],
    'My API',
    '1.0.0',
    '/apis'
);
```

### Returning JsonI from ResponseBody

When a `#[ResponseBody]` method returns a `JsonI` object (such as the OpenAPI spec itself), the framework serializes it directly via `toJSON()` without wrapping or metadata:

``` php
use WebFiori\Json\JsonI;

#[GetMapping]
#[ResponseBody]
#[AllowAnonymous]
public function getSpec(): JsonI {
    $generator = new OpenAPIGenerator();
    return $generator->generateFromNamespace('App\\Apis', 'My API', '1.0.0', '/apis');
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
        
        $output = $this->getRequest($manager, 'get-users', [
            'page' => 1, 
            'limit' => 10
        ]);
        
        $response = json_decode($output, true);
        $this->assertArrayHasKey('users', $response);
    }
    
    public function testCreateUser() {
        $manager = new UserAPI();
        
        $output = $this->postRequest($manager, 'create-user', [
            'name' => 'John', 
            'email' => 'john@example.com'
        ]);
        
        $response = json_decode($output, true);
        $this->assertEquals('User created', $response['message']);
    }
    
    public function testUpdateUser() {
        $manager = new UserAPI();
        
        $output = $this->putRequest($manager, 'update-user', [
            'id' => 1,
            'name' => 'John Updated'
        ]);
        
        $response = json_decode($output, true);
        $this->assertEquals('User updated', $response['message']);
    }
    
    public function testDeleteUser() {
        $manager = new UserAPI();
        
        $output = $this->deleteRequest($manager, 'delete-user', [
            'id' => 1
        ]);
        
        $response = json_decode($output, true);
        $this->assertEquals('User deleted', $response['message']);
    }
    
    public function testWithAuthentication() {
        $manager = new UserAPI();
        
        $output = $this->getRequest(
            $manager,
            'get-profile',
            [],
            ['Authorization' => 'Bearer test-token']
        );
        
        $this->assertStringContainsString('profile', $output);
    }
    
    public function testFileUpload() {
        $manager = new FileAPI();
        
        $this->addFile('document', '/path/to/test.pdf');
        
        $output = $this->postRequest($manager, 'upload-file', [
            'description' => 'Test file'
        ]);
        
        $response = json_decode($output, true);
        $this->assertEquals('File uploaded', $response['message']);
    }
}
```

Available test methods:

| Method | Description |
|--------|-------------|
| `getRequest($manager, $service, $params, $headers)` | Simulate a GET request |
| `postRequest($manager, $service, $params, $headers)` | Simulate a POST request |
| `putRequest($manager, $service, $params, $headers)` | Simulate a PUT request |
| `deleteRequest($manager, $service, $params, $headers)` | Simulate a DELETE request |
| `addFile($name, $path)` | Add a file for upload testing |

## Calling Services

### Modern Approach: ServiceRouter (Recommended)

Use `ServiceRouter::discover()` to auto-register all services in a namespace:

``` php
use WebFiori\Framework\Router\ServiceRouter;
use WebFiori\Framework\Router\RouteOption;

class APIsRoutes {
    public static function create() {
        ServiceRouter::discover('App\\Apis', '/apis', [
            RouteOption::MIDDLEWARE => ['start-session', 'csrf']
        ]);
    }
}
```

This scans `App/Apis/` and registers a route per service:
- `#[RestController(name: 'orders', path: 'shop/orders')]` → `GET|POST /apis/shop/orders`
- `#[RestController('orders')]` → `GET|POST /apis/orders`
- `#[RestController]` on `ProductService` → `GET|POST /apis/product`
- `WebService` without attribute (getName() = 'users') → `GET|POST /apis/users`
- `WebServicesManager` subclass → registered as traditional manager route

**Recursive scanning** discovers nested directories:

``` php
ServiceRouter::discover('App\\Apis', '/apis', [], null, recursive: true);
// App/Apis/Admin/UserService.php → /apis/admin/user
// App/Apis/Auth/LoginService.php → /apis/auth/login
```

Directory names are converted to kebab-case (`UserAuth` → `user-auth`).

**Dynamic resolution** (no restart needed for new services):

``` php
ServiceRouter::dynamic('App\\Apis', '/apis/{controller}', [
    RouteOption::MIDDLEWARE => ['start-session']
]);
// Request to /apis/orders → resolves OrderService at runtime
```

**List discovered services:**

```bash
php webfiori services:list
```

### Traditional Approach: WebServicesManager

Services are called via HTTP with the `service` parameter:

```
GET https://example.com/api?service=get-users
POST https://example.com/api?service=create-user
```

Create a route to your services manager:

``` php
Router::api([
    'path' => '/api/{service}',
    'route-to' => UserAPI::class
]);
```

Then call: `https://example.com/api/get-users`

> **Note:** Both approaches work side by side. Existing `WebServicesManager` routes are unchanged.

## Related Articles

* [MVC Architecture](learn/mvc) - Build APIs with Controllers, Repositories, and Entities
* [Database Management](learn/database) - Use database in web services
* [Middleware](learn/middleware) - Protect API endpoints with middleware
* [Routing](learn/routing) - Create API routes
* [The Library WebFiori JSON](learn/webfiori-json) - Format API responses as JSON
* [The Class Response](learn/class-response) - Send API responses
