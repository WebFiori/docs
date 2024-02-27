# Introduction

<meta name="description" content="An introduction to WebFiori Framework and its core features.">

In this page:
* [What is WebFiori Framework](#what-is-webfiori-framework)
* [Features](#features)
  * [Simple Routing Engine](#simple-routing-engine)
  * [Sessions Management](#sessions-management)
  * [Theming](#theming)
  * [Basic Templating Engine](#basic-templating-engine)
  * [Middleware](#middleware)
  * [Background Tasks](#background-tasks)
  * [Sending HTML Emails](#sending-html-emails)
  * [Command Line Interface](#command-line-interface)
  * [Database Schema and Query Building](#database-schema-and-query-building)
  * [Web Services](#web-services)

## What is WebFiori Framework
WebFiori is a developer-friendly framework built using PHP, a popular programming language, mostly for building websites and web applications. It provides the essential tools you need to get started building web applications, like:

* **Connecting to databases**: Store information like user data, products, or articles.
* **Building web services** (APIs): These are like mini-applications that handle very specific tasks behind the scenes.
* **Creating web pages**: These are the visual elements users see and interact with.

Think of WebFiori as a toolbox that has the essential tools you need to get started. Additionally, it can be used for more complex tasks, like sending emails or running tasks in the background.

While WebFiori focuses on these core functionalities, it empowers you to build feature-rich web applications without getting lost in complex details.


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

Imagine you're building a website, and visitors come in wanting to see different things (like the "About Us" page or a product listing). Routing is like a map that tells the website where to find the right information for each visitor based on their request.

WebFiori's routing system offers a versatile approach compared to traditional MVC frameworks. Unlike traditional MVC routes that point to controller methods, WebFiori routes can target various resources:
* **Static Files**: This includes standard resources like images, HTML pages, text files, etc.
* **PHP Classes**: Routes can directly invoke specific classes for handling requests.
* **PHP Functions**: Closures can be defined and referenced as routes for quick and efficient handling of specific functionalities.
* **Class Methods (MVC)**: Traditional MVC structure is still supported, allowing routes to call specific methods within controllers.

WebFiori prioritizes developer freedom by not enforcing the use of specific classes or methods like some MVC frameworks. Developers have the flexibility to choose the most suitable approach for each route based on the desired functionality and code organization.

In essence, Routing system in WebFiori acts as a powerful and adaptable tool, allowing developers to construct the routes tailored to their specific needs without rigid structural constraints.

``` php
// https://example.com/products/board-games/Chess
Router::page([
   'path' => 'products/{category}/{sub-category}',
   'route-to' => 'ViewProductsPage.php'
]);
// https://example.com/
Router::page([
   'path' => '/',
   'route-to' => 'Home.html'
]);
Router::addRoute([
   'path' => '/class-route',
   'route-to' => MyPHPClass::class
]);
//Optional to use MVC
Router::addRoute([
   'path' => '/api/add-user',
   'route-to' => UsersController::class,
   'methods' => ['post', 'put'],
   'action' => 'addUser'
]);
```

For more information about routing, [check here](learn/routing).

### Sessions Management

Sessions are used to keep client state when moving between different web pages in the application. The frameworks provide developers with a sessions manager which does not depend on the implementation of PHP's session manager. For this reason, some of the limitations which exist in PHP's session management system are no longer a problem. For example, it is possible to have more than one session at same time and each session will have its own state.

``` php
// Start new session
SessionsManager::start('first-session');

//Pause first session and start new with duration = 5 minutes
SessionsManager::start('another-session', [
    'duration' => 5
]);

//Resume the session 'first-session'
SessionsManager::start('first-session');
```

In addition to that, developers can implement their own way for storing sessions by implementing custom storage engine using the interface [`SessionStorage`](https://webfiori.com/docs/webfiori/framework/session/SessionStorage). More information about sessions management [here](learn/sessions-management).

### Theming

The framework gives developers the option to use themes. The main use of themes is to have a unified look and feel in all pages of the application. Inside every class that represent a web page, the only thing that developers must do to change the whole UI is to use its name. In addition to changing the whole user interface, themes can act as plugins that provide additional functionality to the application.

``` php
namespace app\pages;

use webfiori\framework\Page;

class MyPage extends WebPage {
   public function __construct() {
       $this->setTheme('WebFiori V108');
   }
}
```

For more information on how to use and create themes, [check here](learn/themes).

### Basic Template Engine

There are many template engines out there with many fancy features. The framework has a basic template engine which supports slots and passing PHP variables. It is possible to write templates using HTML and then fill the slots with values inside PHP.

``` php
<!--This is the template-->
<div class="container">
 <p>Hello Mr. "<?= $name ?>"</p>
</div>
```

``` php
$template = HTMLNode::fromFile('path/to/my/template.php', [
    'name' => 'Ibrahim Ali'
]);
```

For more information on how to build web pages UI, [check here](learn/ui-package).

### Middleware

Middleware is a way that can be used to filter any request before reaching the actual application. They can be used to start sessions, validate request headers or reject a request. Middleware in WebFiori represented by the class [`AbstractMiddleware`](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware). To have a custom middleware, simply extend this class and implement its abstract methods. A class that represents middleware must be placed in the folder `[APP_DIR]/middleware` of the application to be auto-registered.

For more information on middleware, [check here](learn/middleware).

### Background Tasks

The application might need to perform tasks even if no one is using it. For example, there might be a process to clean up temporary uploaded files or generate a report and send it by email. It is possible to write such process using PHP and have it to execute at specific time with additional configuration using WebFiori. This can be archived using scheduler Sub-system. To implement custom background tasks, extend the class [`AbstractTask`](https://webfiori.com/docs/webfiori/framework/scheduler/AbstractTask). A class that represents a background process must be placed in the folder `[APP_DIR]/tasks` of the application to be auto-registered.

For more information about creating background jobs, [check here](learn/background-tasks).

### Sending HTML Emails

Probably every web application out there will have to send email notifications for logins, registration or other actions performed by end users. The framework has simplified the process of creating HTML emails and sending them. The developer only have to configure SMTP connection information once and use the class [`EmailMessage`](https://webfiori.com/docs/webfiori/framework/EmailMessage) to send nice looking HTML emails.

``` php
$message = new EmailMessage('no-reply');
$message->setSubject('This is a Test Email');
$message->addTo('user@example.com','Blog User');
$paragraph = $message->insert('p');
$paragraph->text('This is a welcome message.');

$this->insert(HTMLNode::fromFile('email-template.php'))

$message->send();
```


For more information about how to use mailing system, [check here](learn/sending-emails).

### Command Line Interface

The framework provide the developers with commands that can be used to speed up development process. In addition to that, it is possible to write custom commands and run them using the framework. For more information about this feature, [check here](learn/command-line-interface)

### Database Schema and Query Building

Currently, the framework has support for schema and query building for MySQL and SQL Server databases. Developer does not have to use this feature if he would like to use another database library or use PDO. For more information about this feature, [check here](learn/database).

### Web Services

Web services are usually used to connect the front-end with the back-end. In most cases, web services will be connected with a web page but they can be also connected to any front-end. The framework uses the library [WebFiori HTTP](https://webfiori.com/docs/webfiori/http) to have web services support. It is possible to implement services by extending the class [`AbstractWebService`](https://webfiori.com/docs/webfiori/http/AbstractWebService).

For more information on web services, [check here](/learn/web-services).

**Next: [Installation](learn/installation)**
