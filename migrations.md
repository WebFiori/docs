# Database Migrations and Seeders

<meta name="description" content="Learn how to manage database schema changes and seed data using migrations and seeders in WebFiori Framework.">

In this page:
* [Introduction](#introduction)
* [How It Works](#how-it-works)
* [Creating Migrations](#creating-migrations)
  * [Using CLI](#using-cli)
  * [Manual Creation](#manual-creation)
  * [Migration Methods](#migration-methods)
  * [Dependencies](#dependencies)
  * [Environment-Specific Migrations](#environment-specific-migrations)
* [Creating Seeders](#creating-seeders)
  * [Using CLI](#using-cli-1)
  * [Manual Creation](#manual-creation-1)
* [Running Migrations](#running-migrations)
  * [Initialize Tracking Table](#initialize-tracking-table)
  * [Execute Migrations](#execute-migrations)
  * [Check Status](#check-status)
  * [Dry Run](#dry-run)
* [Rolling Back Migrations](#rolling-back-migrations)
  * [Rollback Last Batch](#rollback-last-batch)
  * [Rollback Specific Batch](#rollback-specific-batch)
  * [Rollback All](#rollback-all)
  * [Fresh Migration](#fresh-migration)
* [Batch System](#batch-system)
* [Transaction Support](#transaction-support)
* [CLI Commands Reference](#cli-commands-reference)

## Introduction

Database migrations provide a way to modify your database schema over time in a controlled, versioned manner. Instead of manually running SQL scripts, migrations allow you to define schema changes in PHP classes that can be executed, rolled back, and shared across development teams.

Seeders complement migrations by providing a way to populate your database with initial or test data. While migrations handle structure, seeders handle data.

WebFiori framework provides a complete migration system with:
- Automatic discovery of migration and seeder classes
- Dependency resolution between migrations
- Batch tracking for grouped rollbacks
- Environment-specific execution
- CLI commands for all operations

## How It Works

The migration system consists of several key components:

| Component | Description |
|-----------|-------------|
| `AbstractMigration` | Base class for all migrations |
| `AbstractSeeder` | Base class for all seeders |
| `SchemaRunner` | Orchestrates discovery, execution, and rollback |
| `schema_changes` table | Tracks which migrations have been applied |

When you run migrations:
1. The system discovers all migration/seeder classes from `App/Database/Migrations` and `App/Database/Seeders`
2. It checks which ones have already been applied (stored in `schema_changes` table)
3. Pending migrations are executed in dependency order
4. Applied migrations are recorded with a batch number

## Creating Migrations

Migrations are stored in the `App/Database/Migrations` directory.

> **Note:** The namespace assumes your application folder is named `App`. If your application folder has a different name (e.g., `ABC`), the namespace will be `ABC\Database\Migrations` instead of `App\Database\Migrations`. The same applies to seeders (`ABC\Database\Seeders`).

### Using CLI

The recommended way to create a migration is using the CLI:

``` bash
php webfiori create:migration
```

This will prompt you for:
- Class name
- Description
- Environments (optional)
- Dependencies (optional)

You can also provide arguments directly:

``` bash
php webfiori create:migration --class-name=CreateUsersTable --description="Creates the users table"
```

Available arguments:
- `--class-name` - The name of the migration class
- `--description` - A description of what the migration does
- `--environments` - Comma-separated list of environments (e.g., `dev,test`)
- `--depends-on` - Comma-separated list of migration class names this depends on

### Manual Creation

To create a migration manually, create a new class in `App/Database/Migrations` that extends `AbstractMigration`:

``` php
namespace App\Database\Migrations;

use WebFiori\Database\ColOption;
use WebFiori\Database\Database;
use WebFiori\Database\DataType;
use WebFiori\Database\Schema\AbstractMigration;

class CreateUsersTable extends AbstractMigration {
    
    public function up(Database $db): void {
        $db->createBlueprint('users')->addColumns([
            'id' => [
                ColOption::TYPE => DataType::INT,
                ColOption::PRIMARY => true,
                ColOption::AUTO_INCREMENT => true
            ],
            'username' => [
                ColOption::TYPE => DataType::VARCHAR,
                ColOption::SIZE => 50,
                ColOption::NULL => false
            ],
            'email' => [
                ColOption::TYPE => DataType::VARCHAR,
                ColOption::SIZE => 150,
                ColOption::NULL => false
            ],
            'created-at' => [
                ColOption::TYPE => DataType::TIMESTAMP,
                ColOption::DEFAULT => 'current_timestamp'
            ]
        ]);
        
        $db->table('users')->createTable();
        $db->execute();
    }
    
    public function down(Database $db): void {
        $db->raw("DROP TABLE IF EXISTS users")->execute();
    }
}
```

### Migration Methods

Every migration must implement two methods:

| Method | Description |
|--------|-------------|
| `up(Database $db)` | Apply the migration (create tables, add columns, etc.) |
| `down(Database $db)` | Reverse the migration (drop tables, remove columns, etc.) |

The `$db` parameter is a `Database` instance connected to your configured database.

Example operations in `up()`:

``` php
public function up(Database $db): void {
    // Create a table
    $db->createBlueprint('posts')->addColumns([
        'id' => [ColOption::TYPE => DataType::INT, ColOption::PRIMARY => true, ColOption::AUTO_INCREMENT => true],
        'title' => [ColOption::TYPE => DataType::VARCHAR, ColOption::SIZE => 200],
        'content' => [ColOption::TYPE => DataType::TEXT]
    ]);
    $db->table('posts')->createTable();
    $db->execute();
    
    // Add an index
    $db->raw("CREATE INDEX idx_posts_title ON posts(title)")->execute();
    
    // Add a foreign key
    $db->raw("ALTER TABLE posts ADD CONSTRAINT fk_user FOREIGN KEY (user_id) REFERENCES users(id)")->execute();
}
```

### Dependencies

Migrations can declare dependencies on other migrations. This ensures they execute in the correct order. For example, a migration that adds a foreign key must run after the migration that creates the referenced table.

``` php
namespace App\Database\Migrations;

use WebFiori\Database\Database;
use WebFiori\Database\Schema\AbstractMigration;

class AddUserIdToPosts extends AbstractMigration {
    
    public function getDependencies(): array {
        return [
            CreateUsersTable::class,
            CreatePostsTable::class
        ];
    }
    
    public function up(Database $db): void {
        $db->raw("ALTER TABLE posts ADD COLUMN user_id INT")->execute();
        $db->raw("ALTER TABLE posts ADD CONSTRAINT fk_posts_user FOREIGN KEY (user_id) REFERENCES users(id)")->execute();
    }
    
    public function down(Database $db): void {
        $db->raw("ALTER TABLE posts DROP FOREIGN KEY fk_posts_user")->execute();
        $db->raw("ALTER TABLE posts DROP COLUMN user_id")->execute();
    }
}
```

### Environment-Specific Migrations

You can restrict migrations to run only in specific environments:

``` php
namespace App\Database\Migrations;

use WebFiori\Database\Database;
use WebFiori\Database\Schema\AbstractMigration;

class AddTestData extends AbstractMigration {
    
    public function getEnvironments(): array {
        return ['dev', 'test']; // Will not run in 'prod'
    }
    
    public function up(Database $db): void {
        // Add test data
    }
    
    public function down(Database $db): void {
        // Remove test data
    }
}
```

If `getEnvironments()` returns an empty array (the default), the migration runs in all environments.

## Creating Seeders

Seeders are stored in the `App/Database/Seeders` directory and are used to populate tables with data.

### Using CLI

``` bash
php webfiori create:seeder
```

Or with arguments:

``` bash
php webfiori create:seeder --class-name=UsersSeeder --description="Seeds default users" --environments=dev,test
```

### Manual Creation

``` php
namespace App\Database\Seeders;

use WebFiori\Database\Database;
use WebFiori\Database\Schema\AbstractSeeder;

class UsersSeeder extends AbstractSeeder {
    
    public function getEnvironments(): array {
        return ['dev', 'test']; // Only seed in dev and test
    }
    
    public function run(Database $db): void {
        $users = [
            ['username' => 'admin', 'email' => 'admin@example.com', 'role' => 'admin'],
            ['username' => 'user1', 'email' => 'user1@example.com', 'role' => 'user'],
            ['username' => 'user2', 'email' => 'user2@example.com', 'role' => 'user']
        ];
        
        foreach ($users as $user) {
            $db->table('users')->insert($user)->execute();
        }
    }
}
```

Seeders implement the `run()` method instead of `up()` and `down()`. By default, seeders do not have rollback functionality, but you can override the `rollback()` method if needed:

``` php
public function rollback(Database $db): void {
    $db->table('users')->delete()->where('role', 'user')->execute();
}
```

## Running Migrations

### Initialize Tracking Table

Before running migrations for the first time, you need to create the `schema_changes` tracking table:

``` bash
php webfiori migrations:ini --connection=main
```

This creates a table that stores:
- Migration/seeder name
- Type (migration or seeder)
- Batch number
- Applied timestamp

### Execute Migrations

To run all pending migrations and seeders:

``` bash
php webfiori migrations:run --connection=main --env=dev
```

Arguments:
- `--connection` - Name of the database connection to use
- `--env` - Environment name (default: `dev`)

The command will output:
- Applied migrations/seeders
- Skipped items (already applied or environment mismatch)
- Failed items with error messages
- Total execution time

### Check Status

To see which migrations have been applied and which are pending:

``` bash
php webfiori migrations:status --connection=main
```

Output shows:
- Applied migrations (green)
- Pending migrations (yellow)

### Dry Run

To preview what migrations would run without actually executing them:

``` bash
php webfiori migrations:dry-run --connection=main
```

This shows pending migrations and the SQL queries they would execute.

## Rolling Back Migrations

### Rollback Last Batch

To rollback the most recently applied batch:

``` bash
php webfiori migrations:rollback --connection=main
```

### Rollback Specific Batch

To rollback a specific batch number:

``` bash
php webfiori migrations:rollback --connection=main --batch=3
```

### Rollback All

To rollback all migrations:

``` bash
php webfiori migrations:rollback --connection=main --all
```

### Fresh Migration

To rollback everything and re-run all migrations:

``` bash
php webfiori migrations:fresh --connection=main
```

This is useful during development when you want to reset your database to a clean state.

## Batch System

When you run migrations, all migrations applied in a single command are assigned the same batch number. This allows you to rollback groups of related changes together.

Example:
1. Run `migrations:run` - applies Migration1, Migration2, Migration3 (batch 1)
2. Run `migrations:run` - applies Migration4 (batch 2)
3. Run `migrations:rollback` - rolls back Migration4 (batch 2)
4. Run `migrations:rollback` - rolls back Migration1, Migration2, Migration3 (batch 1)

## Transaction Support

By default, migrations are wrapped in database transactions for safety. If a migration fails, changes are rolled back automatically.

You can control this behavior by overriding the `useTransaction()` method:

``` php
public function useTransaction(Database $db): bool {
    // MySQL doesn't support transactional DDL, so disable for DDL operations
    return false;
}
```

For database-aware behavior:

``` php
public function useTransaction(Database $db): bool {
    $dbType = $db->getConnectionInfo()->getDatabaseType();
    
    // MSSQL and PostgreSQL support transactional DDL
    // MySQL auto-commits DDL statements
    return $dbType !== 'mysql';
}
```

## CLI Commands Reference

| Command | Description |
|---------|-------------|
| `php webfiori create:migration` | Create a new migration class |
| `php webfiori create:seeder` | Create a new seeder class |
| `php webfiori migrations:ini` | Create the schema tracking table |
| `php webfiori migrations:run` | Execute pending migrations and seeders |
| `php webfiori migrations:rollback` | Rollback migrations |
| `php webfiori migrations:status` | Show applied and pending migrations |
| `php webfiori migrations:dry-run` | Preview migrations without executing |
| `php webfiori migrations:fresh` | Rollback all and re-run everything |

Common arguments for all commands:
- `--connection` - Database connection name
- `--env` - Environment name (dev, test, prod)

## Related Topics

* [Database](learn/database) - Database abstraction layer basics
* [Command Line Interface](learn/command-line-interface) - CLI usage guide
* [Web Services](learn/web-services) - Building APIs with database access
