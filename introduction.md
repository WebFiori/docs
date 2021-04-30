# Introduction

<meta name="description" content="What is WebFiori Framework? Why I should be using it? What is the deference between this framework and any other existing one?">

In this page:
* [What is WebFiori Framework](#what-is-webFiori-framework)
* [Quality Comes First, Documentation Second](#quality-comes-first-documentation-second)
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

WebFiori framework is a fully object oriented web development framework which was built in top of PHP language. The framework was developed to have the minimum features that a web developer might need to have a functional web application. Initially, the framework was developed as a generic template that can be used to build only static websites. But since then, it evolved into much more than a generic template.

In our opinion, web developer will need the following at minimum level to have a working web application:
* A database (backend).
* Web APIs (or services) (backend).
* A way to create web pages and modify the look and feel easly (frontend).

If we remove web pages, then the backend logic can be connected to any frontend such as a mobile application. WebFiori framework fullfil the 3 by providing a generic database layer, a library for creating web services which is fully integrated with the framework and theming. In addition to the given 3, a developer might need to send email notifications or run a background process to perform specific action. All of that is possible with WebFiori framework.

## Features
The framework comes with many features that can help in the process of building web applications. The main features of the framework are:

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

### Simple Routing Engine

The main aim of routing is to take client request and sending it to correct resource. Most traditional framework which are fully MVC will have routes which points to controller methods. Routes in WebFiori Framework will point to a file in most cases. For example, it is possible to have a route which points to static HTML file or to have a route which points to PHP file that have some code to execute. The developer can decide what he would like to use in the file instead of forcing him to use specific class or function. In addition to routing to files, it is possible to have routes which points to PHP code as a function called closure. Also, it is possible to have routes which points to PHP classes.

``` php
// https://example.com/products/board-games/Chess
Router::view([
   'path' => 'products/{category}/{sub-category}',
   'route-to' => 'ViewProductsPage.php'
]);
// https://example.com/
Router::view([
   'path' => '/',
   'route-to' => 'Home.html'
]);
Router::addRoute([
   'path' => '/class-route',
   'route-to' => MyPHPClass::class
]);
```

For more information about routing, [check here](learn/routing).

### Sessions Management

Sessions are used to keep client state when moving between different web pages in the web application. The frameworks provides the developer with a sessions manager which does not depend on the implementation of PHP's session manager. For this reason, some of the limitations which exist in PHP's session management system are no longer a problem. For example, it is possible to have more than one session at same time and each session will have its own state.

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

In addition to that, the developer can implement his own way for storing sessions by implementing custom storage engine using the interface [`SessionStorage`](https://webfiori.com/docs/webfiori/framework/session/SessionStorage). You can find more information about sessions managemene [here](learn/sessions-management).

### Theming

The framework gives the developer the option to use themes. The main use of themes is to give a unified look and feel in all web pages of the web application. Inside every class that represent a web page, the only thing that the developer must do to change the whole UI is to use its name. In addition to changing the whole user interface, themes can act as plugins that provide additional functionality to the web application.

```php
use webfiori\framework\Page;

class MyPage extends WebPage {
   __construct() {
       $this->setTheme('WebFiori V108');
   }
}
```

For more information on how to use and create themes, [check here](learn/themes).

### Basic Templating Engine

There are many template engines out there with many fancy features. WebFiori framework provides a very basic template engine which only supports slots. The developer can write his template using HTML and then fill the slots with values inside PHP.

``` html
<!--This is the template-->
<div class="container">
 <p>Hello Mr. "\{\{name\}\}"</p>
</div>
```

``` php
$template = HTMLNode::loadComponent('path/to/my/template.html', [
    'name' => 'Ibrahim Ali'
]);
```

For more information on how to build web pages UI, [check here](learn/ui-package).

### Middleware

Middleware is a way that developers use to filter requests before reaching the actual web application. They can be used to start sessions, validate request headers or reject a request. Middleware in WebFiori represented by the class [`AbstractMiddleware`](https://webfiori.com/docs/webfiori/framework/middleware/AbstractMiddleware). The developer have to extend this class to have his own middleware implemented. A class that represents middleware must be placed in the folder `app/middleware` to be auto registered.

For more information on middleware, [check here](learn/middleware).

### Background Tasks

Usually, the application might need to perform some tasks even if no one is using it. For example, there might be a process to cleanup temporary uploaded files or generate a report and send it by email. The developer can write such process using PHP code and have it to execute at specific time. This can be achived using Cron Sub-system. To implement custom background job, the developer can extend the class [`AbstractJob`](https://webfiori.com/docs/webfiori/framework/cron/AbstractJob). A class that represents a background job must be placed in the folder `app/jobs` to be auto registered.

For more information about creating background jobs, [check here](learn/background-tasks).

### Sending HTML Emails

Mostly, every web application out there will have to send email notifications for logins or registration. The framework has simplified the process of creating emails and sending them. The developer only have to configure SMTP connection information once and use the class [`EmailMessage`](https://webfiori.com/docs/webfiori/framework/mail/EmailMessage).

``` php
$message = new EmailMessage('no-reply');
$message->subject('This is a Test Email');
$message->addReceiver('Blog User','user@example.com');
$message->write('This is a welcome message.');
$message->send();
```


For more information about how to use mailing system, [check here](learn/sending-emails).

### Command Line Interface

The framework provides the developer with commands that he can use to speed up development process. In addition to that, the developer can write his own commands and run them using the framework. For more information about the CLI of the framework, [check here](learn/command-line-interface)

### Database Schema and Query Building

Currntly, the framework has support for schema and query building for MySQL database only but more are coming in the future. The developer does not have to use this feature if he would like to use another database library. For more information about this feature, [check here](learn/database).

### Web Services

Web services are usually used to connect the front-end with the back-end. In most cases, web services will be connected with a web page but they can be also connected to any front-end. The framework uses the library [WebFiori HTTP](https://webfiori.com/docs/webfiori/http) to have web services support. The developer can implement services by extending the class [`AbstractWebService`](https://webfiori.com/docs/webfiori/http/AbstractWebService).

For more information on web services, [check here](/learn/web-services).

**Next: [Installation](learn/installation)**
