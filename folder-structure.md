# Folder Structure
<meta name="description" content="Learn about the folders at which the framework uses to keep your code and the content of each folder.">

In this page:

* **[Introduction](#introduction)**
* **[Directory Hierarchy](#directory-hierarchy)**
* **[Setting Custom Application Folder](#setting-custom-application-folder)**


## Introduction

This document outlines a comprehensive approach to organizing your WebFiori application files, promoting both maintainability and adherence to best practices.

## Directory Hierarchy

```
webfiori-project/
├── public/                   # Web-accessible directory
│   ├── index.php             # Application entry point
│   └── assets/               # Static assets (CSS, JS, images)
├── App/                      # Application source code (customizable name)
│   ├── Apis/                 # Web services controllers and API managers
│   ├── Commands/             # Custom CLI commands
│   ├── Config/               # Application configuration files
│   ├── Database/             # Database access and interaction classes
│   ├── Entity/               # Core system entity classes
│   ├── Init/                 # Initialization classes
│   │   └── Routes/           # Route definition classes
│   ├── Langs/                # Translation files for i18n
│   ├── Middleware/           # Request filtering middleware
│   ├── Pages/                # Page classes and HTML templates
│   ├── Storage/              # Application data (not web-accessible)
│   │   ├── Uploads/          # User uploads and generated content
│   │   ├── Sessions/         # Session data files
│   │   ├── Logs/             # Application log files
│   │   └── Cache/            # Cached data and temporary files
│   └── Tasks/                # Background jobs and scheduled tasks
├── tests/                    # Unit and integration tests
└── vendor/                   # Framework core and Composer dependencies
```

* **Public Directory (`public`):**
    * Entry point for all web requests and publicly accessible resources.
    * Contains JavaScript, CSS, images, and other static assets.
    * **Subfolder:**
        * `Assets`: Static assets including theme resources and media files.

* **Application Directory (`App`, customizable)**
    * Main application source code directory (name can be customized via `APP_DIR` constant).
    * Contains all business logic, configuration, and application-specific code.

**Subdirectories within the Application Directory**

* **`Apis`:** REST API controllers and web service endpoints.
* **`Commands`:** Custom CLI commands for application management.
* **`Config`:** Application configuration and settings files.
* **`Database`:** Database models, queries, and connection classes.
* **`Entity`:** Domain entities and business objects.
* **`Init`:** Application initialization and bootstrap classes.
    * **`Routes`:** URL routing definitions and route handlers.
* **`Langs`:** Language files for internationalization and localization.
* **`Middleware`:** HTTP middleware for request/response processing.
* **`Pages`:** Page controllers, views, and HTML templates.
* **`Tasks`:** Background jobs and scheduled task definitions.
* **`Storage`:** Private application data (not web-accessible).
    * **`Uploads`:** User-uploaded files and generated content.
    * **`Sessions`:** Session storage files.
    * **`Logs`:** Application logs and error tracking.
    * **`Cache`:** Temporary cached data for performance.

**Testing (`tests`)**

* Unit tests, integration tests, and test utilities for code quality assurance.

**Framework Core and Libraries (`vendor`)**

* This directory houses the core framework files and any third-party libraries installed using Composer.

**Security Note:**

* Only the `public` directory should be web-accessible (App Root). All other directories (`App`, `tests`, `vendor`) should be protected from direct web access, ensuring application security and preventing unauthorized access to sensitive files.

## Setting Custom Application Folder

* Customize the application folder name (default: `App`) to match your project's naming conventions or PSR-4 requirements.
* Configure the folder name by modifying the `APP_DIR` constant at the top of `public/index.php`.
* When changed, the framework automatically recreates the folder structure with your chosen name. For example, setting `APP_DIR` to `MyProject` creates configuration files at `MyProject/Config`.

This organized structure promotes maintainable code, efficient collaboration, and scalable application development.

## Related Articles

* [Installation](learn/installation) - Set up your WebFiori project
* [Basic Usage](learn/basic-usage) - Learn how to use the framework structure
* [Routing](learn/routing) - Understand where routes are defined
* [Web Pages](learn/web-pages) - Learn about the Pages directory
* [Web Services](learn/web-services) - Understand the APIs directory
