# Installation
In this page:

* **[Introduction](#introduction)**
* **[System Requirements](#system-requirements)**
* **[Local Development Environment Setup (Windows)](#local-development-environment-setup-windows)**
* **[Creating WebFiori Application](#creating-webFiori-application)**
* **[Running the Project](#running-the-roject)**


## Introduction

This domumentation explains the whole process from installing PHP and composer, to running your first WebFiori application. If you are familiar with how to install PHP on windows or you already have configured your environment, skip to the section "[Creating WebFiori Application](#creating-webFiori-application)".

## System Requirements

* **PHP:** Version 7.0 or later
* **PHP Extensions:**
    * mbstring
    * mysqli (for MySQL connections)
    * sqlsrv (for SQL Server connections)
    * fileinfo
    * json
* **Composer:** Dependency manager
* Integrated Development Environment (IDE) or text editor with debugging capabilities (e.g., [Apache NetBeans IDE](https://netbeans.apache.org), [VS Code](https://code.visualstudio.com/) or [PHPStorm](https://www.jetbrains.com/phpstorm/))

## Local Development Environment Setup (Windows):

### Installing PHP

**1. Download and Extract PHP Binaries:**

- Visit the official PHP download page for Windows: [https://windows.php.net/download](https://windows.php.net/download).
- Download the **non-thread-safe** version of PHP that matches your system requirements.
- Create a folder named `C:\PHP` on your local drive.
- Move the downloaded PHP binaries to the newly created folder.
- Extract the downloaded archive. 
- Rename the extracted folder to reflect the specific PHP version, such as `C:\PHP\8.3.3` (replace "8.3.3" with the actual version you downloaded).

**2. Add PHP to PATH Environment Variable:**

- Search for "environment variables" in the Windows Start menu and open "Edit the system environment variables".
- Under the "System variables" section, find the variable named "Path" and click "Edit".
- Click "New" and add the following path: `C:\PHP\8.3.3` (replace "8.3.3" with your actual PHP installation directory if it differs).
- Click "OK" on all open windows to save the changes.

**3. Configure PHP:**

- Open the folder `C:\PHP\8.3.3` (or your respective version folder).
- Rename the file `php-development.ini` to `php.ini`.
- Open the `php.ini` file in a text editor and enable the required extensions (e.g., mbstring, mysqli) by removing the semicolons (;) in front of their corresponding lines.

**4. Verify PHP Installation:**

- Open Command Prompt (CMD).
- Type the following command and press Enter: `php -v`
- If the command outputs information about the PHP version and configuration, the installation is successful.

-- Note: For production environments, a dedicated web server application like [Apache](https://cwiki.apache.org/confluence/display/httpd/PHP), [Nginx](https://www.nginx.com/resources/wiki/start/topics/examples/phpfcgi/), or [IIS](https://learn.microsoft.com/en-us/iis/application-frameworks/install-and-configure-php-applications-on-iis/using-fastcgi-to-host-php-applications-on-iis) is strongly recommended.

### Installing Composer

**1. Download Composer:**

Visit the official Composer download page: [https://getcomposer.org/download](https://getcomposer.org/download) and download the PHAR archive (executable file).

**2. Create a Script Folder and Update Path Environment Variable:**

- Create a folder named `C:\PHP\scripts` to store your Composer script.
- Add the `C:\PHP\scripts` folder to path environment variable. The approach to do it was explained in section 2 of how to install PHP.

**3. Move Composer Archive and Create Batch Script:**

- Move the downloaded Composer PHAR archive to the `C:\PHP\scripts` folder.
- Open a text editor like Notepad.
- Copy and paste the following code into the editor:

```
@echo off
php "%~dp0composer.phar" %*
```

- Save the file as "composer.bat" in the `C:\PHP\scripts` folder.

**4. Verify Installation:**

- Open a command prompt window.
- Type the following command and press Enter: `composer -V`

If Composer is installed correctly, the command should display the installed Composer version information. 

## Creating WebFiori Application

**Method 1: Composer (Recommended)**

1. **Verify Composer Installation:** Ensure that Composer is installed on your system.
2. **Open Command Prompt:** Navigate to the desired project directory using the command prompt (use `cd` command).
3. **Execute Installation Command:** Run the following command, replacing `my-site` with your preferred project folder name:

   ```
   composer create-project webfiori/app my-site --prefer-dist
   ```

**Method 2: Manual Installation**

1. **Download Framework:** Download the latest framework release from [https://webfiori.com/download](https://webfiori.com/download).
2. **Extract Files:** Extract the downloaded files to a sub-folder (e.g., `my-site`).
3. **Project Structure:**
   - Workspace: `my-site\app`
   - Document Root: `my-site\public`

## Running the Project


1. **Open Command Prompt:** Navigate to your project root directory (e.g., `my-site`) using the command prompt.
2. **Start Server:** Run the following command:

   ```
   php -S localhost:8080 -t public
   ```

This launches the server on port 8080. The application can be accessed in your web browser at `localhost:8080`.

**Customization of Application Folder Name:**

The application source code folder name can be modified by adjusting the `APP_DIR` constant located at the beginning of the `index.php` file.

**Important Note:**

- The built-in web server method is intended solely for testing purposes. Production environments require a dedicated web server application.


**Previous: [Introduction](learn/introduction)**

**Next: [Folder Structure](learn/folder-structure)**


