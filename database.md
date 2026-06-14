# Database

<meta name="description" content="Learn how to use the database abstraction layer of the framework.">

In this page:
* [Introduction](#introduction)
* [Supported Databases](#supported-databases)
* [Connecting to a Database](#connecting-to-a-database)
  * [MySQL / MSSQL](#mysql--mssql)
  * [SQLite](#sqlite)
  * [Connection Pooling](#connection-pooling)
* [Creating Database Tables](#creating-database-tables)
  * [Using Blueprints](#using-blueprints)
  * [Using Table Classes](#using-table-classes)
  * [Using PHP 8 Attributes](#using-php-8-attributes)
  * [Registering Tables from Classes](#registering-tables-from-classes)
  * [Creating All Tables at Once](#creating-all-tables-at-once)
* [Creating Database Class](#creating-database-class)
* [Database Queries](#database-queries)
  * [Insert Record](#insert-record)
  * [Update Record](#update-record)
  * [Delete Record](#delete-record)
  * [Select](#select)
  * [Aggregate Functions](#aggregate-functions)
  * [Raw SQL Queries](#raw-sql-queries)
  * [Joins](#joins)
  * [Unions](#unions)
* [Transactions](#transactions)
* [Working With Result Set](#working-with-result-set)
  * [Retrieving Records](#retrieving-records)
  * [Mapping Records to Objects](#mapping-records-to-objects)
* [Dry-Run Mode](#dry-run-mode)
* [Performance Monitoring](#performance-monitoring)
* [Command Line Utilities](#command-line-utilities)

## Introduction

One of the important features of any web application is to have a simple-unified interface at which the developer can use to access application database. WebFiori framework has an abstract layer that provides the developer with all needed tools to create databases and perform queries on them.

> **Note:** It is possible to connect to any database using PDO driver of PHP. The database layer helps in defining your database in easy way and also it helps in making the process of building SQL queries much simpler task.

## Supported Databases

| Database | Driver | Table Class | Column Class |
|----------|--------|-------------|--------------|
| MySQL | `mysql` | `MySQLTable` | `MySQLColumn` |
| MSSQL (SQL Server) | `mssql` | `MSSQLTable` | `MSSQLColumn` |
| SQLite | `sqlite` | `SQLiteTable` | `SQLiteColumn` |

All table and column classes are in the `WebFiori\Database` namespace (sub-namespaces `MySql`, `MsSql`, `Sqlite`).

## Connecting to a Database

### MySQL / MSSQL

Database connections are represented by the class [`ConnectionInfo`](https://webfiori.com/docs/WebFiori/Database/ConnectionInfo). Connection information is stored in the JSON configuration file at `[APP_DIR]/Config/app-config.json`.

``` php
use WebFiori\Database\ConnectionInfo;
use WebFiori\Database\Database;

$connection = new ConnectionInfo('mysql', 'root', '123456', 'my_database', 'localhost', 3306);
$db = new Database($connection);
```

For MSSQL:

``` php
$connection = new ConnectionInfo('mssql', 'sa', 'password', 'my_database', 'localhost', 1433);
$db = new Database($connection);
```

There are two ways to add connection information to the framework: editing the `app-config.json` file directly, or using the CLI command `php webfiori add:db-connection` (recommended, since it validates the connection).

### SQLite

SQLite requires no username/password. Pass the file path as the database name:

``` php
use WebFiori\Database\ConnectionInfo;
use WebFiori\Database\Database;

// File-based database
$connection = new ConnectionInfo('sqlite', '', '', '/path/to/database.db');
$db = new Database($connection);

// In-memory database (useful for testing)
$connection = new ConnectionInfo('sqlite', '', '', ':memory:');
$db = new Database($connection);
```

SQLite uses the same query builder API as MySQL and MSSQL. Table blueprints and attribute-based tables work the same way — the library handles type mapping automatically (e.g., `DataType::INT` becomes `INTEGER`, `DataType::VARCHAR` becomes `TEXT`).

### Connection Pooling

The library includes a built-in connection pool that reuses idle connections instead of creating new ones on every request. This prevents "Too many connections" errors and reduces handshake overhead.

Connection pooling works automatically — every `Database` instance acquires connections from a shared `ConnectionPool` singleton:

``` php
use WebFiori\Database\ConnectionPool;

// Connections are pooled automatically. No special setup needed.
$db1 = new Database($connection);
// ... use $db1 ...
$db1->close(); // Released back to pool (not destroyed)

$db2 = new Database($connection); // Reuses the idle connection

// Configure pool limits
ConnectionPool::getInstance()->setMaxTotal(50);   // Max 50 active connections
ConnectionPool::getInstance()->setMaxPerKey(10);  // Max 10 idle per host/db combo

// Check pool status
echo ConnectionPool::getInstance()->getActiveCount();
echo ConnectionPool::getInstance()->getIdleCount();

// Close all connections (useful in tests or shutdown)
ConnectionPool::getInstance()->closeAll();
```

When a `Database` object is destroyed or `close()` is called, the connection is returned to the pool for reuse rather than being closed.

## Creating Database Tables

### Using Blueprints

For quick table creation without defining a separate class:

``` php
use WebFiori\Database\ColOption;
use WebFiori\Database\DataType;

$db->createBlueprint('users')->addColumns([
    'id' => [
        ColOption::TYPE => DataType::INT,
        ColOption::PRIMARY => true,
        ColOption::AUTO_INCREMENT => true
    ],
    'username' => [
        ColOption::TYPE => DataType::VARCHAR,
        ColOption::SIZE => 50
    ],
    'email' => [
        ColOption::TYPE => DataType::VARCHAR,
        ColOption::SIZE => 150
    ]
]);

$db->table('users')->createTable()->execute();
```

### Using Table Classes

Each table in the database can be represented as a class. For MySQL:

``` php
namespace App\Database;

use WebFiori\Database\MySql\MySQLTable;
use WebFiori\Database\ColOption;
use WebFiori\Database\DataType;

class ContactsTable extends MySQLTable {
    public function __construct() {
        parent::__construct('contacts');
        
        $this->addColumns([
            'id' => [
                ColOption::TYPE => DataType::INT,
                ColOption::SIZE => 11,
                ColOption::PRIMARY => true,
                ColOption::AUTO_INCREMENT => true
            ],
            'name' => [
                ColOption::TYPE => DataType::VARCHAR,
                ColOption::SIZE => 50,
                ColOption::NULL => false
            ],
            'email' => [
                ColOption::TYPE => DataType::VARCHAR,
                ColOption::SIZE => 255,
                ColOption::NULL => true
            ]
        ]);
    }
}
```

### Using PHP 8 Attributes

Define tables declaratively with attributes:

``` php
namespace App\Database;

use WebFiori\Database\Attributes\Column;
use WebFiori\Database\Attributes\ForeignKey;
use WebFiori\Database\Attributes\Table;
use WebFiori\Database\DataType;

#[Table(name: 'users')]
#[Column(name: 'id', type: DataType::INT, primary: true, autoIncrement: true)]
#[Column(name: 'username', type: DataType::VARCHAR, size: 50)]
#[Column(name: 'email', type: DataType::VARCHAR, size: 150)]
class UserTable {
}
```

Build and register:

``` php
use WebFiori\Database\Attributes\AttributeTableBuilder;

$table = AttributeTableBuilder::build(UserTable::class, 'mysql');
$db->addTable($table);
$db->table('users')->createTable()->execute();
```

You can define foreign keys using attributes:

``` php
#[Table(name: 'posts')]
class PostTable {
    #[Column(type: DataType::INT, primary: true, autoIncrement: true)]
    public int $id;

    #[Column(type: DataType::VARCHAR, size: 200)]
    public string $title;

    #[Column(name: 'author-id', type: DataType::INT)]
    #[ForeignKey(table: 'users', column: 'id')]
    public int $authorId;
}
```

### Registering Tables from Classes

Instead of manually building tables, register them directly from class names:

``` php
// Single class (works with attribute-based or Table subclasses)
$db->addTableFromClass(UserTable::class);

// Multiple classes at once
$db->addTablesFromClasses([UserTable::class, PostTable::class, CommentTable::class]);
```

If the class engine differs from the connection (e.g., a MySQL table class used with an SQLite connection), it is converted automatically.

### Creating All Tables at Once

Create all registered tables in dependency order (respects foreign keys):

``` php
$db->addTablesFromClasses([UserTable::class, PostTable::class]);
$db->createTables(); // Creates in correct order
```

## Creating Database Class

After creating tables, add them to a [`Database`](https://webfiori.com/docs/WebFiori/Database/Database) instance. In the framework, extend the class [`DB`](https://webfiori.com/docs/WebFiori/Framework/DB):

``` php
namespace App\Database;

use WebFiori\Framework\DB;

class TestingDatabase extends DB {
    public function __construct() {
        parent::__construct('connection-00');
        
        $this->addTable(new ContactsTable());
    }
}
```

The constructor accepts the name of the connection (as stored in `app-config.json`). You can also use `register()` to add multiple tables from the same namespace at once.

## Database Queries

The library provides a query builder for constructing queries. All query builders extend [`AbstractQuery`](https://webfiori.com/docs/WebFiori/Database/AbstractQuery).

### Insert Record

``` php
$db->table('contacts')->insert([
    'name' => 'Ibrahim BinAlshikh',
    'email' => 'xyz@example.com'
])->execute();
```

Insert multiple records:

``` php
$db->table('contacts')->insert([
    'cols' => ['name', 'email'],
    'values' => [
        ['Contact 1', '1@example.com'],
        ['Contact 2', '2@example.com'],
        ['Contact 3', '3@example.com']
    ]
])->execute();
```

Get the last inserted ID:

``` php
$db->table('contacts')->insert(['name' => 'New Contact'])->execute();
$id = $db->getLastInsertId();
```

### Update Record

``` php
$db->table('contacts')->update([
    'email' => 'new-email@example.com'
])->where('name', 'Contact 1')->execute();
```

### Delete Record

``` php
$db->table('contacts')->delete()->where('name', 'Contact 1')->execute();
```

### Select

``` php
// Select all
$result = $db->table('contacts')->select()->execute();

// Select specific columns
$result = $db->table('contacts')->select(['name', 'email'])->execute();

// Column aliases
$result = $db->table('contacts')->select([
    'name' => 'full_name',
    'email' => 'contact_email'
])->execute();
```

Where conditions:

``` php
$result = $db->table('contacts')->select()
    ->where('age', 15, '>')
    ->andWhere('name', 'Ibrahim')
    ->execute();

// OR condition
$result = $db->table('contacts')->select()
    ->where('email', null, 'is not')
    ->orWhere('mobile', null, 'is not')
    ->execute();
```

Ordering:

``` php
$result = $db->table('contacts')->select()
    ->orderBy(['name' => 'ASC', 'age' => 'DESC'])
    ->execute();
```

Pagination with limit/offset:

``` php
$result = $db->table('contacts')->select()
    ->limit(20)
    ->offset(40)
    ->execute();

// Or use the page helper
$result = $db->table('contacts')->select()
    ->page(3, 20) // Page 3, 20 items per page
    ->execute();
```

Grouping:

``` php
$result = $db->table('orders')->select(['status', 'count(*)' => 'total'])
    ->groupBy('status')
    ->execute();
```

### Aggregate Functions

Convenience methods for common aggregates:

``` php
// Count
$result = $db->table('contacts')->selectCount()->execute();
echo $result->fetch()['count'];

// Count specific column with alias
$result = $db->table('contacts')->selectCount('email', 'email_count')->execute();

// Max
$result = $db->table('products')->selectMax('price')->execute();
echo $result->fetch()['max'];

// Min
$result = $db->table('products')->selectMin('price')->execute();
echo $result->fetch()['min'];

// Average
$result = $db->table('products')->selectAvg('price')->execute();
echo $result->fetch()['avg'];
```

### Raw SQL Queries

For complex queries or database-specific features:

``` php
// Simple raw query
$result = $db->raw("SELECT * FROM contacts WHERE age > 25")->execute();

// With parameters (prevents SQL injection)
$result = $db->raw(
    "SELECT * FROM contacts WHERE age > ? AND name LIKE ?",
    [25, '%Ibrahim%']
)->execute();

// Complex queries
$result = $db->raw("
    SELECT c.name, COUNT(o.id) as order_count
    FROM contacts c
    LEFT JOIN orders o ON c.id = o.contact_id
    GROUP BY c.id
    HAVING order_count > ?
", [5])->execute();
```

### Joins

``` php
$db->table('contacts')->select([
    'contacts.name',
    'contacts.email',
    'orders.total'
])->join([
    'table' => 'orders',
    'on' => [
        'contacts.id' => 'orders.contact_id'
    ]
])->execute();
```

Specify join type:

``` php
$db->table('contacts')->select()
   ->join([
       'table' => 'orders',
       'type' => 'left',
       'on' => [
           'contacts.id' => 'orders.contact_id'
       ]
   ])->execute();
```

### Unions

``` php
$query1 = $db->table('contacts')->select(['name', 'email'])
             ->where('age', 25, '>');

$query2 = $db->table('subscribers')->select(['name', 'email'])
             ->where('active', true);

$query1->union($query2)->execute();
```

## Transactions

Transactions ensure that a group of operations either all succeed or all fail:

``` php
$db->transaction(function (Database $db) {
    $db->table('accounts')
       ->update(['balance' => 900])
       ->where('id', 1)
       ->execute();
    
    $db->table('accounts')
       ->update(['balance' => 1100])
       ->where('id', 2)
       ->execute();
    
    $db->table('transfers')->insert([
        'from_account' => 1,
        'to_account' => 2,
        'amount' => 100
    ])->execute();
});
```

- If the callback completes without exceptions, changes are committed
- If an exception is thrown, all changes are rolled back
- Nested transactions are supported via savepoints

## Working With Result Set

### Retrieving Records

The `execute()` method returns a [`ResultSet`](https://webfiori.com/docs/WebFiori/Database/ResultSet) for select queries:

``` php
$result = $db->table('contacts')->select()->execute();

// Iterate
foreach ($result as $record) {
    echo $record['name'];
}

// Row count and column names
echo $result->getRowsCount();
echo implode(', ', $result->getColsNames());

// Fetch methods
$firstRow = $result->fetch();       // Single row
$allRows = $result->fetchAll();     // All rows as array
```

### Mapping Records to Objects

Use `setMappingFunction()` to transform rows into objects:

``` php
$result = $db->table('contacts')->select()->execute();

$result->setMappingFunction(function ($dataset) {
    $retVal = [];
    foreach ($dataset as $record) {
        $contact = new Contact();
        $contact->setName($record['name']);
        $contact->setEmail($record['email']);
        $retVal[] = $contact;
    }
    return $retVal;
});

foreach ($result as $contact) {
    echo $contact->getName();
}
```

> **Tip:** For a more structured approach to entity mapping, see the [Repository Pattern](database-repository.md) which provides built-in CRUD, pagination, and eager loading.

## Dry-Run Mode

Preview what SQL would be generated without executing it:

``` php
$db->setDryRun(true);

$db->table('users')->insert(['name' => 'Test'])->execute(); // Not executed
$db->table('users')->select()->where('age', 25, '>')->execute(); // Not executed

// Get all captured queries
$queries = $db->getCapturedQueries();
foreach ($queries as $sql) {
    echo $sql . "\n";
}
```

This is useful for testing, debugging, and previewing migrations.

## Performance Monitoring

Track and analyze query performance:

``` php
use WebFiori\Database\Performance\PerformanceOption;

$db->setPerformanceConfig([
    PerformanceOption::ENABLED => true,
    PerformanceOption::SLOW_QUERY_THRESHOLD => 100,  // ms
    PerformanceOption::WARNING_THRESHOLD => 50,      // ms
    PerformanceOption::SAMPLING_RATE => 1.0,         // 100% of queries
    PerformanceOption::MAX_SAMPLES => 1000
]);

// Execute queries...
$db->table('users')->select()->execute();

// Get metrics
$analyzer = $db->getPerformanceMonitor()->getAnalyzer();
echo "Total queries: " . $analyzer->getQueryCount();
echo "Average time: " . $analyzer->getAverageTime() . " ms";
echo "Slow queries: " . $analyzer->getSlowQueryCount();

// Identify slow queries
$slowQueries = $analyzer->getSlowQueries();
foreach ($slowQueries as $metric) {
    echo $metric->getQuery() . " — " . $metric->getExecutionTimeMs() . " ms";
}

// Clear collected metrics
$db->clearPerformanceMetrics();
```

You can also enable/disable monitoring dynamically:

``` php
$db->enablePerformanceMonitoring();
// ... queries here are tracked ...
$db->disablePerformanceMonitoring();
```

## Command Line Utilities

WebFiori framework provides CLI commands for database management:

| Command | Description |
|---------|-------------|
| `php webfiori add:db-connection` | Add a new database connection (validates before saving) |
| `php webfiori create:table` | Create a new database table schema class |
| `php webfiori create:repository` | Create a new repository class |
| `php webfiori create:entity` | Create a new domain entity class |
| `php webfiori create:resource` | Create a complete CRUD resource (entity, table, repository, service) |
| `php webfiori create:migration` | Create a new migration class |
| `php webfiori create:seeder` | Create a new seeder class |
| `php webfiori migrations:run` | Execute pending migrations |
| `php webfiori migrations:rollback` | Roll back migrations |
| `php webfiori migrations:status` | Show migration status |
| `php webfiori migrations:dry-run` | Preview pending migrations without executing |
| `php webfiori migrations:fresh` | Rollback all and run fresh |
| `php webfiori migrations:skip` | Mark migrations as applied without executing (baseline) |
| `php webfiori migrations:step` | Interactively apply or skip one at a time |
| `php webfiori migrations:ini` | Create the migrations tracking table |

## Related Topics

* [Repository Pattern](database-repository.md) — Repository, Active Record, eager loading, and pagination
* [Migrations and Seeders](migrations.md) — Schema versioning and data seeding
* [MVC Architecture](mvc.md) — Build APIs with Controllers, Repositories, and Entities
* [Web Services](web-services.md) — Use database in API endpoints
* [Command Line Interface](command-line-interface.md) — All CLI commands
