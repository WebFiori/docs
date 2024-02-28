# Installation
In this page:

* **System Requirements**
    * PHP Version
    * PHP Extensions
* **Recommended Environment**
* **Local Development Environment Setup (Windows)**
    * Obtaining PHP Binaries
    * Choosing a Web Server
* **Installation Options**
    * Method 1: Composer (Recommended)
    * Method 2: Manual Installation
* **Running the Project**
    * Method 1: Built-in Web Server (Testing Only)
    * Method 2: Web Server Application (Recommended)
* **Customization**
* **Important Note**



**System Requirements:**

* **PHP:** Version 7.0 or later
* **PHP Extensions:**
    * mbstring
    * mysqli (for MySQL connections)
    * sqlsrv (for SQL Server connections)
    * fileinfo
    * json
* **Composer:** Dependency manager

**Recommended Environment:**

* Integrated Development Environment (IDE) or text editor with debugging capabilities (e.g., Apache NetBeans IDE)

**Local Development Environment Setup (Windows):**

1. **Obtain PHP Binaries:**
   - Download the latest PHP binaries from the official website. Note that these include only the PHP interpreter, not additional software like MySQL or Apache.

2. **Choose a Web Server:**
   - For basic development and testing, the built-in PHP server is sufficient.
   - For production environments, a dedicated web server application like Apache, Nginx, or IIS is strongly recommended.

**Installation Options:**

**Method 1: Composer (Recommended)**

1. **Verify Composer Installation:** Ensure that Composer is installed on your system.
2. **Open Command Prompt:** Navigate to the desired project directory using the command prompt.
3. **Execute Installation Command:** Run the following command, replacing `my-site` with your preferred project name:

   ```
   composer create-project webfiori/app my-site --prefer-dist
   ```

**Method 2: Manual Installation**

1. **Download Framework:** Download the latest framework release from [https://webfiori.com/download](https://webfiori.com/download).
2. **Extract Files:** Extract the downloaded files to a sub-folder (e.g., `my-site`).
3. **Project Structure:**
   - Workspace: `my-site\app`
   - Document Root: `my-site\public`

**Running the Project:**

**Method 1: Built-in Web Server (Testing Only)**

1. **Open Command Prompt:** Navigate to your project root directory (e.g., `my-site`) using the command prompt.
2. **Start Server:** Run the following command:

   ```
   php -S localhost:8080 -t public
   ```

   - This launches the server on port 8080.
   - Access the application in your web browser at `localhost:8080`.

**Method 2: Web Server Application (Recommended)**

- Refer to the documentation of your chosen web server for specific configuration instructions to run the application.

**Customization:**

- The application source code folder name can be modified by adjusting the `APP_DIR` constant located at the beginning of the `index.php` file.

**Important Note:**

- The built-in web server method is intended solely for testing purposes. Production environments require a dedicated web server application.


**Previous: [Introduction](learn/introduction)**

**Next: [Folder Structure](learn/folder-structure)**


