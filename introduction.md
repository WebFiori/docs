# Introduction

<meta name="description" content="An introduction to WebFiori Framework and its core features.">

In this page:
* [What is WebFiori Framework](#what-is-webfiori-framework)
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
WebFiori is a developer-friendly framework built using PHP, a popular programming language, mostly for building websites and web applications. It provides the essential tools you need to get started building web applications, like:

* **Connecting to databases**: Store information like user data, products, or articles.
* **Building web services** (APIs): These are like mini-applications that handle very specific tasks behind the scenes.
* **Creating web pages**: These are the visual elements users see and interact with.

Think of WebFiori as a toolbox that has the essential tools you need to get started. Additionally, it can be used for more complex tasks, like sending emails or running tasks in the background.

While WebFiori focuses on these core functionalities, it empowers you to build feature-rich web applications without getting lost in complex details.


## Features

* [Simple Routing Engine](#simple-routing-engine)
* [Sessions Management](#sessions-management)
* [Theming](#theming)
* [Basic Template Engine](#basic-template-engine)
* [Middleware](#middleware)
* [Background Tasks](#background-tasks)
* [Sending Email Messages](#sending-email-messages)
* [Command Line Interface](#command-line-interface)
* [Database and Query Building](#database-schema-and-query-building)
* [Web Services](#web-services)

### Simple Routing Engine

Imagine you're building a website, and visitors come in wanting to see different things (like the "About Us" page or a product listing). Routing is like a map that tells the website where to find the right information for each visitor based on their request.

WebFiori's routing system offers a versatile approach compared to traditional MVC frameworks. Unlike traditional MVC routes that point to controller methods, WebFiori routes can target various resources:
* **Static Files**: This includes standard resources like images, HTML pages, text files, etc.
* **PHP Classes**: Routes can directly invoke specific classes for handling requests.
* **PHP Functions**: Closures can be defined and referenced as routes for quick and efficient handling of specific functionalities.
* **Class Methods (MVC)**: The traditional MVC, allowing routes to call specific methods within controllers.

WebFiori prioritizes developer freedom by not enforcing the use of specific classes or methods like some MVC frameworks. Developers have the flexibility to choose the most suitable approach for each route based on the desired functionality and code organization.

In essence, Routing system in WebFiori acts as a powerful and adaptable tool, allowing developers to construct the routes tailored to their specific needs without rigid structural constraints.

``` php
// https://example.com/products/board-games/Chess
Router::page([
   RouteOption::PATH => 'products/{category}/{sub-category}',
   RouteOption::TO => 'ViewProductsPage.php'
]);
// https://example.com/
Router::page([
   RouteOption::PATH => '/',
   RouteOption::TO => 'Home.html'
]);
Router::addRoute([
   RouteOption::PATH => '/class-route',
   RouteOption::TO => MyPHPClass::class
]);
//Optional to use MVC
Router::addRoute([
   RouteOption::PATH => '/api/add-user',
   RouteOption::TO => UsersController::class,
   RouteOption::REQUEST_METHODS => ['post', 'put'],
   RouteOption::ACTION => 'addUser'
]);
```

For more information about routing, [check here](learn/routing).

### Sessions Management

WebFiori implements a robust session management system independent of PHP's native session handling. This approach offers several advantages:

* **Improved flexibility**: Developers are not bound by the limitations inherent to PHP's session management.
* **Enhanced scalability**: WebFiori's session handling can accommodate scenarios requiring multiple concurrent sessions per user, exceeding the capabilities of the default sessions management system offered by PHP.
* **Streamlined development**: The separation from PHP's session management simplifies development by providing a consistent and well-defined interface for managing user state across web pages within the application.

In essence, WebFiori's independent session management empowers developers to create more scalable and adaptable web applications without the constraints imposed by the underlying PHP framework.

``` php
// Start new session
SessionsManager::start('first-session');

//Pause first session and start new with duration = 5 minutes
SessionsManager::start('another-session', [
    SessionOption::DURATION => 5
]);

//Resume the session 'first-session'
SessionsManager::start('first-session');
```

More information about sessions management [here](learn/sessions-management).

### Theming

WebFiori's themes system empowers developers to create visually cohesive and functionally robust web applications. This system offers several benefits:

* **Unified User Experience**: Themes ensure a consistent look and feel across all application pages, maintaining a professional and recognizable experience for users.
* **Streamlined Design Workflow**: Developers can effortlessly modify the application's entire user interface by simply switching themes within relevant web page classes. This reduces the need for repetitive code.
* **Functional Enhancement**: Themes extend beyond visual appeal, acting as modular components that can introduce additional functionalities to the application. This allows developers to seamlessly integrate new features without extensive custom coding, promoting code reuse and maintainability.

By leveraging WebFiori's theming system, developers can achieve design consistency, streamline development processes, and efficiently build feature-rich web applications, ultimately delivering a professional and user-friendly experience.

``` php
namespace app\pages;

use webfiori\framework\Page;
use themes\myTheme\MyThemeCore;

class MyPage extends WebPage {
   public function __construct() {
       $this->setTheme(MyThemeCore::class);
   }
}
```

For more information on how to use and create themes, [check here](learn/themes).

### Basic Template Engine

WebFiori's template engine prioritizes clarity and ease of use, offering a streamlined approach to web page creation. While it may not boast the extensive feature set of other options, it excels in:

* **Intuitive Syntax**: The engine leverages familiar HTML syntax, allowing developers to construct templates using established web development practices. This minimizes the learning curve and promotes rapid development.
* **Seamless Integration with PHP**: WebFiori's template engine seamlessly integrates with PHP code. Developers can effortlessly embed dynamic content within templates by using placeholders.


``` php
<!--This is the template-->
<div class="container">
 <p>Hello Mr. "<?= $name ?>"</p>
</div>
```

``` php
$template = HTMLNode::fromFile('path/to/my/template.php', [
    'name' => 'Ibrahim Ali'
]);
```

For more information on how to build web pages, [check here](learn/ui-package).

### Middleware

WebFiori's middleware system empowers developers to intercept and manipulate incoming requests before they reach the application's core. This unlocks various benefits:

* **Enhanced Security**: Implement robust request validation and tailor application responses for improved security.
* **Granular Control**: Create custom filters to manage application access and seamlessly integrate session management mechanisms.
* **Simplified Development**: Leverage the `webfiori/framework/middleware/AbstractMiddleware` class as a base for efficient custom middleware creation.

By placing custom middleware in the folder `[APP_DIR]/middleware` of the application, WebFiori automatically registers and integrates all middleware in that folder.

For more information on middleware, [check here](learn/middleware).

### Background Tasks

WebFiori's scheduler system empowers developers to automate background tasks, enhancing application efficiency. This functionality ensures seamless execution of essential processes, even in the absence of user interaction. The Key Benefits of the system includes:

* **Automated Workflows**: Schedule repetitive tasks to run at predefined intervals, eliminating manual intervention and fostering operational efficiency.
* **Enhanced Reliability**: Guarantee the timely execution of crucial processes regardless of user activity, ensuring data integrity and task completion.
* **Improved User Experience**: Offload time-consuming computations and resource-intensive operations to the background, maintaining smooth application performance and responsiveness.

By leveraging the `webfiori/framework/scheduler/AbstractTask` class as a foundation, developers can effortlessly create custom background tasks. Simply place your implemented task class within the designated `[APP_DIR]/tasks` folder, allowing WebFiori to automatically discover and register it for seamless integration.

For more information about creating background jobs, [check here](learn/background-tasks).

### Sending HTML Emails
WebFiori empowers developers to implement seamless and user-centric email communication within their web applications. This functionality simplifies the process of sending professional-looking HTML notifications, ensuring users remain informed and engaged. The main key advantages are:

* **Effortless Configuration**: A single configuration step establishes your SMTP connection details, allowing you to readily send emails throughout your application.
* **Intuitive Class Integration**: The class`webfiori/framework/EmailMessage` facilitates the creation and transmission of HTML email content.
* **Focus on User Experience**: WebFiori prioritizes developer experience by promoting the use of familiar HTML syntax for crafting compelling email content. This reduces the learning curve and empowers developers to focus on creating valuable user interactions without getting overwhelmed by intricate configuration details.


``` php
$message = new EmailMessage('no-reply');
$message->setSubject('This is a Test Email');
$message->addTo('user@example.com','Blog User');
$paragraph = $message->insert('p');
$paragraph->text('This is a welcome message.');

$this->insert(HTMLNode::fromFile('email-template.php'))

$message->send();
```


For more information about how to use mailing system, [check here](learn/sending-emails).

### Command Line Interface

WebFiori equips developers with a suite of built-in commands designed to expedite the development process. These commands can streamline common tasks, saving valuable time and effort.

Furthermore, WebFiori empowers developers to create custom commands tailored to their specific project needs. These custom commands seamlessly integrate into the framework, allowing for efficient execution and enhanced development workflows.

By leveraging both pre-built and custom commands, developers using WebFiori can significantly accelerate development cycles, fostering increased productivity and project efficiency.

For more information about this feature, [check here](learn/command-line-interface)

### Database Schema and Query Building

WebFiori streamlines database interaction for developers by offering built-in support for schema and query creation, specifically tailored for MySQL and SQL Server databases. This functionality fosters efficient development workflows and simplifies database management.

WebFiori's commitment to developer freedom extends to database management. While offering built-in functionalities, it remains adaptable to integrate seamlessly with alternative tools and libraries such as the use of native database drivers or the use of PDO (PHP Data Objects), empowering you to craft tailored solutions for your project demands.

For more information about this feature, [check here](learn/database).

### Web Services

Web services play a crucial role in establishing communication channels between various application components, including front-end and back-end logic. While commonly utilized to connect web pages with the back-end, they can be integrated with diverse other front-end systems which are hosted in the web.

WebFiori leverages the library [WebFiori HTTP](https://github.com/webfiori/http) to empower developers with robust web service functionalities. Implementing these services is straightforward, requiring developers to extend the pre-defined `webfiori\http\AbstractWebService` class. This approach promotes efficient development and fosters the creation of well-structured web services.

For more information on web services, [check here](/learn/web-services).

**Next: [Installation](learn/installation)**
