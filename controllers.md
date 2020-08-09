# Controllers

<meta name="description" content="Learn about controllers and why to use them.">

In this page:

[Introduction](#introduction)

## Introduction

Controllers are classes which are used to control something. The "something" might be data flow, access, sesstion and many. In WebFiori framework, controllers which extends the class [`Controller`](https://webfiori.com/docs/webfiori/logic/Controller) can be used in doing two things, data and sesstions. The most common use case is to use controllers in getting data from the database.

## The Class `Controller`

The class [`Controller`](https://webfiori.com/docs/webfiori/logic/Controller) is a concrete class which has utility methods that can be used to access data from the database. In addition to that, the class can be used to manage sesstions. A use case at which this class might be used is as follows:

Each controller is usually associated with an instance of the class [`MySQLQuery`](https://webfiori.com/docs/phMySql/MySQLQuery) which is used to build database queries. It is possible to use one controller with more that one `MySQLQuery` instance but it is not recommended as it might result in issues with the readability of the code.




