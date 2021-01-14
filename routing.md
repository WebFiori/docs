# Routing

<meta name="description" contect="Routing in simple terms is the process of taking client request and sending it to correct location.">

In this page:
* [Introduction](#introduction)
* [The Definition of Routing](#the-definition-of-routing)
* [Life Cycle of a Request](#life-cycle-of-a-request)
* [The Class Router](#the-class-router)
* [Types of Routes](#types-of-routes)
  * [View Route](#view-route)
  * [API Route](#api-route)
  * [Closure Route](#closure-route)
  * [Custom Route](#custom-route)
  * [Class Route](#class-route)
* [Generic Routes](#generic-routes)
  * [URI Variable](#uri-variable)
  * [Using URI Variable](#using-uri-variable)
* [Customizing Routes](#customizing-routes)
  * [Sitemap](#sitemap)
  * [Case Sensitivity](#case-sensitivity)
  * [Languages](#languages)
  * [Variables Values](#variables-values)

## Introduction

One of the things that website owners care about is to have a friendly URLs that can be remembered easily. In addition to that, having a well defined URL structure can help in SEO. For example, instead of having something like `http://example.com/view-something?something={a-something}`, it is better to have something like `http://example.com/view-something/{a-something}`. In this example, something can be an image, user profile, a file or a simple HTML web page. Routing can help in achiving such as URL structure without having to create each individual resource.

## The Definition of Routing

One of the essential parts of the framework is routing. Routing in simple trems is sending user request to its correct destination. The class [`Router`](https://webfiori.com/docs/webfiori/framework/router/Router) is one of the core classes which are resposibile for performing this task. It can be used to create routes and send requests to correct resource.

After finishing the following set of topics, you will be able to understand how routing sub-system works and create your own custom URIs structure.

## Life Cycle of a Request

Every web server must have some kind of a configuration file. In Apache server, the configuration files have the name [`.htaccess`](https://httpd.apache.org/docs/current/howto/htaccess.html) and in IIS has the name `web.config`. Usually, the configuration files will get executed before the actual code.

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

Once the request reaches the file `index.php`, The initialization process of the framework will start. After the initialization is completed without any errors, the final stage is to route the request to its final destination.

The routing process is completed by the class [`Router`](https://webfiori.com/docs/webfiori/framework/router/Router). The whole magic of routing is completed by sending the requested URL as a parameter to the method [`Router::route()`](https://webfiori.com/docs/webfiori/framework/router/Router#route). If a resource was found at which the given URL is pointing to, the request will be sent to it. If no route is found, a 404 error is generated.

## The Class `Router`

The class [`Router`](https://webfiori.com/docs/webfiori/framework/router/Router) is one of the core framework classes. The main aim of this class is to direct client request to the correct resource. In addition to that, this class is used to create routes to different resources.

A resource can be simply a file such as a text file, an image or web page or a complex report that was generated dynamically by gathering data and representing it in a good looking way.

Most of the time, this class will be used to create routes but it can be used to perform other tasks as well. In general, there are 4 types of routes that can be created using this class:
* View Route.
* API Route.
* Closure Route.
* Custom Route.

For each type of route, there is a specific static method that can be used to create it. The 4 methods that corresponds to each type are:
* [Router::view()](https://webfiori.com/docs/webfiori/framework/router/Router#view)
* [Router::api()](https://webfiori.com/docs/webfiori/framework/router/Router#api)
* [Router::closure()](https://webfiori.com/docs/webfiori/framework/router/Router#closure)
* [Router::addRoute()](https://webfiori.com/docs/webfiori/framework/router/Router#addRoute)

## Types of Routes

We have said that there are 4 different types of routes. In general, the idea of creating route for each type will be the same. The only difference will be the location of the resource that the route will point to.

### View Route

This type of route is the most common type of routes. It is a route that will point to a web page. The page can be simple HTML page or dynamic PHP web page. Usually, the folder `app/pages` will contain all the views. The method [Router::view()](https://webfiori.com/docs/webfiori/framework/router/Router#view) is used to create such route.

In order to make it easy for developers, they can use the class [`ViewRoutes`](https://webfiori.com/docs/webfiori/framework/router/ViewRoutes) to create routes to all views. The developer can modify the body of the method [`ViewRoutes::create()`](https://webfiori.com/docs/webfiori/framework/router/ViewRoutes#create) to add new routes as needed.

Lets assume that we have 3 views inside the folder `app/pages` as follows:
* /pages/HomeView.html
* /pages/LoginView.php
* /pages/system-views/DashboardView.php

Also, Lets assume that the base URL of the website is `https://example.com/`. We want the user to see the pages as follows:
* `https://example.com/` should point to the view `HomeView.html`
* `https://example.com/home` should point to the view `HomeView.html`
* `https://example.com/user-login` should point to the view `LoginView.php`
* `https://example.com/dashboard` should point to the view `DashboardView.html`

The following sample code shows how to create such a URL structre using the class [`ViewRoutes`](https://webfiori.com/docs/webfiori/framework/router/ViewRoutes).
``` php
class ViewRoutes {
    public static function create(){
        Router::view([
            'path'=>'/', 
            'route-to'=>'/HomeView.html'
        ]);
        Router::view([
            'path'=>'/home', 
            'route-to'=>'/HomeView.html'
        });
        Router::view([
            'path'=>'/user-login', 
            'route-to'=>'/ThemeTestPage.php'
        ]);
        Router::view([
            'path'=>'/dashboard', 
            'route-to'=>'/system-views/DashboardView.php'
        ]);
    }
}
```
> Note: The forward slash is optional at the start of the path and the resource and can be ignored.
  
### API Route

An API route is a route that usually will point to a PHP class that exist in the folder `app/apis`. Usually the class will extend the class [`WebServicesManager`](https://webfiori.com/docs/webfiori/restEasy/WebServicesManager) or the class [`ExtendedWebServicesManager`](https://webfiori.com/docs/webfiori/framework/ExtendedWebServicesManager). To execute one of the services at which the class manages, we have to include an extra `GET` or `POST` parameter which has the name `service-name` or `service`.

Suppose that we have 3 services classes as follows:
* `apis/UserServices.php`
* `apis/ArticleServicess.php`
* `apis/ContentServicess.php`

Assuming that the base URL of the website is `https://example.com/`, We want the URLs of the APIs (or services) to be like that:
* `https://example.com/web-apis/user/add-user` should point to `apis/UserServices.php`
* `https://example.com/web-apis/user/update-user` should point to `apis/UserServices.php`
* `https://example.com/web-apis/user/delete-user` should point to `apis/UserServices.php`
* `https://example.com/web-apis/article/publish-article` should point `apis/writer/ArticleServicess.php`
* `https://example.com/web-apis/article/revert-publish` should point `apis/writer/ArticleServicess.php`
* `https://example.com/web-apis/article-content/add-content` should point to `/apis/writer/ContentServicess.php`
* `https://example.com/web-apis/article-content/remove-content` should point to the view `/apis/writer/ContentServicess.php`

One thing to note about creating APIs is that API name (or service name) must be passed alongside request body as a GET or POST parameter (e.g. `service=add-user` or `service-name=add-user`). as you can see from the above URLs, the name of the service is appended to the end of the URL. To let the router know that this is a route to a web service, we can use Generic Route.

A generic route is a route that has some of its path parts unkown. They can be used to serve dynamic content pased on generic path value. A generic path value is a value that is enclosed between `{}` while creating the route. For example, the first 3 APIs can have one URL in the form `https://example.com/web-apis/user/{service-name}` or `https://example.com/web-apis/user/{service}`.

The following sample code shows how to create such a URL structre using the class APIRoutes. You can see how API name is passed. Note that the value of the generic must be `action`, `service-name` or `service` or the API call will fail.
``` php 
class APIRoutes {
    public static function create(){
        Router::api([
            'path'=>'/web-apis/user/{action}', 
            'route-to'=>'/UserAPIs.php'
        ]);
        Router::api([
            'path'=>'/web-apis/article/{service-name}', 
            'route-to'=>'/writer/ArticleAPIs.php'
        ]);
        Router::api([
            'path'=>'/web-apis/article-content/{service-name}', 
            'route-to'=>'/writer/ContentAPIs.php'
        ]);
    }
}
```
### Closure Route

A closure route is a route to a user defined code that will be executed when the resource is requested. In other terms, it is a function that will be called when a request is made. It is simply an Anonymous Function.Suppose that we would like to create a route to a function that will output `Hello Mr.{A_Name}` When its called. The URL that will be requested will have the form `https://example.com/say-hello-to/{A_Name}`. As you can see, we have added the name as a generic path variable in the URL.

The value of the variable can be accessed using the method [`Router::getVarValue()`](https://webfiori.com/docs/webfiori/framework/router/Router#getVarValue). All what we have to do is to pass variable name and the method will return its value.The following code sample shows how its done.

``` php 
use webfiori\framework\Response;

class ClosureRoutes {
    public static function create(){
        Router::closure([
            'path'=>'/say-hello-to/{A_Name}', 
            'route-to'=>function(){
                $name = Router::getVarValue('A_Name');
                Response::apped('Hello Mr.'.$name);
            }
        ]);
    }
}
```

### Custom Route

A custom route is a route that points to a file which exist in a folder that was created by the developer. The folder must exist inside the framework scope in order to create a route to it. under this type of route, we can have APIs routes or views routes.

Let's assume that we have a folder which has the name `sys-files` and inside this folder, we have two folders. One has the name `views` which contains web pages and another one has the name `apis` which contains system's web APIs. Assuming that the `views` folder has a view which has the name `HomeView.php` and the folder `apis` has one file which has the name `AuthAPI.php`. The following code shows how to create a route to each file:
``` php
class ClosureRoutes {
    public static function create(){
        Router::addRoute([
            'path'=>'/index.php', 
            'route-to'=>'sys-files/views/HomeView.php'
        ]);
        Router::addRoute([
            'path'=>'/apis/auth/{action}', 
            'route-to'=>'sys-files/apis/AuthAPI.php',
            'as-api'=>true
        ]);
    }
}
```

### Class Route

It is possible to have a route to a PHP class. When such route is created, the router will try to create an instance of the class at which the route is pointing to.

``` php
use my\super\HomePage;

class ClosureRoutes {
    public static function create(){
        Router::addRoute([
            'path'=>'/rout-to-class', 
            'route-to'=> HomePage::class
        ]);
    }
}
```

## Generic Routes

Suppose that we have a we have a website that publishes news. Each post will have its own link. The posts has a URI structute that looks like `https://example.com/news/some_post`. One way to route the user to the correct post is to create a unique route for each post. If there are 1000 posts, then we have to create 1000 routes which is overwhelming.

WebFiori framework provides a way to create one route to all posts. This can be achived by using URI variables while creating your route. By adding variables, we created a generic route which can serve content based on the value of the variable.

### URI Variable

A URI Variable is a string which is a part of URI's path which can have many values. The value of the variable can be specified when sending a request to the resource that the URI represents. In the above example, the value of the variable most probably will be the name of the post. URI variables can be also used to replace query string variables to make URIs user friendly.

### Using URI Variable

We have already seen variables when we explained the diffrent types of routes ([Closure Route](#closure-route)). In order to add a variable to a route, we have to enclose its name between two curly braces `{}`. To access the value of the variable, the method [`Router::getVarValue()`](https://webfiori.com/docs/webfiori/framework/router/Router#getVarValue) is used. The following sample code shows how to use URI variables in creating generic routes.

``` php
use webfiori\framework\Response;

class ClosureRoutes {
    public static function create(){
        Router::closure([
            'path'=>'/news/{post-title}', 
            'route-to'=>function(){
                $name = Router::getVarValue('post-title');
                Response::append('You tried to open the post which has the title "'.$name.'"');
            }
        ]);
    }
}
```

## Customizing Routes

Each method which is used to add new route supports options which can be used to customize the route. In this section, we will have a look at the available options.

### Sitemap

The option `in-sitemap` is a boolean which is used to tell if the URI will be in the sitemap which is generated automatically or not. Default value of this option is `false`

> Note: To create a sitemap using added routes, the method [`Router::incSiteMapRoute()`](https://webfiori.com/webfiori/framework/router/Router#incSiteMapRoute). The sitemap will be accessable throght `http://example/sitemap.xml`.

### Case Sensitivity

One of the available options is `case-sensitive`. This option is used to make the URI case sensitive or not. If this one is set to false, then if a request is made to the URI `https://example.com/one/two`, It will be the same as requesting the URI `https://example.com/OnE/tWO`. By default, this one is set to true.

### Languages

The option `languages` is used to tell which languages the URI supports. The option accepts an array that contains language codes (such as `AR`). This one is used only when [generating sitemap](#generating-sitemap) of the website to tell search engines about the languages at which the page is available on.

### Variables Values

The option `vars-values` is used in generating the sitemap of the website. The option accepts sub-associative arrays. The key of the array represents the name of variable name and the value is a sub array that contains possible variable values.

### Middleware

The option `middleware` is used to specify which middleware the request will go through when a request is made to the given resource. This option accespts middleware name or an array that holds middleware names. Also, this option can have the name of middleware group or an array of middleware groups. For more information about middleware, [here](learn/middleware).

**Next: [The Class `Response`](learn/class-response)**

**Previous: [Basic Usage](learn/basic-usage)**

