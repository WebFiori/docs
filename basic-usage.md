
# Basic Usage

The simplest way in using the framework is to create HTML web pages and add routes to each page. A route is simply a way to serve the correct requested resource. Pages in WebFiori Framework can be created inside the folder `app/pages`. It is possible to create pages in different place but leave this for later.

Lets assume that we have HTML page which hase the name `index.html`. Also, let's assume tha the page has the following code in it:

``` html
<!DOCTYPE html>
<html>
    <head>
        <title>Hello</title>
        <meta charset="UTF-8">
    </head>
    <body>
        <div>
            <p>
                Welcome to My Home Page
            </p>
            <p>
                This page is served using 
                <a href="https://webfiori.com/webfiori">WebFiori Framework</a>
            </p>
        </div>
    </body>
</html>
```
In order to view the page, we have to create a route for it. Assuming that this page is in the folder `app/pages`, we can use the method <a href="https://webfiori.com/docs/webfiori/entity/router/Router#view">`Router::view()`</a> to create its route. The class <a href="https://webfiori.com/docs/webfiori/entity/router/ViewRoutes">`ViewRoutes`</a> has one static method which the developer can use to create routes to views. The following code shows how the route is created.
``` php
namespace webfiori\entity\router;

class ViewRoutes {
    public static function create() {
        Router::view([
            'path' => '/home', 
            'route-to' => '/index.html'
        ]);
    }
}
```

Assuming that the base URL of the website is `https//example.com`, if we navigate to `https//example.com/home` the page that we have created should appear in the web browser. 
