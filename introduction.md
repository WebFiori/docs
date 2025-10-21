# Introduction

<meta name="description" content="An introduction to WebFiori Framework and its core features.">

In this page:
* [What is WebFiori Framework](#what-is-webfiori-framework)
  * [Core Philosophy](#core-philosophy)
  * [Real-World Use Cases](#real-world-use-cases)
  * [Performance Benefits](#performance-benefits)
  * [Key Capabilities](#key-capabilities)
  * [Why Choose WebFiori?](#why-choose-webfiori)
* [Features](#features)
  * [Simple Routing Engine](#simple-routing-engine)
  * [Sessions Management](#sessions-management)
  * [Theming](#theming)
  * [Basic Template Engine](#basic-template-engine)
  * [Middleware](#middleware)
  * [Background Tasks](#background-tasks)
  * [Sending HTML Emails](#sending-html-emails)
  * [Command Line Interface](#command-line-interface)
  * [Database Schema and Query Building](#database-schema-and-query-building)
  * [Web Services](#web-services)
* [Requirements](#requirements)
* [Quick Start](#quick-start)
* [Framework Architecture](#framework-architecture)
* [Getting Help](#getting-help)

## What is WebFiori Framework

WebFiori is a modern, developer-friendly PHP framework designed for building robust web applications and APIs. Built with PHP 8.1+ compatibility, it provides essential tools while maintaining simplicity and flexibility.

### Core Philosophy

* **Developer Freedom**: No rigid MVC constraints - choose your preferred architecture
* **Minimal Learning Curve**: Familiar syntax and intuitive APIs
* **Performance First**: Lightweight core with optimized autoloading and efficient routing
* **Security Built-in**: CSRF protection, input validation, and secure session handling

### Real-World Use Cases

* **E-commerce Platforms**: Product catalogs, shopping carts, payment processing
* **Content Management**: Blogs, news sites, documentation portals
* **Business Applications**: CRM systems, inventory management, reporting dashboards
* **API Services**: Mobile app backends, microservices, data integration
* **Educational Platforms**: Learning management systems, online courses

### Performance Benefits

* **Optimized Autoloading**: Fast class loading with PSR-4 compliance
* **Efficient Routing**: Direct method invocation without heavy middleware chains
* **Caching Support**: Built-in caching mechanisms for improved response times

### Key Capabilities

* **Database Operations**: Store and manage user data, products, content, etc... with MySQL/MSSQL support
* **API Development**: Build RESTful web services and microservices with built-in JSON handling
* **Dynamic Web Pages**: Create interactive user interfaces using object-oriented HTML generation
* **Background Processing**: Handle email sending, data processing, and scheduled tasks asynchronously
* **Email Communication**: Send HTML emails with attachments and template support

### Why Choose WebFiori?

**Compared to other frameworks**, WebFiori offers:
* **Flexibility**: No forced architectural patterns - use what fits your project
* **Simplicity**: Less boilerplate code, more focus on business logic
* **Completeness**: Built-in solutions for common web development needs
* **Modern PHP**: Takes advantage of PHP 8.1+ features and performance improvements 


WebFiori empowers developers to build everything from simple websites to complex applications without unnecessary complexity.


## Features

* [Simple Routing Engine](#simple-routing-engine)
* [Sessions Management](#sessions-management)
* [Theming](#theming)
* [Basic Template Engine](#basic-template-engine)
* [Middleware](#middleware)
* [Background Tasks](#background-tasks)
* [Sending Email Messages](#sending-email-messages)
* [Command Line Interface](#command-line-interface)
* [Database and Query Building](#database-schema-and-query-building)
* [Web Services](#web-services)

### Simple Routing Engine

WebFiori's routing system offers unprecedented flexibility compared to traditional MVC frameworks. Instead of forcing routes to point only to controller methods, WebFiori routes can target multiple resource types:

**Route Target Options:**
* **Static Files**: Images, HTML pages, CSS, JavaScript, documents
* **PHP Classes**: Direct class instantiation for object-oriented handling
* **Closures**: Inline functions for quick prototyping and simple logic
* **MVC Controllers**: Traditional controller methods when preferred

```php
// API endpoint
Router::addRoute([
    RouteOption::PATH => '/api/users/{id}',
    RouteOption::TO => UserController::class,
    RouteOption::ACTION => 'getUser',
    RouteOption::REQUEST_METHODS => ['GET']
]);

// File download
Router::closure([
    RouteOption::PATH => '/download/{file}',
    RouteOption::TO => function($file) {
        $file = new File("uploads/$file");
        $file->view();
    }
]);
```

This flexibility allows developers to choose the most appropriate approach for each route without architectural constraints.

``` php
//Assuming application folder name is "App"
namespace App\Init\Routes;

use WebFiori\Framework\Router\Router;
use WebFiori\Framework\Router\RouteOption;

class PagesRoutes{
    public static function init() {
        // https://example.com/products/board-games/Chess
        Router::page([
            RouteOption::PATH => 'products/{category}/{sub-category}',
            RouteOption::TO => 'ViewProductsPage.php'
        ]);
        // https://example.com/
        Router::page([
            RouteOption::PATH => '/',
            RouteOption::TO => 'Home.html'
        ]);
        Router::addRoute([
            RouteOption::PATH => '/class-route',
            RouteOption::TO => MyPHPClass::class
        ]);
        //Optional to use MVC
        Router::addRoute([
            RouteOption::PATH => '/api/add-user',
            RouteOption::TO => UsersController::class,
            RouteOption::REQUEST_METHODS => ['post', 'put'],
            RouteOption::ACTION => 'addUser'
        ]);
    }
}
```

For more information about routing, [check here](learn/routing).

### Sessions Management

WebFiori implements a robust session management system that surpasses PHP's native session handling with enterprise-grade features:

**Advanced Capabilities:**
* **Multiple Concurrent Sessions**: Support multiple active sessions per user
* **Custom Storage Engines**: File-based, database, or custom storage implementations
* **Enhanced Security**: Automatic session regeneration and hijacking protection
* **Flexible Duration Control**: Per-session timeout configuration

**Production Benefits:**
* **Scalability**: Handle high-traffic applications with distributed session storage
* **Security**: Built-in protection against session fixation and hijacking
* **Debugging**: Comprehensive session monitoring and logging capabilities

```php
// E-commerce example: separate sessions for cart and user preferences
SessionsManager::start('shopping-cart', ['duration' => 60]); // 1 hour
SessionsManager::set('items', $cartItems);

SessionsManager::start('user-preferences', ['duration' => 1440]); // 24 hours
SessionsManager::set('theme', 'dark-mode');
```


``` php
use WebFiori\Framework\Session\SessionsManager;

// Start new session
SessionsManager::start('first-session');

//Pause first session and start new with duration = 5 minutes
SessionsManager::start('another-session', [
    SessionOption::DURATION => 5
]);

//Resume the session 'first-session'
SessionsManager::start('first-session');
```

More information about sessions management [here](learn/sessions-management).

### Theming

WebFiori's themes system empowers developers to create visually cohesive web applications. This system offers several benefits:

* **Unified User Experience**: Themes ensure a consistent look and feel across all application pages, maintaining a professional and recognizable experience for users.
* **Streamlined Design Workflow**: Developers can effortlessly modify the application's entire user interface by simply switching themes within relevant web page classes. This reduces the need for repetitive code.
* **Functional Enhancement**: Themes extend beyond visual appeal, acting as modular components that can introduce additional functionalities to the application. This allows developers to seamlessly integrate new features without extensive custom coding, promoting code reuse and maintainability.

``` php
//Assuming application folder name is "App"
namespace App\Pages;

use WebFiori\Framework\Ui\WebPage;
use themes\myTheme\MyThemeCore;

class MyPage extends WebPage {
   public function __construct() {
       parent::__construct();
       $this->setTheme(MyThemeCore::class);
   }
}
```

For more information on how to use and create themes, [check here](learn/themes).

### Basic Template Engine

WebFiori's template engine prioritizes clarity and ease of use, offering a streamlined approach to web page creation. While it may not boast the extensive feature set of other template engines, it excels in:

* **Intuitive Syntax**: The engine leverages familiar HTML syntax, allowing developers to construct templates using established web development practices. This minimizes the learning curve and promotes rapid development.
* **Seamless Integration with PHP**: WebFiori's template engine seamlessly integrates with PHP code. Developers can effortlessly embed dynamic content within templates by using PHP variables as placeholders.


``` php

<!--This is the template-->
<div class="container">
<!--PHP variable as placeholder-->
 <p>Hello Mr. "<?= $name ?>"</p>
</div>
```

``` php
use WebFiori\Ui\HTMLNode;

$template = HTMLNode::fromFile(APP_PATH.'Pages/Components/Hello.php', [
    'name' => 'Ibrahim Ali'
]);
```

For more information on how to build web pages, [check here](learn/ui-package).

### Middleware

WebFiori's middleware system gives developers the option to intercept and manipulate incoming requests before they reach the application's core logic. The use of middleware has various benefits that includes:

* **Enhanced Security**: Implement robust request validation and tailor application responses.
* **Granular Control**: Create custom filters to manage application access and seamlessly integrate session management mechanisms.
* **Simplified Development**: Leverage the `WebFiori\Framework\Middleware\AbstractMiddleware` class as a base for efficient custom middleware creation.

By placing custom middleware in the folder `[APP_DIR]/Middleware` of the application, WebFiori automatically registers and integrates all middleware in that folder.

For more information on middleware, [check here](learn/middleware).

### Background Tasks

WebFiori's scheduler system allows developers to automate background tasks, enhancing application efficiency. This functionality ensures seamless execution of essential processes, even in the absence of user interaction. The key benefits of the system includes:

* **Automated Workflows**: Schedule repetitive tasks to run at predefined intervals, eliminating manual intervention and fostering operational efficiency.
* **Enhanced Reliability**: Guarantee the timely execution of crucial processes regardless of user activity, ensuring data integrity and task completion.
* **Improved User Experience**: Offload time-consuming computations and resource-intensive operations to the background, maintaining smooth application performance and responsiveness.

By leveraging the `WebFiori\Framework\Scheduler\AbstractTask` class as a foundation, developers can effortlessly create custom background tasks. Simply place your implemented task class within the designated `[APP_DIR]/tasks` folder, allowing WebFiori to automatically discover and register it for seamless integration.

For more information about creating background jobs, [check here](learn/background-tasks).

### Sending HTML Emails
WebFiori provides developers with the option to implement seamless and user-centric email communication within their web applications. This functionality simplifies the process of sending professional-looking HTML notifications, ensuring users remain informed and engaged. The main key advantages are:

* **Effortless Configuration**: A single configuration step establishes your SMTP connection details, allowing you to readily send emails throughout your application.
* **Intuitive Class Integration**: The class `WebFiori\Framework\EmailMessage` facilitates the creation and transmission of HTML email content.
* **Focus on User Experience**: WebFiori prioritizes developer experience by promoting the use of familiar HTML syntax for crafting compelling email content. This reduces the learning curve and empowers developers to focus on creating valuable user interactions without getting overwhelmed by intricate configuration details.


``` php

use WebFiori\Framework\EmailMessage;
use WebFiori\Ui\HTMLNode;

$message = new EmailMessage('no-reply');
$message->setSubject('This is a Test Email');
$message->addTo('user@example.com','Blog User');
$paragraph = $message->insert('p');
$paragraph->text('This is a welcome message.');

$message->insert(HTMLNode::fromFile('email-template.php'));

$message->send();
```


For more information about how to use mailing system, [check here](learn/sending-emails).

### Command Line Interface

WebFiori equips developers with a suite of built-in commands designed to expedite the development process. These commands can streamline common tasks, saving valuable time and effort.

Furthermore, WebFiori allows for custom commands creation which are tailored to specific project needs. These custom commands seamlessly integrate into the framework, allowing for efficient execution and enhanced development workflows.

By leveraging both pre-built and custom commands, developers using WebFiori can significantly accelerate development cycles, fostering increased productivity and project efficiency.

For more information about this feature, [check here](learn/command-line-interface)

### Database Schema and Query Building

WebFiori streamlines database interaction for developers by offering built-in support for schema and query creation, specifically tailored for MySQL and SQL Server databases. This functionality fosters efficient development workflows and simplifies database management.

WebFiori's commitment to developer freedom extends to database management. While offering built-in functionalities, it remains adaptable to integrate seamlessly with alternative tools and libraries such as the use of native database drivers or the use of PDO (PHP Data Objects), empowering you to craft tailored solutions for your project demands.

For more information about this feature, [check here](learn/database).

### Web Services

Web services play a crucial role in establishing communication channels between various application components, including front-end and back-end logic. While commonly utilized to connect web pages with the back-end, they can be integrated with diverse other front-end systems which are hosted in the web.

WebFiori leverages the library [WebFiori HTTP](https://github.com/webfiori/http) to empower developers with robust web service functionalities. Implementing these services is straightforward, requiring developers to extend the pre-defined `WebFiori\Http\AbstractWebService` class. This approach promotes efficient development and fosters the creation of well-structured web services.

For more information on web services, [check here](/learn/web-services).

## Requirements

* **PHP**: 8.1 or higher
* **Composer**: For dependency management
* **Web Server**: Apache, Nginx, or built-in PHP server

## Quick Start

```php
// 1. Install via Composer
composer create-project webfiori/app my-app

// 2. Create a simple page
namespace App\Pages;
use WebFiori\Framework\Ui\WebPage;

class HomePage extends WebPage {
    public function __construct() {
        parent::__construct();
        $this->insert('h1')->text('Hello WebFiori!');
    }
}

// 3. Add a route
Router::page([
    RouteOption::PATH => '/',
    RouteOption::TO => HomePage::class
]);
``

## Framework Architecture

WebFiori follows a modular architecture that promotes:

* **Component Independence**: Use only what you need
* **Extensibility**: Easy integration with third-party libraries
* **Testing Support**: Support for unit and integration testing 

## Getting Help

* **Documentation**: Comprehensive guides and API reference
* **GitHub**: [Source code and issue tracking](https://github.com/WebFiori/framework)

**Next: [Installation](learn/installation)**
