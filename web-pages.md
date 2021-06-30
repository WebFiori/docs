# Web Pages

<meta name="description" content="One essential part of any website or web application are the pages that the user will interact with. WebFiori framework provides the developers with simple way to create web pages.">

## Introduction

One essential part of any website or web application are the pages that the user will interact with. WebFiori framework provides the developers with simple way to create dynamic web pages. Any class that represents a web page must extend the class [`WebPage`](https://webfiori.com/docs/webfiori/framework/ui/WebPage). This class has all needed utilities to add and modify the content of a web page without having to touch HTML.

## Simple Use Case

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

Now when we refersh the page on the web browser, the statement "Hello World!" should appear.

## Adding Meta Tags

The developer can use the method [`WebPage::addMeta()`](https://webfiori.com/docs/webfiori/framework/ui/WebPage#addMeta). This method accespts two parameters and one optional. The first argument is the value of the attribute `name` of the tag. The second one is the value of the attribute `content` of the tag. The last argument is used to tell if the value of the attribute `content` will be overreden if a tag with such `name` already exist.

``` php
namespace webfiori\examples\views;

use webfiori\framework\ui\WebPage;

class ExamplePage extends WebPage {
    public function __construct() {
        parent::__construct();
        
        $this->addMeta('robots', 'index, follow');
        
        $div = $this->insert('div');
        $div->text("Hello World!");
    }
}
```


## Adding CSS or JS Resource Files

Adding extra CSS or JS resource files to your page can be performed using two methods. The first one is [`WebPage::addJS()`](https://webfiori.com/docs/webfiori/framework/ui/WebPage#addJS). This method accepts two parameters. The first one is the value of the attribute `src` and the second one is an array that can have additional attributes for the JavaScript resource file such as `async`. 

The second method is [`WebPage::addCSS()`](https://webfiori.com/docs/webfiori/framework/ui/WebPage#addCSS). Similar to [`WebPage::addJS()`](https://webfiori.com/docs/webfiori/framework/ui/WebPage#addJS), this method accepts two parameters. The first one is the value of the attribute `href` and the second one is an array that can have additional attributes for the CSS resource file. 

``` php
namespace webfiori\examples\views;

use webfiori\framework\ui\WebPage;

class ExamplePage extends WebPage {
    public function __construct() {
        parent::__construct();
        
        $this->addJS('https://example.com/ext-javascript.js');
        
        $this->addCSS('https://example.com/ext-style.css');
        
        $div = $this->insert('div');
        $div->text("Hello World!");
    }
}
```

## Applying a Theme

One of the great fetaures of the framework is the ability to create and apply themes. The main aim of themes is to give a unified look and feel for all the pages of the website or web application. Simply, to change the whole look of the page without modifying the content, change one line of code to apply new style. For more information on how to create themes, [check here](learn/themes). 

There are two ways to apply a theme to a web page. One way is to use the name of the theme if you know its name and the other way is to use the class that represents the theme. One of the themes that comes with the framework has a class name `WebfioriV108`. To apply this theme, simply use the following syntax:
``` php
namespace webfiori\examples\views;

use webfiori\framework\ui\WebPage;

//First, import the theme.
use webfiori\theme\WebFioriV108;

class ExamplePage extends WebPage {
    public function __construct() {
        parent::__construct();
        
        //Apply the theme
        $this->setTheme(WebFioriV108::class);
        
        $div = $this->insert('div');
        $div->text("Hello World!");
    }
}
```

## Before Render Action

Suppose that we would like to perform an event before the page is rendered. For example, we would like to make an element be the last one to be added to the page. This can be achived using the method [`WebPage::addBeforeRender()`](https://webfiori.com/docs/webfiori/framework/ui/WebPage#addBeforeRender).

``` php
namespace webfiori\examples\views;

use webfiori\framework\ui\WebPage;

class ExamplePage extends WebPage {
    public function __construct() {
        parent::__construct();
        
        $this->addBeforeRender(function (WebPage $page, $name) {
            $div = $page->insert('div');
            $div->text($name);
        }, ["Ibrahim"]);
        
        $div = $this->insert('div');
        $div->text("Hello");
    }
}
```
**Next: [UI Package](learn/ui-package)**

**Previous: [The Class 'Response'](learn/class-response)**
