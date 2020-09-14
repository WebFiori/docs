# WebFiori UI Package

<meta name="description" content="WebFiori has its own custom library which provides utilities to create custom user interface. The library called WebFiori UI Package.">

In this page:

* [Introduction](#introduction)
* [Main Classes](#main-classes)
  * [The Class `HTMLNode`](#the-class-htmlnode`)
  * [The Class `HTMLDoc`](#the-class-htmldoc`)
* [Using The Library](#using-the-library)
  * [Basic Usage](#basic-usage)
  * [Building More Complex DOM](#building-more-complex-dom)
  * [Adding Children](#adding-children)
* [Attributes](#attributes)
  * [No Value Attributes](#no-value-attributes)
  * [Setting Multiple Attributes](#setting-multiple-attributes)
  * [Specific Attributes Methods](#specific-attributes-methods)
* [Working With HTML Template Files](#working-with-html-template-files)
  * [Loading Template File](#loading-template-file)
  * [Slots](#slots)

## Introduction

One of the essential parts of any web application is to have an easy way to create the front end at which the users of the application will use. The front end of any web application will mostly consist of HTML, CSS and JavaScript. WebFiori frameworks gives the developers a package that contains a set of classes at which it can be used to build the DOM of a web page using PHP language without having to write HTML code. 

## Main Classes

All the classes which are related to the library can be found in the namespace [`webfiori\ui`](https://webfiori.com/docs/webfiori/ui).

### The Class `HTMLNode`

The library consist of many classes which represents different types of HTML elements. All classes are children of the class [`HTMLNode`](https://webfiori.com/docs/webfiori/ui/HTMLNode). This class basically can be used to represent any HTML or XML tag. The developer can extend this class and create his own custom HTML elements.

### The class `HTMLDoc`

This class represents HTML document. It can be used to build the DOM of the web page and modify it directly inside PHP. 

> Note: An instance of the class `HTMLDoc` is wrapped inside the class [`Page`](https://webfiori.com/docs/webfiori/entity/Page). It is always recomended to use the class `Page` to represent any web page.

## Using The Library

### Basic Usage
The very basic use case is to have HTML document with some text in its body. What we have to do is simply to create an instance of the class [`HTMLDoc`](https://webfiori.com/docs/webfiori/ui/HTMLDoc) and add a text to its body:
``` php
$doc = new HTMLDoc();
$doc->getBody()->text('Hello World!');
```

The structure of HTML document that be rendered will be similar to the following HTML code:
``` html
<!DOCTYPE html>
<html>
    <head>
        <title>
            Default
        </title>
        <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    </head>
    <body itemscope="" itemtype="http://schema.org/WebPage">
        Hello World!
    </body>
</html>
```
### Building More Complex DOM
To add more elements to the body of the document, the class <a href="https://webfiori.com/docs/webfiori/ui/HTMLNode">HMLNode</a> can be used to do that. It simply can be used to create any type of HTML element. The developer even can extend the class to create his own custom UI components. The library has already some pre-made components which are used in the next code sample. A list of the components can be found <a href="https://webfiori.com/docs/webfiori/ui">here</a>. The following code shows a code which is used to create a basic login form.

``` php

//Create new instance of "HTMLDoc".
$doc = new HTMLDoc();

// Build a login form.
$doc->getBody()->text('Login to System')
        ->hr()
        ->form(['method' => 'post', 'action' => 'https://example.com/login'])
        ->label('Username:')
        ->br()
        ->input('text', ['placeholder'=>'You can use your email address.', 'style' => 'width:250px'])
        ->br()
        ->label('Password:')
        ->br()
        ->input('password', ['placeholder' => 'Type in your password here.', 'style' => 'width:250px'])
        ->br()
        ->input('submit', ['value' => 'Login']);
```

The given PHP would be rendered to the following HTML code.

``` html
<!DOCTYPE html>
<html>
    <head>
        <title>
            Default
        </title>
        <meta name = "viewport" content = "width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    </head>
    <body itemscope = "" itemtype = "http://schema.org/WebPage">
        Login to System
        <hr>
        <form method = "post" action = "https://example.com/login">
        </form>
        <label>
            Username:
        </label>
        <br>
        <input type = "text" placeholder = "You can use your email address." style = "width:250px;">
        <br>
        <label>
            Password:
        </label>
        <br>
        <input type = "password" placeholder = "Type in your password here." style = "width:250px;">
        <br>
        <input type = "submit" value = "Login">
    </body>
</html>
```

### Adding Children

It is possible to add children to [`HTMLNode`](https://webfiori.com/docs/webfiori/ui/HTMLNode) instance using the method [`HTMLNode::addChild()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#addChild). Same method is also availble in the class [`HTMLDoc::addChild()`](https://webfiori.com/docs/webfiori/ui/HTMLDoc#addChild). The method accepts a string that represents the name of the node that will be added or an instance of the class `HTMLNode`

``` php
$doc = new HTMLDoc();

$superChild = new HTMLNode('section', ['id' => 'main-sec']);
$doc->addChild($superChild);

$superChild->addChild('h1', ['class' => 'section-heading'], false)
           ->text('Section Title');

$superChild->addChild(new HTMLNode('p'), [], false)
           ->text('This is a paragraph.');
```

The output of the code would be something similar to the following:

``` html
<!DOCTYPE html>
<html>
    <head>
        <title>
            Default
        </title>
        <meta name = "viewport" content = "width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    </head>
    <body itemscope itemtype = "http://schema.org/WebPage">
        <section id = "main-sec">
            <h1 class = "section-heading">
                Section Title
            </h1>
            <p>
                This is a paragraph.
            </p>
        </section>
    </body>
</html>
```

## Attributes

Each DOM node can have a set of attributes such as `style` or `class`. The class [`HTMLNode`](https://webfiori.com/docs/webfiori/ui/HTMLNode) has a generic method which can be used to set any attribute value which is the method [`HTMLNode::setAttribute()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#setAttribute). The method accepts two parameters, the first one is the value of the attribute and the second one is its value.

``` php 
$node = new HTMLNode();
$node->setAttribute('class','red-button');
```

### No Value Attributes

It is possible to have attributes without values (such as the attribute `disabled` or `itemscope`).

``` php
$node = new HTMLNode();
$node->setAttribute('class','red-button')
     ->setAttribute('itemscope')
     ->setAttribute('disabled');
```

This would render to the following HTML.

``` html
<div class = "red-button" itemscope disabled>
</div>
```

### Setting Multiple Attributes

If the developer would like to set the values for more than one attribute at one shot, he can use the method [`HTMLNode::setAttributes()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#setAttributes). The method accespts an array that contains the values of the attributes.

``` php
$node = new HTMLNode();
$node->setAttributes([
    'class' => 'red-button',
    'itemscope',
    'disabled',
    'onclick' => 'performAction()'
])->text("I'm a Red Button.");
```

HTML output:

``` html
<div class = "red-button" itemscope disabled onclick = "performAction()">
    I'm a Red Button.
</div>
```

### Specific Attributes Methods

In addition to the method [`HTMLNode::setAttribute()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#setAttribute), there are methods which are specific for setting different attributes of the node. This section lists the available methods.

* [`HTMLNode::setClassName()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#setClassName): Sets the attribute `class`.
* [`HTMLNode::setID()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#setID): Sets the attribute `id`.
* [`HTMLNode::setName()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#setName): Sets the attribute `name`.
* [`HTMLNode::setStyle()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#setStyle): Sets the attribute `style`.
* [`HTMLNode::setTabIndex()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#setTabIndex): Sets the attribute `tabindex`.
* [`HTMLNode::setTitle()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#setTitle): Sets the attribute `title`.
* [`HTMLNode::setWritingDir()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#setWritingDir): Sets the attribute `dir`.


## Working With HTML Template Files

One of the features of the library that it provides a very basic templating engine which only uses slots. It is possible to write HTML document in a separate HTML files and then combine files inside PHP to build the whole document. This can be helpful in making your code reusable.

### Loading Template File

Loading HTML files can be achived using the static method [`HTMLNode::loadComponent()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#loadComponent). Assuming that we have the following HTML code and we would like to load it into `HTMLNode` instance.

``` html
<section>
    <h1>Super Page</h1>
    <p>
        This is a paagraph inside my super page.
    </p>
</section>
```

The following PHP code can be used to load the file.

``` php
$component = HTMLNode::loadComponent('path/to/template/my-html-file.html');

//Now we can modify the component as needed.
$component->setClassName('main-section')
          ->paragraph('Another paragraph inside my super page.')
          ->comment('The element was modified inside PHP.')
          ->paragraph('Hello World!');
```

The rendered HTML of the given example will be similar to the following:


``` html
<section>
    <h1>Super Page</h1>
    <p>
        This is a paagraph inside my super page.
    </p>
    <p>
        Another paragraph inside my super page.
    </p>
    <!--The element was modified inside PHP.-->
    <p>
        Hello World!
    </p>
</section>
```

### Slots

Slots in simple terms are places in your code that can be filled with values. The values are usually dynamically generated or taken from a storage such as a database or files. Slots in HTML template files are represented using mustache syntax. Any string in your HTML code which is placed between `{{}}` is considered as slot. Slots can be placed in any place in your HTML code.

Assuming that we have the following HTML document which has slots:

``` html
<html>
    <head>
        <base href="{{base}}">
        <title>{{page-title}} | {{site-name}}</title>
    </head>
    <body>
        <p>
            Welcome {{visitor}} to my website!
        </p>
        <p>
            This page has slots!
        </p>
        <p>
            You visited this website at {{visit-time}}
        </p>
    </body>
</html>
```
In order to fill slots with values, we have to use the second argument of the method [`HTMLNode::loadComponent()`](https://webfiori.com/docs/webfiori/ui/HTMLNode#loadComponent). The second argument is an associative array that can have slots values.

The following code shows how to load the document into  [`HTMLDoc`](https://webfiori.com/docs/webfiori/ui/HTMLDoc) instance and fill the solts with values. 

``` php
$doc = HTMLNode::loadComponent('template.html', [
    'page-title' => 'Welcome to My Website',
    'site-name' => 'Awesome Website',
    'visitor' => 'Ibrahim',
    'visit-time' => date('Y-m-d H:i:s'),
    'base' => 'https://portal.example.com'
]);
```

The rendered document with filled slots would be similar to the following HTML code.

``` html
<!DOCTYPE html>
<html>
    <head>
        <base href = "https://portal.example.com">
        <title>
            Welcome to My Website | Awesome Website
        </title>
    </head>
    <body>
        <p>
            Welcome Ibrahim to my website!
        </p>
        <p>
            This page has slots!
        </p>
        <p>
            You visited this website at 2020-09-14 10:26:04
        </p>
    </body>
</html>
```
