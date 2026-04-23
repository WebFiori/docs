# Coding Standards of WebFiori Framework

<meta name="description" content="Here you will find some coding standards which where put in mind while bulding the code base of the framework.">

In this page:
* [Autoload](#autoload)
* [Coding Style](#coding-style)
* [Documentation Style](#documentation-style)

## Autoload
The framework has its own autoloader implementation which follows <a href="https://www.php-fig.org/psr/psr-4/" target="_blank">PSR-4</a> in almost all aspects of autoload. There are some differences which are:

* The autoloader will throw `ClassLoaderException` if a class was not found.
* It is recommended that a fully qualified class name to have the following form: `VendorName\SubNamespaceNames\ClassName` (names uses camleCase).

The framework can be configured to not throw an exception. This can be performed during the process of initializing the autoloader <a href="https://webfiori.com/docs/WebFiori/Framework/Autoload/ClassLoader" target="_blank">ClassLoader</a>.

## Coding Style
Recommended coding style is driven from PHP's recommended style and PSR, but it does not strictly follow it. For this reason, there are few differences. The rules are divided into two types, styles that must always be followed and recommended styles.

### Must Follow
* Always use `<?php` instead of `<?` for PHP tags.
* In case of templates, use the short-hand `echo` command (e.g. `<?= 'A a string' ?>` instead of `<?php echo 'A a string' ?>`).
* Use 4 spaces for indentation (no tabs).
* Namespaces must be all `Pascal\Case`.
* Class constants and global constants names must be declared in `ALL_CAPS`.

``` php
// Correct
define('MY_CONST', 44);
const CLASS_CONSTANT = 44;

// Wrong
define('my_const', 44);
const classConstant = 44;
```

* Methods names, non-static class attributes must be declared in `camelCase`.

``` php
// Correct
class Hello {
    private $fullName;
    private $email;
    
    public function helloWorld() {
    }
}

// Wrong
class Hello {
    private $FullName;  // Should be camelCase
    private $Email;     // Should be camelCase
    
    function helloWorld() {  // Missing access modifier
    }
}
```
* Classed names, static attributes names must be declared in `PascalCase`.

``` php
// Correct
class HelloWorldClass {
    public static $HelloString;
}

// Wrong
class helloWorldClass {  // Should be PascalCase
    public static $helloString;  // Static attributes should be PascalCase
}
```

* An opening braces `{` must be on the same line for functions, method and class declaration.
* Never use `elseif`. Always use `else if`.
* Always include access modifiers for class methods, even if they are public.
* Do not use aliases for imported classes or class constants (e.g. `use WebFiori\MyClass as XYZ`).
* Method names and attributes must be defined as follows, first include access modifier followed by `static` or/then `final`.

``` php
// Correct
public static final function myFunc() {
}

// Wrong
static public final function myFunc() {  // Wrong order
final static function myFunc() {        // Missing access modifier
```

* For comparison, always use strict comparison (`===` or `!==`) when comparing `bool` or `null`.
* Do not use the construct `empty()` to check for `null` values.
* Use 'null coalescing operator' when needing to use a ternary in conjunction with `isset()`

``` php
// Correct
$nextLine = $traceEntry['line'] ?? 'X';

// Wrong
$nextLine = isset($traceEntry['line']) ? $traceEntry['line'] : 'X';
```

### Recommendations
* Always place PHP code in classes.
* Always specify method parameter's data type.
* Always specify method's return type.
* Limit line length to 120 characters.
* Use meaningful variable and method names that clearly express intent.
* Keep methods focused on a single responsibility.
* Prefer composition over inheritance when possible.

``` php
// Good example with type hints and clear naming
public function calculateTotalPrice(array $items): float {
    $total = 0.0;

    foreach ($items as $item) {
        $total += $item->getPrice();
    }
    
    return $total;
}
```

## Unit Tests Style
* The name of test classes should always follow this syntax:
``` php
class XXXTest {

}
```
In this context, the `XXX` should be replaced by entity name that test class represent. (e.g. `GenerateReportTest`). 

* Name of test methods should follow this syntax:
``` php
// Correct
public function testGetData() {
}

public function testGetDataWithInvalidInput() {
}

// Wrong  
public function testXXX00() {  // Avoid meaningless suffixes
}
```

* `@test` annotation should be included on top of every test method.

## Documentation Style
The PHPDoc block must be similar to the following style:

``` php
/**
 * SHORT_DESC.
 *
 * LONG_DESC.
 * 
 * @annotation1 DETAILS
 *
 * @annotation2 DETAILS
 */
```

Where:
* `SHORT_DESC` is a short description of the class or method. Must be ended by a dot.
* `LONG_DESC` is a big paragraph that contains more details.
* `@annotation1` and `@annotation2` are simple PHPDoc annotations such as `@return` or `@param`.

Notice that each part of the doc block must be followed by an empty space. This helps in improving readability.

## Related Articles

* [Introduction](learn/introduction) - Learn about WebFiori framework principles
* [Global Constants](learn/env-vars) - Follow naming conventions for constants
* [Basic Usage](learn/basic-usage) - Apply coding standards in practice
* [Folder Structure](learn/folder-structure) - Understand code organization standards
