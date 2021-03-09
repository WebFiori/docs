# Web Pages

<meta name="description" content="One essential part of any website or web application are the pages that the user will interact with. WebFiori framework provides the developers with simple way to create web pages.">

# Introduction

One essential part of any website or web application are the pages that the user will interact with. WebFiori framework provides the developers with simple way to create web pages. Any class that represents a web page must extend the class [`WebPage`](https://webfiori.com/docs/webfiori/framework/ui/WebPage). This class has all needed utilities to add and modify the content of a web page without having to touch HTML.

# Simple Use Case

Assume that we would like to have a page that has only one statement which says "Hello World!". The first step is to create new PHP class that extends the class [`WebPage`](https://webfiori.com/docs/webfiori/framework/ui/WebPage).

``` php
namespace webfiori\examples\views;

use webfiori\framework\ui\WebPage;

class ExamplePage extends WebPage {
    public function __construct() {
        parent::__construct();
        
    }
}
```

Once we have the class, we can create a route for it inside the class [`ViewRoutes`](https://webfiori.com/docs/webfiori/framework/router/ViewRoutes) as follows:

``` php
namespace webfiori\framework\router;

use webfiori\examples\views\ExamplePage;

class ViewRoutes {
    /**
     * Create all views routes. Include your own here.
     * @since 1.0
     */
    public static function create() {
        Router::view([
            'path' => '/example', 
            'route-to' => ExamplePage::class
        ]);
    }
}
```

If we run the website using PHP's built-in server using the command `php -S localhost:8086 -t public` and navigate to `http://localhost:8086/example`, an empty page will appear. But when we inspect the HTML of the empty page, HTML code which looks like the following will be seen:

``` html
<!DOCTYPE html>
<html lang="EN">
    <head>
        <base href="http://localhost:8086">
        <title>
            WebFiori | WebFiori
        </title>
        <link rel="canonical" href="http://localhost:8086/example">
        <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    </head>
    <body itemscope itemtype="http://schema.org/WebPage">
        <div id="page-header">
        </div>
        <div id="page-body">
            <div id="side-content-area">
            </div>
            <div id="main-content-area">
            </div>
        </div>
        <div id="page-footer">
        </div>
    </body>
</html>
```

This structure is the default page structure which is used by WebFiori framework. The developer can modify this structure as needed but we will not look into that now. The last thing that we have to do is to add the text "Hello World!" to the body of our page. We will be creating new `div` element and add the text to that `div`.

``` php
namespace webfiori\examples\views;

use webfiori\framework\ui\WebPage;

class ExamplePage extends WebPage {
    public function __construct() {
        parent::__construct();
        
        $div = $this->insert('div');
        $div->text("Hello World!");
    }
}
```
