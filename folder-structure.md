# Folder Structure
<meta name="description" content="Learn about the folders at which the framework uses to keep your code and the content of each folder.">

In this page:

* **[Introduction](#introduction)**
* **[Directory Hierarchy](#directory-hierarchy)**
* **[Setting Custom Application Folder](#setting-custom-application-folder)**


## Introduction

This document outlines a comprehensive approach to organizing your WebFiori application files, promoting both maintainability and adherence to best practices.

## Directory Hierarchy

* **Public Directory (`public`):**
    * This directory serves as the foundation for all incoming requests.
    * It accommodates publicly accessible resources like JavaScript, CSS, and images.
    * **Subfolder:**
        * `assets`: Houses all static assets utilized by the application, including theme resources.

* **Application Directory (`app`, customizable)**
    * By default, this directory is named `app`, but you have the flexibility to personalize it using the `APP_DIR` constant.
    * It serves as the central hub for your application's code and configuration files.

**Subdirectories within the Application Directory**

* **`apis`:** This directory houses web services controllers and service managers (also known as Web APIs).
* **`commands`:** This directory accommodates custom-made command-line interface (CLI) commands.
* **`config`:** This directory is responsible for storing application configuration files.
* **`database`:** This directory encompasses classes related to database access and interaction.
* **`entity`:** This directory houses classes representing the core entities of your system.
* **`ini`:** This directory accommodates initialization classes for various aspects of the application, including privileges, background jobs, CLI commands, autoloading directories, and middleware.
    * **`routes`:** This subdirectory houses classes responsible for defining different types of routes within your application.
* **`langs`:** This directory stores translation files for internationalization (i18n) support. You can create custom language classes here or extend existing ones.
* **`middleware`:** This directory houses classes implementing middleware functionality. Middleware provides a layer to filter requests before routing occurs.
* **`pages`:** This directory houses classes representing application pages and their components. This directory may also contain HTML template files (views).
* **`tasks`:** This directory houses background jobs, which are code snippets scheduled to execute at specific times.

**Framework Core and Libraries (`vendor`)**

* This directory houses the core framework files and any third-party libraries installed using Composer.

**Theming (`themes`):**

* This directory houses pre-built themes bundled with the framework, each in its own subfolder.
* Theme-related CSS and JavaScript files are located in `public/assets`.
* You can also create custom themes, storing their resources outside `public/assets` folder and leveraging a Content Delivery Network (CDN) to include them.

## Customizing the Application Folder Name

* The framework allows you to rename the application folder (`app`) to adhere to PSR-4 standards.
* Define the desired name for the application folder at the top of the `public/index.php` file using the `APP_DIR` constant.
* Changing this value will cause the framework to recreate the default folder structure with your chosen name. For example, changing `APP_DIR` to `coolApp` will re-create configuration files at `coolApp/config`.

By adhering to this structure, you can establish a well-organized and maintainable application, fostering efficient collaboration and project scalability.

**Previous: [Installation](learn/installation)**

**Next: [Basic Usage](learn/basic-usage)**
