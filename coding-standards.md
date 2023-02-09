# Coding Standards of WebFiori Framework

<meta name="description" content="Here you will find some coding standards which where put in mind while bulding the code base of the framework.">

In this page:
* [Autoload](#autoload)
* [Coding Style](#coding-style)
* [Documentation Style](#documentation-style)

## Autoload
The framework has its own autoloader implementation which follows <a href="https://www.php-fig.org/psr/psr-4/" target="_blank">PSR-4</a> in almost all aspects of autoload. There are some differences which are:

* The autoloader will throw `ClassLoaderException` if a class was not found.
* It is recommended that a fully qualified class name to have the following form: `vendorName\subNamespaceNames\ClassName` (names uses camleCase).

The framework can be configured to not throw an exception. This can be performed during the process of initializing the autoloader <a href="https://webfiori.com/docs/webfiori/entity/AutoLoader" target="_blank">AutoLoader</a>.

## Coding Style
Recommended coding style is driven from PHP's recommended style and PSR, but it does not strictly follow it. For this reason, there are few differences. The rules are divided into two types, styles that must always be followed and recommended styles.

### Must Follow
* Always use `<?php` instead of `<?` for PHP tags.
* Namespaces must be all `lower\case`.
* Class constants and global constants names must be declared in `ALL_CAPS`.\
``` php
define('MY_CONST', 44);
const CLASS_CONSTANT = 44;
```
* Methods names, non-static class attributes must be declared in `camelCase`.
``` php
class Hello {
    private $fullName;
    private $email;
    public function helloWorld() {
    }
}
```
* Classed names, static attributes names must be declared in `PascalCase`.
``` php
class HelloWorldClass {
    public static $HelloString;
}
```
* An opening braces `{` must be on the same line for functions, method and class declaration.
* Never use `elseif`. Always use `else if`.
* Always include access modifiers for class methods, even if they are public.
* Do not use aliases for imported classes or class constants (e.g. `use webfiori\MyClass as XYZ`).
* Method names and attributes must be defined as follows, first include access modifier followed by `static` or/then `final`.
``` php
public static final function myFunc() {
}
```
* For comparison, always use strict comparison (`===` or `!==`) when comparing `bool` or `null`.
* Do not use the construct `empty()` to check for `null` values.

### Recommendations
* Always place PHP code in classes.
* Always specify method parameter's data type.
* Always specify method's return type.

## Documentation Style
The PHPDoc block must be similar to the following style:
``` 
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
