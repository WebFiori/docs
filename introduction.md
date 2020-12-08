# Introduction

<meta name="description" content="What is WebFiori Framework? Why I should be using it? What is the deference between this framework and any other existing one?">

In this page:
* [What is WebFiori Framework](#what-is-webFiori-framework)
* [Features](#features)
  * [Simple Routing Engine](#simple-routing-engine)
  * [Sessions Management](#sessions-management)
  * []()
  * []()
  * []()

## What is WebFiori Framework

WebFiori framework is a fully object oriented web development framework which was built in top of PHP language. The framework comes with minimum features that a web developer might need to have a functional web application. Initially, the framework was developed as a generic template that can be used to only build static websites. But since then, it evolved into much more than a generic template.

In our opinion, web developer will need the followig at minimum level to have a working web application:
* A database (backend).
* Web APIs (or services) (backend).
* A way to create web pages and nodify the look and feel easly (frontend).

If we remove web pages, then the backend logic can be connected to any frontend such as a mobile application. 

## Features
The framework comes with many features that can help in the process of building web applications. The main features of the framework are:

### Simple Routing Engine

The main aim of [routing](learn/routing) is to take client request and sending it to correct resource. Routes in WebFiori Framework will point to a file in most cases. For example, it is possible to have a route which points to static HTML file or to have a route which points to PHP file that have some code to execute.

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
```

In addition to routing to files, it is possible to have routes which points to PHP code as a function called closure.


``` php
// https://example.com/run-code
Router::closure([
   'path' => 'run-code',
   'route-to' => function () {
      // TODO: Write php code
   }
]);
// https://example.com/
```

### Sessions Management

Sessions are used to keep client state when moving between different web pages. The frameworks provides the developer with a session manager which does not depend on the implementation of session manager of PHP. For this reason, some of the limitations which exist in PHP's session management system are no longer a problem. For example, it is possible to have more than one session at same time and each session will have its own state.

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

In addition to that, the developer can implement his own way for storing sessions by implementing custom storage engine using the interface `SessionStorage`.

### Theming

The framework gives the developer the option to use themes. The main use of themes is to give a unified look and feel in all web pages of the web application. Inside every class that represent a web page, the only thing that the developer must do to change the whole UI is to use its name.

```php
use webfiori\framework\Page;

class MyPage {
   __construct() {
       Page::theme('WebFiori V108');
       Page::render();
   }
}
return __NAMESPACE__;
```

### Basic Templating Engine
### Middleware
### Background Tasks
### Sending HTML Emails
### Command Line Interface
### Database Schema and Query Building
### Web Services

