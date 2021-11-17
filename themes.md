# Themes
<meta name="description" content="Themes in webfiori framework works like a UI template which can change the whole look and feel of a web page. Learn how to use themes and create them.">

In this page:

* [Introduction](#introduction)
* [Using Themes](#using-themes)
* [Creating Custom Theme](#creating-custom-theme)
  * [Creating Theme Directory and Resources Folders](#creating-theme-directory-and-resources-folders)
  * [Adding Theme Resources](#adding-theme-resources)
  * [Implementing The Theme](#implementing-the-theme)
* [Using Command Line to Create Themes](#using-command-line-to-create-themes)

## Introduction

A web application or a website which provide useful content isn't good enough if it does not provide good and easy to use user interface. User interface is one of the important factors that can be used to mesure how good a website is. For that reason, WebFiori Framework provide the needed tools which can be used to create a custom unified user interface for your website or web application.

Themes in WebFiori Framework are used to create different custom user interfaces for your website or web application. In addition, they work like a plug-ins and can provide additional functionality. Themes can be found inside the folder `themes` of the framework.

## Using Themes

In order to apply a theme to your web page, we have to use the class [`WebPage`](https://webfiori.com/docs/webfiori/framework/ui/WebPag). For more information about web pages, [check here](learn/web-pages). After that, all what you need to know about the theme is its name or the class that represent the theme. The name of the theme acts as an identifier for it. If theme name is known, simply supply its name to the method [`WebPage::setTheme()`](https://webfiori.com/docs/webfiori/framework/ui/WebPage#setTheme) before rendering the page. For example, if theme name is `WebFiori Theme`, then the theme can be applied as follows:

``` php
use webfiori\framework\ui\WebPage;

class MyWebPage extends WebPage {
    public function __construct() {
        $this->setTheme('WebFiori Theme');
        
        // Add page content here
    }
}
```
Note that if the method [`WebPage::setTheme()`](https://webfiori.com/docs/webfiori/framework/ui/WebPage#setTheme) is called without supplying any parameters and no theme was loaded before, it will load the default theme which is set in the class [`AppConfig`](https://webfiori.com/docs/app/AppConfig).

In addition to loading a theme using its name, it is possible to use the class of the theme in order to apply it to a web page.


``` php
use webfiori\framework\ui\WebPage;
use app\theme\MyTheme;

class MyWebPage extends WebPage {
    public function __construct() {
        $this->setTheme(MyTheme::class);
        
        // Add page content here
        
    }
}
```

## Creating Custom Theme

Creating new theme is very simple. In general, theme creation process consist of the following:
* Creating theme folder (usually inside the folder `/themes`).
* Creating theme assets folder inside the folder `public/assets`.
* Creating new PHP class inside theme directory that extends the class [`Theme`](https://webfiori.com/docs/webfiori/framework/Theme).
* Implementing the abstract methods of the class.

In the next set of steps, we will show you how to create a very basic theme.

### Creating Theme Directory and Resources Folders

Themes in WebFiori Framework exist inside the folder `/themes` but it is possible to have a theme in another folder. Usually, the folder will hold theme components such as template HTML files or any PHP files that the theme depends on. The first step in creating new theme is to create new folder for it inside themes folder. let's assume that the name of the folder is `customTheme`. 

After creating theme folder, we have to create theme resources folder. Resources folder of the theme must exist inside the folder `public/assets` and must have the same name as theme folder. In this case, the directory of theme resources folder will be `public/assets/customTheme`. We also can create 3 additional optional folders inside the new folder. The 3 folders will be used to hold theme CSS, JavaScript and images. Let's give the names `js`, `css` and `images`. This means that the folder structure of the theme will be as follows:
* `/themes/customTheme` For theme components.
* `/public/assets/customTheme/css` For CSS files.
* `/public/assets/customTheme/js` For JavaScript files.
* `/public/assets/customTheme/images` For the images that the theme might use.

### Adding Theme Resources

At this step, we will create one CSS file. The file will be added in the folder `/public/assets/customTheme/css`. Let's give the file the name `theme.css` This CSS file will only contain selectors to give different colors for each section within the page. The code within the file will be something like the following:

``` css
#page-body{
    color: white;
    background-color: black;
    padding: 20px;
}
#page-header{
    background-color: blueviolet;
    padding: 20px;
}
#page-footer{
    background-color: royalblue;
    padding: 20px;
}
#main-content-area{
    background-color: violet;
    padding: 20px;
}
#side-content-area{
    background-color: honeydew;
    color:black;
    padding: 20px;
}
```
### Implementing The Theme

At this step, we will start by writing the code which will make the theme functional. At minimum level, we need to do the following for our theme:
* Set the name of the theme.
* Set the names of theme resource directories (CSS, JS and Images).
* Implementing the abstract methods of the class [`Theme`](https://webfiori.com/docs/webfiori/framework/Theme).

The name of the theme is needed because it acts like an identifier for it and is used to load it. The name can be set using the method [`Theme::setName()`](https://webfiori.com/docs/webfiori/framework/Theme#setName). Setting the names of theme resources folders will help in loading all JavaScript and CSS files automatically. For JavaScript files, there exist the method [`Theme::setJsDirName()`](https://webfiori.com/docs/webfiori/framework/Theme#setJsDirName), for CSS files, there exist the method [`Theme::setCssDirName()`](https://webfiori.com/docs/webfiori/framework/Theme#setCssDirName) and for images, there exist the method [`Theme::setImagesDirName()`](https://webfiori.com/docs/webfiori/framework/Theme#setImagesDirName). The last step is simply to define the actual structure of the page when the theme is loaded. 

Let's assume that the name of the class that represents the theme is `CustomTheme`. The file that will contain the code of the theme will be `themes/customTheme/CustomTheme.php`.

``` php
<?php
namespace themes\customTheme;

use webfiori\framework\Theme;

class CustomTheme extends Theme {
    public function __construct() {
        parent::__construct();
        $this->setName('Custom Theme');
        $this->setCssDirName('css');
        $this->setJsDirName('js');
        $this->setImagesDirName('images');
    }
}
```

> Note: If the names of your resources folders are `css`, `js` and `images`, you don't have to set them as they will be the default values.

After that, we will start the actual theme implementation. In this step, we have to implement 4 abstract methods which exist in the class [`Theme`](https://webfiori.com/docs/webfiori/framework/Theme). The 4 abstract methods are:
* [`Theme::getHeadNode()`](https://webfiori.com/docs/webfiori/framework/Theme#getHeadNode)
* [`Theme::getHeadrNode()`](https://webfiori.com/docs/webfiori/framework/Theme#getHeaderNode)
* [`Theme::getAsideNode()`](https://webfiori.com/docs/webfiori/framework/Theme#getAsideNode)
* [`Theme::getFooterNode()`](https://webfiori.com/docs/webfiori/framework/Theme#getFooterNode)

Each method of the mentioned must return an instance of [`HTMLNode`](https://webfiori.com/docs/webfiori/ui/HTMLNode) except for the method [`Theme::getHeadNode()`](https://webfiori.com/docs/webfiori/framework/Theme#getHeadNode). It must return an instance of [`HeadNode`](https://webfiori.com/docs/webfiori/ui/HeadNode). 
The fist method is used to define the tags which will be exist in the `<head>` tag. Usually, the tag will have meta tags, CSS and JS files included. The remaining 3 are used to define the layout of different parts of the page. In general, the page will have 3 main sections:
* Header section which usually contain main navigation menu.
* Aside section which usually contain extra navigation links or ads.
* Footer section which usually contains social media links.

For our theme, we will allow each method of the 3 to return a `<div>` element with a text that descripes which part of the page it represents. First, we need to import the class [`HTMLNode`](https://webfiori.com/docs/webfiori/ui/HTMLNode) and the class [`HeadNode`](https://webfiori.com/docs/webfiori/ui/HeadNode) since we are going to use them.

``` php 
<?php
namespace themes\customTheme;

use webfiori\framework\Theme;
use webfiori\ui\HTMLNode;
use webfiori\ui\HeadNode;

class CustomTheme extends Theme {
    public function __construct() {
        parent::__construct();
        $this->setName('Custom Theme');
        $this->setCssDirName('css');
        $this->setJsDirName('js');
        $this->setImagesDirName('images');
    }

    public function getAsideNode() {
        $aside = new HTMLNode();
        $aside->text('Aside Section');
        return $aside;
    }

    public function getFooterNode() {
        $footer = new HTMLNode();
        $footer->text('Footer Section');
        return $footer;
    }

    public function getHeadNode() {
        $head = new HeadNode();
        return $head;
    }

    public function getHeaderNode() {
        $header = new HTMLNode();
        $header->text('Header Section');
        return $header;
    }
}
```
Once this step is finished, our basic theme is ready. What we can do now is to test the look and feel of the theme. Using the `ExamplePage` which comes with the framework by default, we can apply the theme as follows:

``` php 
<?php

namespace webfiori\examples\views;

use webfiori\framework\ui\WebPage;

class ExamplePage extends WebPage {
    public function __construct() {
        $this->setTheme('Custom Theme');
        
        $this->setTitle($this->get('pages/sample-page/title'));
        $this->setDescription($this->get('pages/sample-page/description'));
        
        $mainContentArea = $this->getDocument()->getChildByID('main-content-area');
        
        //Load HTML component and insert it in the body of the page.
        $templateDir = ROOT_DIR.DS.'app'.DS.'pages'.DS.'example-template.html';
        $mainContentArea->component($templateDir, $this->get('pages/sample-page'));
        
    }
}
```

## Using Command Line to Create Themes

Instead of going through all theme creation steps manually, it is recommended to use command line interface to create themes using the command `create` of the framework. This command will create a basic theme skeleton where the developer can use to build his theme. 

### Steps
* Run the command `php webfiori create`.
* From the options, select the option `Theme`
* Provide theme class information (name, namespace and path)

Running this command will create 5 classes:
* Main theme class.
* A class with the name `HeadSection` which can be used to include theme resource files.
* A class with the name `HeaderSection` which represents header section of the page.
* A class with the name `AsideSection` which represents side section of the page.
* A class with the name `FooterSection` which represents footer section of the page.

The following shell output shows actuall run for creating a theme.

``` 
PS C:\Server\apache2\htdocs\app> php webfiori create
What would you like to create?
0: Database table class.
1: Entity class from table.
2: Web service.
3: Background job.
4: Middleware.
5: Database table from class.
6: CLI Command.
7: Theme.
8: Quit. <--
7
Enter a name for the new class:
SuperTheme
Enter an optional namespace for the class: Enter = "themes"
myOrganization\theme1
Where would you like to store the class? (must be a directory inside 'C:\Server\apache2\htdocs\app') Enter = "myOrganization\theme1"

Creating theme at "C:\Server\apache2\htdocs\app\myOrganization\theme1"...
Success: Created.
PS C:\Server\apache2\htdocs\app>
```


**Next: [Uploading Files](learn/uploading-files)**

**Previous: [UI Package](learn/ui-package)**

