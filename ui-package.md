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
* [Working With HTML Template Files](#working-with-html-template-files)
  * [Loading Template File](#loading-template-file)
  * [Slots](#slots)

## Introduction
One of the essential parts of any web application is to have an easy way to create the front end at which the users of the application will use. The front end of any web application will mostly consist of HTML, CSS and JavaScript. WebFiori frameworks gives the developers a package that contains a set of classes at which it can be used to build the DOM of a web page using PHP language without having to write HTML code. 

## Main Classes

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
