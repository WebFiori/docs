# Routing

<meta name="description" content="Routing in simple terms is the process of taking client request and sending it to correct resource.">

In this page:
* [Introduction](#introduction)
* [The Definition of Routing](#the-definition-of-routing)
* [Life Cycle of a Request](#life-cycle-of-a-request)
* [The Class Router](#the-class-router)
* [Types of Routes](#types-of-routes)
  * [Page Route](#view-route)
  * [API Route](#api-route)
  * [Closure Route](#closure-route)
  * [Custom Route](#custom-route)
  * [Class Route](#class-route)
  * [Redirect Route](#redirect-route)
* [Generic Routes](#generic-routes)
  * [URI Variable](#uri-variable)
  * [Using URI Variable](#using-uri-variable)
* [Grouping Routes](#grouping-routes)
* [Customizing Routes](#customizing-routes)
  * [Sitemap](#sitemap)
  * [Case Sensitivity](#case-sensitivity)
  * [Languages](#languages)
  * [Variables Values](#variables-values)
  * [Middleware](#middleware)
  * [Request Method](#request-method)
  * [Action](#action)

## Introduction

One thing that website owners care about is to have a friendly URLs that can be remembered easily. In addition to that, having a well-defined URL structure can help in SEO. For example, instead of having something like `http://example.com/view-something?something={a-something}`, it is better to have something like `http://example.com/view-something/{a-something}`. In this example, something can be an image, user profile, a file or a simple HTML web page. Routing can help in achieving such URL structure without having to create each individual resource.

## The Definition of Routing

One essential part of the framework is routing. Routing in simple terms is sending user request to its correct destination. The class [`Router`](https://webfiori.com/docs/webfiori/framework/router/Router) is one of the core classes which are responsible for performing this task. It can be used to create routes and send requests to correct resources.

After finishing the following set of topics, the developer should be able to understand how routing sub-system works and create his own custom URIs structure.

## Life Cycle of a Request

Every web server must have some kind of configuration file. In Apache server, the configuration files have the name [`.htaccess`](https://httpd.apache.org/docs/current/howto/htaccess.html) and in IIS has the name `web.config`. Usually, the configuration files will get executed before the actual code.

WebFiori Framework uses a custom `.htaccess` file that has a rule to re-write the requested URL and redirect every request to the root the file `index.php`. The file can be found inside the directory `public`. If the content of the `.htaccess` file is viewed, the re-write rule will look like the following:
```
ReWriteRule ^(.*)$ index.php [L,QSA]
```

Also, the framework has its own custom `web.config` file that has a rule which is used to send requests to the file `index.php`. If the content of the file `web.config` is viewed, the redirect rule will look like the following:
``` xml
<rule name="Bloom The Seed" stopProcessing="true">
    <match url="^(.*)$" ignoreCase="false" />
    <conditions>
        <!--Send all trafic to framework seed and make your work bloom.-->
        <add input="{REQUEST_FILENAME}" matchType="IsFile" ignoreCase="false" negate="true" />
    </conditions>
    <action type="Rewrite" url="index.php" appendQueryString="true" />
</rule>
```

Once the request reaches the file `index.php`, initialization process of the application will start. After the initialization is completed without any errors, the final stage is to route the request to its final destination.

> <b>Note:</b> [`mod_rewrite`](https://httpd.apache.org/docs/2.4/mod/mod_rewrite.html) must be enabled on Apache to use Re-Write Rules. For IIS, [URL ReWrite](https://www.iis.net/downloads/microsoft/url-rewrite) must be installed.

The routing process is completed by the class [`Router`](https://webfiori.com/docs/webfiori/framework/router/Router). The whole magic of routing is completed by sending the requested URL as a parameter to the method [`Router::route()`](https://webfiori.com/docs/webfiori/framework/router/Router#route). If a resource was found at which the given URL is pointing to, the request will be sent to it. If no route is found, a 404 error is generated.

## The Class `Router`

The class [`Router`](https://webfiori.com/docs/webfiori/framework/router/Router) is one of the core framework classes. The main aim of this class is to create routes to resources and direct client request to them.

A resource can be a file such as a text file, an image or web page or a complex report that was generated dynamically by gathering data and representing it in a good-looking way.

In general, there are 4 types of routes that can be created using this class:
* Page Route.
* API Route.
* Closure Route.
* Custom Route.

For each type of route, there is a specific static method that can be used to create it. The 4 methods that corresponds to each type are:
* [Router::page()](https://webfiori.com/docs/webfiori/framework/router/Router#page)
* [Router::api()](https://webfiori.com/docs/webfiori/framework/router/Router#api)
* [Router::closure()](https://webfiori.com/docs/webfiori/framework/router/Router#closure)
* [Router::addRoute()](https://webfiori.com/docs/webfiori/framework/router/Router#addRoute)


## Types of Routes
In general, the idea of creating route for each type of routes will be the same. The only difference will be the location of the resource that the route will point to in addition to [middleware](learn/middleware) group that the route will be assigned to.

### Page Route

This type is a route that will point to a web page. The page can be simple HTML page or dynamic PHP web page. Usually, the folder `[APP_DIR]/pages` of the application will contain all pages. The method [Router::page()](https://webfiori.com/docs/webfiori/framework/router/Router#page) is used to create such route.

In order to make it easy for developers, they can use the class `[APP_DIR]/ini/routes/PagesRoutes` to create routes to all pages. The developer can modify the body of the method `PagesRoutes::create()` to add new routes as needed.

Assuming that there exist 3 pages inside the folder `[APP_DIR]/pages` as follows:

* `[APP_DIR]/pages/HomeView.html`
* `[APP_DIR]/pages/LoginView.php`
* `[APP_DIR]/pages/system-views/DashboardView.php`

Also, assuming that the base URL of the website is `https://example.com/`. Assuming that the developer would like from the user to see the pages as follows:
* `https://example.com/` should point to the view `HomeView.html`
* `https://example.com/home` should point to the view `HomeView.html`
* `https://example.com/user-login` should point to the class `LoginView.php`
* `https://example.com/dashboard` should point to the view `DashboardView.html`

The following sample code shows how to create such a URL structure using the class [`ViewRoutes`](https://webfiori.com/docs/app/ini/routes/ViewRoutes).
``` php
namespace app\ini\routes;

use webfiori\framework\router\Router;
use webfiori\framework\router\RouteOption;

class PagesRoutes {
    public static function create(){
        Router::page([
            RouteOption::PATH => '/',
            RouteOption::TO => '/HomeView.html'
        ]);
        Router::page([
            RouteOption::PATH => '/home',
            RouteOption::TO => '/HomeView.html'
        ]);
        Router::page([
            RouteOption::PATH => '/user-login',
            RouteOption::TO => \app\pages\LoginView::class
        ]);
        Router::page([
            RouteOption::PATH => '/dashboard',
            RouteOption::TO => '/system-views/DashboardView.html'
        ]);
    }
}
```

> Note: The forward slash is optional at the start of the path and the resource and can be ignored.
  
### API Route

An API route is a route that will point to a PHP class that exist in the folder `[APP_DIR]/apis`. Usually the class will extend the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/http/WebServicesManager) or the class [`ExtendedWebServicesManager`](https://webfiori.com/docs/webfiori/framework/ExtendedWebServicesManager). To execute one of the services at which the class manages, the developer have to include an extra `GET` or `POST` parameter which has the name `service-name` or `service`. More information about web services can be found [here](learn/web-services).

Suppose that there exist 3 services classes as follows:
* `[APP_DIR]/apis/UserServicesManager.php`
* `[APP_DIR]/apis/ArticleServicesManager.php`
* `[APP_DIR]/apis/ContentServicesManager.php`

Assuming that the base URL of the website is `https://example.com/`, Assuming that the developer would like to have the URLs of the APIs (or services) to be like that:
* `https://example.com/web-apis/user/add-user` should point to `UserServicesManager.php`
* `https://example.com/web-apis/user/update-user` should point to `UserServicesManager.php`
* `https://example.com/web-apis/user/delete-user` should point to `UserServicesManager.php`
* `https://example.com/web-apis/article/publish-article` should point `ArticleServicesManager.php`
* `https://example.com/web-apis/article/revert-publish` should point `ArticleServicesManager.php`
* `https://example.com/web-apis/article-content/add-content` should point to `ContentServicesManager.php`
* `https://example.com/web-apis/article-content/remove-content` should point to the view `ContentServicesManager.php`

One thing to note about creating APIs is that API name (or service name) must be passed alongside request body as a GET or POST parameter (e.g. `service=add-user` or `service-name=add-user`). As noticed from the above URLs, the name of the service is appended to the end of the URL. The router will know that this is a route to a web service if Generic Route is used.

A generic route is a route that has some of its path parts unknown. They can be used to serve dynamic content passed as path parameter. A path parameter is a value that is enclosed between `{}` while creating the route. For example, the first 3 APIs can have one URL in the form `https://example.com/web-apis/user/{service-name}` or `https://example.com/web-apis/user/{service}`.

The following sample code shows how to create such a URL structure using the class `APIRoutes`. Note that the value of path parameter must be `action`, `service-name` or `service` or the API call will fail.

``` php
namespace app\ini\routes;

use webfiori\framework\router\Router;
use webfiori\framework\router\RouteOption;
use ContentServices;

class APIRoutes {
    public static function create(){
        Router::api([
            RouteOption::PATH => '/web-apis/user/{service}',
            RouteOption::TO => '/UserAPIs.php'
        ]);
        Router::api([
            RouteOption::PATH => '/web-apis/article/{service}',
            RouteOption::TO => '/writer/ArticleAPIs.php'
        ]);
        Router::api([
            RouteOption::PATH => '/web-apis/article-content/{service}',
            RouteOption::TO => ContentServices::class
        ]);
    }
}
```
### Closure Route

A closure route is a route to a user defined code that will be executed when the resource is requested. In other terms, it is a function that will be called when a request is made. It is simply an Anonymous Function. Suppose that a developer would like to create a route to a function that will output `Hello Mr.{A_Name}` When it's called. The URL that will be requested will have the form `https://example.com/say-hello-to/{A_Name}`. As noticed,a generic path variable in the URL is added which will hold the name that the function will say hello to.

The value of the variable can be accessed using the method [`Router::getVarValue()`](https://webfiori.com/docs/webfiori/framework/router/Router#getVarValue). All what needed to be done is to pass variable name and the method will return its value. The following code sample shows how it's done.

``` php
namespace app\ini\routes;

use webfiori\framework\router\Router;
use webfiori\framework\router\RouteOption;
use webfiori\framework\Response;
 
class ClosureRoutes {
    public static function create(){
        Router::closure([
            RouteOption::PATH => '/say-hello-to/{A_Name}',
            RouteOption::TO => function(){
                $name = Router::getVarValue('A_Name');
                Response::apped('Hello Mr.'.$name);
            }
       ]);
    }
}
```

### Custom Route

A custom route is a route that points to a file which exist in a folder that was created by the developer outside the folder `apis` or `pages`. The folder must exist inside the framework scope in order to create a route to it.

Assuming that there exist a folder which has the name `sys-files` and inside this folder,there exist two folders. One has the name `views` which contains web pages and another one has the name `apis` which contains system's web APIs. Assuming that the `views` folder has a view which has the name `HomeView.php` and the folder `apis` has one file which has the name `AuthAPI.php`. The following code shows how to create a route to each file:
``` php
namespace app\ini\routes;

use webfiori\framework\router\Router;
use webfiori\framework\router\RouteOption;

class ClosureRoutes {
    public static function create(){
        Router::addRoute([
           RouteOption::PATH => '/index.php',
           RouteOption::TO => 'sys-files/views/HomeView.php'
       ]);
       Router::addRoute([
           RouteOption::PATH => '/apis/auth/{service}',
           RouteOption::TO => 'sys-files/apis/AuthAPI.php',
           RouteOption::API => true
       ]);
    }
}
```

### Class Route

It is possible to have a route to a PHP class. When such route is created, the router will try to create an instance of the class at which the route is pointing to.

``` php
namespace app\ini\routes;

use webfiori\framework\router\Router;
use webfiori\framework\router\RouteOption;
use my\super\HomePage;

class ClosureRoutes {
    public static function create(){
        Router::addRoute([
            RouteOption::PATH => '/rout-to-class',
            RouteOption::TO => HomePage::class
        ]);
    }
}
```

Also, it is possible to have the route point to specific method in the class by using the option `action`. This is useful when using MVC in building the application and want from the route to point to specific controller method.

``` php
use my\super\FrontController;

class ClosureRoutes {
    public static function create(){
        Router::addRoute([
           RouteOption::PATH => '/rout-to-class',
           RouteOption::TO => FrontController::class,
           RouteOption::ACTION => 'index'
       ]);
    }
}
```
### Redirect Route

This type of route is used to redirect specific request to another web page or website. This type of route can be added using the method [`Router::redirect()`](https://webfiori.com/docs/webfiori/framework/router/Router#redirect)



## Route Parameters

Suppose that there exist a website that publishes news. Each post will have its own link. Each post have a URI structure that looks like `https://example.com/news/some_post`. One way to route the user to the correct post is to create a unique route for each post. If there are 1000 posts, then developer have to create 1000 routes which is overwhelming.

WebFiori framework provides a way to create one route to all posts. This can be archived by using route parameters while creating your route. By adding parameters, the developer just  created a generic route which can serve content based on the value of the variable.

### URI Parameters

A URI parameter is a string which is a part of URI's path which can have many values. It is a string which is placed between `{}`. The value of the parameter can be specified when sending a request to the resource that the URI represents. In the following example, the value of the parameter most probably will be the name of the post. 

``` php
Router::closure([
    RouteOption::PATH => '/news/{post-title}', 
    RouteOption::TO => function(){}
]);
```

URI parameters can be also used to replace query string parameters to make URIs user-friendly.

### Using URI Parameters

In order to add a parameter to a route, the developer have to enclose the name of the parameter between two curly braces `{}`. To access the value of the parameter, the method [`Router::getParameterValue()`](https://webfiori.com/docs/webfiori/framework/router/Router#getParameterValue) is used. The following sample code shows how to use URI parameters in creating generic routes.

``` php
use webfiori\framework\Response;

class ClosureRoutes {
    public static function create(){
        Router::closure([
            RouteOption::PATH => '/news/{post-title}',
            RouteOption::TO => function(){
                $name = Router::getParameterValue('post-title');
                Response::append('You tried to open the post which has the title "'.$name.'"');
            }
        ]);
    }
}
```
### Optional URI Parameters

It is possible to have URI parameters as optional. In this case, the route will be resolved even if no value is provided in the path. To mark a parameter as optional, a question mark can be added to the end of parameter name as follows: `{var?}`.
``` php
use webfiori\framework\Response;

class ClosureRoutes {
    public static function create(){

        Router::closure([
            RouteOption::PATH => '/news/{post-title?}',
            RouteOption::TO => function () {
                $name = Router::getParameterValue('post-title');
                if ($name === null) {
                    Response::append('No title is provided.');
                } else {
                    Response::append('You tried to open the post which has the title "' . $name . '".');
                }
            }
        ]);

    }
}
```

## Grouping Routes

One of the features of the router is the ability to group routes which share same part of the path. Suppose that there are 3 pages with following links:
* `https://example.com/users`
* `https://example.com/users/view-user/{user-id}`
* `https://example.com/users/add-user`

One thing to notice about the routes is that they share the same `users` path part. It is possible to create each route by itself but there is a better way that can be used to group the routes. The following code shows how to group routes.
``` php
Router::group([
    RouteOption::PATH => '/users', 
    RouteOption::CASE_SENSITIVE => false,
    RouteOption::MIDDLEWARE => [
        'sample-middleware','sample-middleware-2'
    ],
    RouteOption::REQUEST_METHODS => 'get',
    RouteOptions::SUB_ROUTES => [
        [
            RouteOption::PATH => '/', 
            RouteOption::TO => ListUsersPage::class,
        ],
        [
            RouteOption::PATH => 'view-user/{user-id}', 
            RouteOption::TO => ViewUserPage::class,
        ]
        [
            RouteOption::PATH => 'add-user', 
            RouteOption::TO => AddUserPage::class,
        ]
    ]
]);
```
By grouping routes, the developer can achieve the following:
* Compact code
* Sub-routes will share same properties of parent path (middleware, request methods, languages and so on)


## Customizing Routes

Each method which is used to add new route supports options which can be used to customize the route. The options are kept as constants in the class `webfiori\framework\router\RouteOption`. This section explains each option in more details.

### Sitemap

The option `RouteOption::SITEMAP` is a boolean which is used to tell if the URI will be in the sitemap which is generated automatically or not. Default value of this option is `false`

> Note: To create a sitemap using added routes, the method [`Router::incSiteMapRoute()`](https://webfiori.com/webfiori/framework/router/Router#incSiteMapRoute). The sitemap will be accessible through `http://example/sitemap.xml`.

### Case Sensitivity

One of the available options is `RouteOption::CASE_SENSITIVE`. This option is used to make the URI case-sensitive or not. If this one is set to false, then if a request is made to the URI `https://example.com/one/two`, It will be the same as requesting the URI `https://example.com/OnE/tWO`. By default, this one is set to true.

### Languages

The option `RouteOption::LANGS` is used to tell which languages the URI supports. The option accepts an array that contains language codes (such as `AR`). This one is used only when [generating sitemap](#sitemap) of the website to tell search engines about the languages at which the page is available on.

### Variables Values

The option `RouteOption::VALUES` is used in generating the sitemap of the website. The option accepts sub-associative arrays. The key of the array represents the name of variable name and the value is a sub array that contains possible variable values.

### Middleware

The option `RouteOption::MIDDLEWARE` is used to specify which middleware the request will go through when a request is made to the given resource. This option accepts middleware name or an array that holds middleware names. Also, this option can have the name of middleware group or an array of middleware groups. For more information about middleware, [here](learn/middleware).

## Request Method

By default, a route to a resource can be called using any request method. But it is possible to restrict that to specific request method (or methods). The option `RouteOption::REQUEST_METHODS` can have a string which represents the method at which the resource can be fetched with (e.g. 'GET') or it can be an array that holds the names of request methods at which the resource can be fetched with.

## Action

The `RouteOption::ACTION` option is used when the route is pointing to a class and the developer would like to call one action from the class. This option can make your application more MVC-like.

**Next: [The Class `Response`](learn/class-response)**

**Previous: [Basic Usage](learn/basic-usage)**

