# Database

<meta name="description" content="Learn how to use the database abstraction layer of the framework.">

In this page:
* [Introduction](#introduction)
* [The Idea](#the-idea)
* [Initializing your Database](#initializing-your-database)
* []()

## Introduction

One of the important features of any web application is to have a simple-unified interface at which the developer can use to access application database. WebFiori framework has an abstract layer that provides the developer with all needed tools to create databases and performs queries on them. Currently, the abstraction layer supportes only MySQL database but there are plans to support more in the future.

> **Note:** It is possible to connect to any database using PDO driver of PHP. The database layer helps in defining your database in easy way and also it helps in making the process of building SQL queries much simpler task.

## The Idea

Each table in your MySQL database is represented by the class [`MySQLTable`](https://webfiori.com/docs/webfiori/database/mysql/MySQLTable). Every table consist of columns. Every column is represented by the class [`MySQLColumn`](https://webfiori.com/docs/webfiori/database/mysql/MySQLColumn). Each table must be part of a schema (or database). The database is represented by the class [`Database`](https://webfiori.com/docs/webfiori/database/Database). WebFiori framework has the class [`DB`](https://webfiori.com/docs/webfiori/framework/DB) which adds extra functionality to the class [`Database`](https://webfiori.com/docs/webfiori/database/Database). The database instance is used to connect to MySQL database and run queries on it. 

## Initializing your Database

### Adding Connection Information
### Creating Database Tables
### Creating Database Class
