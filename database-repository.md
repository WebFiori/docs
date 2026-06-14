# Repository Pattern

<meta name="description" content="Learn how to use the Repository pattern, Active Record, eager loading, and pagination with WebFiori's database layer.">

In this page:
* [Introduction](#introduction)
* [Repository Pattern](#repository-pattern)
  * [Creating an Entity](#creating-an-entity)
  * [Creating a Repository](#creating-a-repository)
  * [CRUD Operations](#crud-operations)
  * [Custom Queries](#custom-queries)
* [Pagination](#pagination)
  * [Offset-Based Pagination](#offset-based-pagination)
  * [Cursor-Based Pagination](#cursor-based-pagination)
* [Eager Loading](#eager-loading)
  * [Defining Relationships](#defining-relationships)
  * [Using `with()` (Preload Strategy)](#using-with-preload-strategy)
  * [Using `withJoin()` (Single Query)](#using-withjoin-single-query)
* [Active Record Pattern](#active-record-pattern)
* [Bulk Operations](#bulk-operations)
* [CLI Scaffolding](#cli-scaffolding)

## Introduction

The database library provides [`AbstractRepository`](https://webfiori.com/docs/WebFiori/Database/Repository/AbstractRepository) — a base class that handles common data access operations (CRUD, pagination, eager loading) so you don't have to write boilerplate SQL for each entity.

Two patterns are supported:
- **Repository Pattern**: Separate entity and repository classes. Best for clean architecture and testability.
- **Active Record Pattern**: Entity and repository merged into one class. Best for rapid development and simple models.

## Repository Pattern

### Creating an Entity

An entity is a plain PHP class representing your domain object. No framework dependencies:

``` php
class Product {
    public ?int $id = null;
    public string $name;
    public string $category;
    public float $price;
    public int $stock;

    public function __construct(string $name = '', string $category = '', float $price = 0, int $stock = 0) {
        $this->name = $name;
        $this->category = $category;
        $this->price = $price;
        $this->stock = $stock;
    }
}
```

### Creating a Repository

Extend `AbstractRepository` and implement four abstract methods:

``` php
use WebFiori\Database\Repository\AbstractRepository;

class ProductRepository extends AbstractRepository {
    protected function getTableName(): string {
        return 'products';
    }

    protected function getIdField(): string {
        return 'id';
    }

    protected function toEntity(array $row): object {
        $product = new Product();
        $product->id = (int) $row['id'];
        $product->name = $row['name'];
        $product->category = $row['category'];
        $product->price = (float) $row['price'];
        $product->stock = (int) $row['stock'];
        return $product;
    }

    protected function toArray(object $entity): array {
        return [
            'id' => $entity->id,
            'name' => $entity->name,
            'category' => $entity->category,
            'price' => $entity->price,
            'stock' => $entity->stock
        ];
    }
}
```

| Method | Purpose |
|--------|---------|
| `getTableName()` | Returns the database table name |
| `getIdField()` | Returns the primary key column name |
| `toEntity(array $row)` | Maps a database row to an entity object |
| `toArray(object $entity)` | Maps an entity to an associative array for insert/update |

### CRUD Operations

``` php
$db = new Database($connectionInfo);
$repo = new ProductRepository($db);

// Create
$product = new Product('Widget', 'Hardware', 29.99, 50);
$repo->save($product);

// Read
$product = $repo->findById(1);
$allProducts = $repo->findAll();
$count = $repo->count();

// Update (save detects existing ID and updates)
$product->price = 24.99;
$repo->save($product);

// Delete
$repo->deleteById(1);
$repo->deleteAll();

// Reload from database
$fresh = $repo->reload($product);
```

The `save()` method is smart: if the entity has a non-null ID, it updates; otherwise, it inserts.

### Custom Queries

For queries beyond basic CRUD, use `getDatabase()` or `createQuery()`:

``` php
class ProductRepository extends AbstractRepository {
    // ... abstract methods ...

    public function findByCategory(string $category): array {
        $result = $this->getDatabase()->table($this->getTableName())
            ->select()
            ->where('category', $category)
            ->execute();

        return array_map(fn($row) => $this->toEntity($row), $result->fetchAll());
    }

    public function findLowStock(int $threshold = 10): array {
        $result = $this->getDatabase()->table($this->getTableName())
            ->select()
            ->where('stock', $threshold, '<')
            ->execute();

        return array_map(fn($row) => $this->toEntity($row), $result->fetchAll());
    }
}
```

## Pagination

### Offset-Based Pagination

Traditional page numbers. Good for UIs with "Page 1, 2, 3..." navigation:

``` php
$page = $repo->paginate(page: 1, perPage: 20);

echo $page->getTotalItems();    // Total records in table
echo $page->getTotalPages();    // Calculated total pages
echo $page->getCurrentPage();   // Current page number
echo $page->hasNextPage();      // bool
echo $page->hasPreviousPage();  // bool
echo $page->getNextPage();      // int or null
echo $page->getPreviousPage();  // int or null

foreach ($page->getItems() as $product) {
    echo $product->name;
}
```

With ordering:

``` php
$page = $repo->paginate(page: 2, perPage: 10, orderBy: ['price' => 'DESC']);
```

### Cursor-Based Pagination

Better for large datasets and infinite scroll. Uses a cursor (last seen value) instead of offset:

``` php
// First page
$page = $repo->paginateByCursor(cursor: null, limit: 20, cursorColumn: 'id', direction: 'ASC');

foreach ($page->getItems() as $product) {
    echo $product->name;
}

echo $page->hasMore();          // bool
echo $page->getNextCursor();    // string (pass to next request)

// Next page
$nextPage = $repo->paginateByCursor(
    cursor: $page->getNextCursor(),
    limit: 20,
    cursorColumn: 'id',
    direction: 'ASC'
);
```

Cursor-based pagination is more efficient than offset for large tables because it doesn't require counting all rows or skipping past them.

## Eager Loading

Eager loading solves the N+1 query problem. Without it, fetching 100 authors and their posts would require 101 queries (1 for authors + 100 for each author's posts). With eager loading, it takes 2 queries.

### Defining Relationships

Relationships are defined on the **table class** using attributes:

**One-to-Many (HasMany):**

``` php
use WebFiori\Database\Attributes\Column;
use WebFiori\Database\Attributes\HasMany;
use WebFiori\Database\Attributes\Table;
use WebFiori\Database\DataType;

#[Table(name: 'authors')]
#[HasMany(entity: Post::class, foreignKey: 'author-id', property: 'posts', table: 'posts')]
class AuthorsTable {
    #[Column(type: DataType::INT, primary: true, autoIncrement: true)]
    public int $id;

    #[Column(type: DataType::VARCHAR, size: 100)]
    public string $name;
}
```

**Many-to-One (BelongsTo via ForeignKey):**

``` php
#[Table(name: 'posts')]
#[HasMany(entity: Comment::class, foreignKey: 'post-id', property: 'comments', table: 'comments')]
class PostsTable {
    #[Column(type: DataType::INT, primary: true, autoIncrement: true)]
    public int $id;

    #[Column(type: DataType::VARCHAR, size: 200)]
    public string $title;

    #[Column(name: 'author-id', type: DataType::INT)]
    #[ForeignKey(table: AuthorsTable::class, column: 'id', property: 'author', entity: Author::class)]
    public int $authorId;
}
```

Key parameters:
- `#[HasMany]`: `entity` (target class), `foreignKey` (FK column in child table), `property` (property name on parent entity), `table` (child table name)
- `#[ForeignKey]` with `property` and `entity`: defines a belongsTo relationship alongside the FK constraint

The repository must reference the table class via `getTableClass()`:

``` php
class AuthorRepository extends AbstractRepository {
    protected function getTableClass(): string {
        return AuthorsTable::class;
    }

    protected function getTableName(): string {
        return 'authors';
    }

    protected function getIdField(): string {
        return 'id';
    }

    protected function toEntity(array $row): object {
        $author = new Author();
        $author->id = (int) $row['id'];
        $author->name = $row['name'];
        return $author;
    }

    protected function toArray(object $entity): array {
        return ['id' => $entity->id, 'name' => $entity->name];
    }
}
```

### Using `with()` (Preload Strategy)

Loads related data using 1+N queries (1 per relation type, not per entity):

``` php
// Load authors with their posts (2 queries: 1 for authors + 1 for all posts)
$authors = $authorRepo->with('posts')->findAll();

foreach ($authors as $author) {
    echo $author->name;
    foreach ($author->posts as $post) {
        echo "  - " . $post->title;
    }
}

// Multiple relations
$posts = $postRepo->with(['author', 'comments'])->findAll();

// Works with findById and paginate too
$author = $authorRepo->with('posts')->findById(1);
$page = $authorRepo->with('posts')->paginate(page: 1, perPage: 10);
```

### Using `withJoin()` (Single Query)

For belongsTo relations only, uses a LEFT JOIN to load related data in a single query:

``` php
// Load posts with their author (1 query with JOIN)
$posts = $postRepo->withJoin('author')->findAll();

foreach ($posts as $post) {
    echo $post->title . " by " . $post->author->name;
}
```

`withJoin()` is more efficient (single query) but only works for belongsTo (many-to-one) relations. For hasMany (one-to-many), use `with()` — using a JOIN would create a cartesian product.

``` php
// This throws RepositoryException — hasMany cannot be joined
$authorRepo->withJoin('posts')->findAll(); // ERROR
```

## Active Record Pattern

For simpler projects, merge the entity and repository into a single class:

``` php
use WebFiori\Database\Attributes\Column;
use WebFiori\Database\Attributes\Table;
use WebFiori\Database\Database;
use WebFiori\Database\DataType;
use WebFiori\Database\Repository\AbstractRepository;

#[Table(name: 'articles')]
class Article extends AbstractRepository {
    #[Column(type: DataType::INT, primary: true, autoIncrement: true)]
    public ?int $id = null;

    #[Column(type: DataType::VARCHAR, size: 200)]
    public string $title = '';

    #[Column(type: DataType::TEXT)]
    public string $content = '';

    #[Column(name: 'author-name', type: DataType::VARCHAR, size: 100)]
    public string $authorName = '';

    public function __construct(Database $db) {
        parent::__construct($db);
    }

    // Custom query
    public function findByAuthor(string $author): array {
        $result = $this->getDatabase()->table($this->getTableName())
            ->select()
            ->where('author-name', $author)
            ->execute();

        return array_map(fn($row) => $this->toEntity($row), $result->fetchAll());
    }

    protected function getTableName(): string { return 'articles'; }
    protected function getIdField(): string { return 'id'; }

    protected function toEntity(array $row): object {
        $article = new self($this->db);
        $article->id = (int) $row['id'];
        $article->title = $row['title'];
        $article->content = $row['content'];
        $article->authorName = $row['author-name'] ?? '';
        return $article;
    }

    protected function toArray(object $entity): array {
        return [
            'id' => $entity->id,
            'title' => $entity->title,
            'content' => $entity->content,
            'author-name' => $entity->authorName,
        ];
    }
}
```

Usage:

``` php
// Create the table from attributes
$table = AttributeTableBuilder::build(Article::class, 'mysql');
$db->addTable($table);
$db->table('articles')->createTable()->execute();

// Use the model
$article = new Article($db);
$article->title = 'Hello World';
$article->content = 'My first article';
$article->authorName = 'Ibrahim';
$article->save();

// Query
$all = $article->findAll();
$one = $article->findById(1);
$byAuthor = $article->findByAuthor('Ibrahim');

// Update
$article->title = 'Updated Title';
$article->save();

// Delete
$article->deleteById(1);
```

**When to use which:**

| | Repository Pattern | Active Record |
|---|---|---|
| Testability | High (inject mock DB) | Lower (tightly coupled) |
| Separation of concerns | Entity is pure, no DB knowledge | Entity knows about DB |
| Code amount | More files | Fewer files |
| Best for | Complex domains, team projects | Simple CRUD, prototypes |

## Bulk Operations

Save multiple entities in a single transaction:

``` php
$products = [
    new Product('Widget A', 'Hardware', 10.00, 100),
    new Product('Widget B', 'Hardware', 15.00, 50),
    new Product('Widget C', 'Hardware', 20.00, 25),
];

$repo->saveAll($products);
```

`saveAll()` automatically batches inserts for new entities and updates for existing ones, wrapped in a transaction.

## CLI Scaffolding

The framework provides commands to generate repository-related classes:

``` bash
# Create a repository class
php webfiori create:repository

# Create a domain entity class
php webfiori create:entity

# Create a complete CRUD resource (entity + table + repository + service)
php webfiori create:resource
```

The `create:resource` command is the quickest way to scaffold a full CRUD setup — it generates all the files needed for a working API endpoint with database access.

## Related Topics

* [Database](database.md) — Connections, query builder, table definitions
* [Migrations and Seeders](migrations.md) — Schema versioning
* [MVC Architecture](mvc.md) — Controllers, Repositories, and Entities
* [Web Services](web-services.md) — Expose repositories as APIs
