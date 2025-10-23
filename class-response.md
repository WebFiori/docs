
# The Class Response

<meta name="description" content="The class response is used to send back server output to the client.">

In this page:

* [Introduction](#introduction)
* [Collecting Server Output](#collecting-server-output)
* [Sending Output](#sending-the-output)
* [Sending Custom Headers](#sending-custom-headers)
* [Working With HTTP Cookies](#working-with-http-cookies)
* [Custom Response Code](#custom-response-code)
* [Clearing Output](#clearing-output)

## Introduction

While PHP offers `echo` and `print` for sending output, WebFiori's `WebFiori\Http\Response` class provides a centralized approach. It gathers server output for streamlined delivery to clients after request processing. Additionally, it allows for sending custom HTTP headers. This empowers developers to construct well-structured and efficient responses within their WebFiori applications.

The Response class buffers all output by default. Output is automatically sent to the client at the end of request processing, or can be sent immediately by calling `Response::send()`. 

> **Note**: This class is part of the library [`webfiori\http`](https://github.com/WebFiori/http).

## Collecting Server Output

Collecting server output can be archived using the static method [`Response::write()`](https://webfiori.com/docs/webfiori/http/Response#write). From its name, we can infer that it is used to write a string to response body.

> **Note:** In most cases, the developer will not have to collect the output him self. If the class [`WebPage`](https://webfiori.com/docs/webfiori/framework/ui/WebPage) is used for rendering HTML, no need to use it. This also applies to web services if the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/http/WebServicesManager) or [`ExtendedWebServicesManager`](https://webfiori.com/docs/webfiori/framework/ExtendedWebServicesManager) is used to manage your web APIs.

``` php
namespace App\Init\Routes;

use WebFiori\Http\Response;
use WebFiori\Framework\Router\Router;
use WebFiori\Framework\Router\RouteOption;

class ClosureRoutes {
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/closure',
            RouteOption::TO => function() {
                Response::write('Hello,<br/>');
                Response::write('Welcome to my website!<br/>');
                Response::write('Current Time is: '.date('H:i:s'));
            }
        ]);
    }
}
```

## Sending The Output

By default, the Response class buffers all output and sends it automatically at the end of request processing. However, it is possible to terminate code execution and send the response immediately by manually invoking the static method [`Response::send()`](https://webfiori.com/docs/webfiori/http/Response#send). This can be useful for debugging or when you need to send a response before the complete request processing is finished.

``` php
namespace App\Init\Routes;

use WebFiori\Http\Response;
use WebFiori\Framework\Router\Router;
use WebFiori\Framework\Router\RouteOption;

class ClosureRoutes {
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/closure',
            RouteOption::TO => function() {
                $myVar = 'Good';
                Response::write($myVar);
                $myVar = 'Hello';
                Response::write($myVar);
                
                // This will terminate code execution
                Response::send();
                
                // This will show the value of $myVar twice.
            }
        ]);
    }
}
```

## Sending Custom Headers

The developer can use the method [`Response::addHeader()`](https://webfiori.com/docs/webfiori/http/Response#addHeader) to add custom headers to the response. For example, we can send a JSON string and set its content type to `application/json`.

``` php
namespace App\Init\Routes;

use WebFiori\Http\Response;
use WebFiori\Framework\Router\Router;
use WebFiori\Framework\Router\RouteOption;

class ClosureRoutes {
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/closure',
            RouteOption::TO => function() {
                $jsonData = '{"username":"WarriorVx", "email":"my-email@example.com", "age":33}';
                Response::write($jsonData);
                Response::addHeader('content-type', 'application/json');
            }
        ]);
    }
}
```

### Common Headers

Modern web applications often require specific headers for security, CORS, and caching. Here are examples of commonly used headers:

**CORS Headers** - Enable cross-origin requests for APIs:
``` php
// Allow requests from any origin
Response::addHeader('Access-Control-Allow-Origin', '*');
// Or allow specific domain
Response::addHeader('Access-Control-Allow-Origin', 'https://example.com');

// Specify allowed HTTP methods
Response::addHeader('Access-Control-Allow-Methods', 'GET, POST, PUT, DELETE, OPTIONS');

// Specify allowed request headers
Response::addHeader('Access-Control-Allow-Headers', 'Content-Type, Authorization, X-Requested-With');
```

**Security Headers** - Protect against common attacks:
``` php
// Prevent page from being embedded in frames (clickjacking protection)
Response::addHeader('X-Frame-Options', 'DENY');

// Control resource loading to prevent XSS
Response::addHeader('Content-Security-Policy', "default-src 'self'");

// Prevent MIME type sniffing
Response::addHeader('X-Content-Type-Options', 'nosniff');

// Enable XSS filtering in browsers
Response::addHeader('X-XSS-Protection', '1; mode=block');
```

**Cache Control Headers** - Manage browser and proxy caching:
``` php
// Prevent caching for dynamic content
Response::addHeader('Cache-Control', 'no-cache, no-store, must-revalidate');

// Cache static resources for 1 hour
Response::addHeader('Cache-Control', 'public, max-age=3600');

// Set expiration date
Response::addHeader('Expires', gmdate('D, d M Y H:i:s', time() + 3600) . ' GMT');
```

**File Download Headers** - For file downloads and attachments:
``` php
// Force file download
Response::addHeader('Content-Disposition', 'attachment; filename="document.pdf"');

// Set proper content type for files
Response::addHeader('Content-Type', 'application/pdf');
Response::addHeader('Content-Length', filesize('/path/to/file.pdf'));
```

## Working With HTTP Cookies

In simple terms, [HTTP Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) are small piece of text which is set by the server for session management, personalizing content or tracking. Cookies are set by sending HTTP header [`set-cookie`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie). At minimum, a cookie must have a name and value.

Cookies have multiple attributes that must be set on order to have it valid like duration, path and others. Cookies in WebFiori are represented by the class [`HttpCookie`](https://webfiori.com/docs/webfiori/http/HttpCookie)

In order to send a new cookie, an object of type [`HttpCookie`](https://webfiori.com/docs/webfiori/http/HttpCookie) is created and then added to the response using the method [`Response::addCookie(HttpCookie)`](https://webfiori.com/docs/webfiori/http/Response#addCookie) as follows:

### Basic Cookie Example

``` php
namespace App\Init\Routes;

use WebFiori\Http\Response;
use WebFiori\Http\HttpCookie;
use WebFiori\Framework\Router\Router;
use WebFiori\Framework\Router\RouteOption;

class ClosureRoutes {
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/set-cookie',
            RouteOption::TO => function() {
                $cookie = new HttpCookie();
                $cookie->setName('user_preference');
                $cookie->setValue('dark_theme');
                $cookie->setPath('/');                    // Available site-wide
                $cookie->setExpires(time() + 86400);     // 24 hours
                $cookie->setDomain('example.com');       // Specific domain
                Response::addCookie($cookie);
                Response::write('Cookie set successfully!');
            }
        ]);
    }
}
```

### Security Best Practices

For secure applications, always configure cookies with appropriate security attributes:

``` php
// Secure session cookie
$sessionCookie = new HttpCookie();
$sessionCookie->setName('session_id');
$sessionCookie->setValue('abc123xyz789');
$sessionCookie->setIsHttpOnly(true);      // Prevent JavaScript access (XSS protection)
$sessionCookie->setIsSecure(true);        // HTTPS only transmission
$sessionCookie->setSameSite('Strict');    // CSRF protection
$sessionCookie->setPath('/');             // Available site-wide
$sessionCookie->setExpires(time() + 3600); // 1 hour expiration
Response::addCookie($sessionCookie);

// Authentication token cookie
$authCookie = new HttpCookie();
$authCookie->setName('auth_token');
$authCookie->setValue('secure_token_here');
$authCookie->setIsHttpOnly(true);         // Critical for auth cookies
$authCookie->setIsSecure(true);           // Must use HTTPS
$authCookie->setSameSite('Lax');          // Balance security and usability
$authCookie->setExpires(time() + 7200);   // 2 hours
Response::addCookie($authCookie);
```

### Practical Examples

**Setting a Persistent Cookie (remembers user for 30 days):**
``` php
$rememberCookie = new HttpCookie();
$rememberCookie->setName('remember_user');
$rememberCookie->setValue('user123');
$rememberCookie->setExpires(time() + (30 * 24 * 3600));  // 30 days
$rememberCookie->setIsHttpOnly(true);
$rememberCookie->setIsSecure(true);
Response::addCookie($rememberCookie);
```

**Deleting a Cookie:**
``` php
$deleteCookie = new HttpCookie();
$deleteCookie->setName('old_cookie');
$deleteCookie->setValue('');
$deleteCookie->setExpires(time() - 3600);     // Set to past date
$deleteCookie->setPath('/');                  // Must match original path
Response::addCookie($deleteCookie);
```

**Language Preference Cookie:**
``` php
$langCookie = new HttpCookie();
$langCookie->setName('user_language');
$langCookie->setValue('en');
$langCookie->setExpires(time() + (365 * 24 * 3600)); // 1 year
$langCookie->setPath('/');
$langCookie->setSameSite('Lax');               // Allow cross-site for language
Response::addCookie($langCookie);
```

### Cookie Security Guidelines

- **Always use `setIsHttpOnly(true)`** for authentication and session cookies to prevent XSS attacks
- **Use `setIsSecure(true)`** when serving over HTTPS to prevent transmission over unencrypted connections
- **Set `setSameSite()`** attribute:
  - `Strict`: Maximum protection, may break some legitimate cross-site functionality
  - `Lax`: Good balance, allows some cross-site requests (like following links)
  - `None`: Least restrictive, requires `setIsSecure(true)`
- **Set appropriate expiration** - short-lived for sensitive data, longer for preferences
- **Use specific paths and domains** to limit cookie scope when possible

## Custom Response Code

By default, the class `Response` will send any output with code `200 - Ok`. But it is possible to send the response using different response code. The method [`Response::setCode()`](https://webfiori.com/docs/webfiori/http/Response#setCode) can be used to achieve this.

``` php
namespace App\Init\Routes;

use WebFiori\Http\Response;
use WebFiori\Framework\Router\Router;
use WebFiori\Framework\Router\RouteOption;

class ClosureRoutes {
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/closure',
            RouteOption::TO => function() {
                Response::write('Resource Was Not Found!');
                Response::setCode(404);
            }
        ]);
    }
}
```

## Clearing Output

For some reason or another, the developer may want to clear the collected server output and replace it with another. Also, he might want to clear the headers. Clearing all output can be performed using the method [`Response::clear()`](https://webfiori.com/docs/webfiori/http/Response#clear). To clear response body only, the method [`Response::clearBody()`](https://webfiori.com/docs/webfiori/http/Response#clearBody) can be used. To clear headers only, the method [`Response::clearHeaders()`](https://webfiori.com/docs/webfiori/http/Response#clearHeaders) can be used.

``` php
namespace App\Init\Routes;

use WebFiori\Http\Response;
use WebFiori\Framework\Router\Router;
use WebFiori\Framework\Router\RouteOption;

class ClosureRoutes {
    /**
     * Create all closure routes. Include your own here.
     * @since 1.0
     */
    public static function create() {
        Router::closure([
            RouteOption::PATH => '/closure',
            RouteOption::TO => function() {
                Response::write('Resource Was Not Found!');
                Response::setCode(404);
                
                Response::clear();
                Response::write('Resource Was Found!');
                Response::setCode(200);
                //This will send back "Resource Was Found!"
            }
        ]);
    }
}
```


**Next: [UI Package](learn/ui-package)**

**Previous: [Routing](learn/routing)**
