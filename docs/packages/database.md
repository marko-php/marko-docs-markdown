---
title: marko/database
description: Entity-driven schema definition with the Data Mapper pattern.
---

Database abstraction with entity-driven schema, type inference, migrations, and seeders.

**This package has no implementation.** Install `marko/database-mysql` or `marko/database-pgsql` for actual database connectivity.

## Installation

```bash
composer require marko/database
```

You typically install a driver package (like `marko/database-pgsql`) which requires this automatically.

## Entity-Driven Schema

Your entity class is the single source of truth for both your PHP code and database structure. No separate migration files to write by hand, no XML mappings, no YAML configuration. Define your entities with attributes, and Marko generates the SQL to make your database match.

### Complete Example

```php title="app/blog/Entity/Post.php"
<?php

declare(strict_types=1);

namespace App\Blog\Entity;

use DateTimeImmutable;
use Marko\Database\Attributes\Table;
use Marko\Database\Attributes\Column;
use Marko\Database\Attributes\Index;
use Marko\Database\Entity\Entity;

#[Table('blog_posts')]
#[Index('idx_status_created', ['status', 'created_at'])]
class Post extends Entity
{
    #[Column(primaryKey: true, autoIncrement: true)]
    public int $id;

    #[Column(length: 255)]
    public string $title;

    #[Column(length: 255, unique: true)]
    public string $slug;

    #[Column(type: 'text')]
    public ?string $content = null;

    #[Column(default: 'draft')]
    public PostStatus $status = PostStatus::Draft;

    #[Column(references: 'users.id', onDelete: 'cascade')]
    public int $authorId;

    #[Column(type: 'timestamp', default: 'CURRENT_TIMESTAMP')]
    public DateTimeImmutable $createdAt;

    #[Column(type: 'timestamp')]
    public ?DateTimeImmutable $updatedAt = null;
}
```

### Attributes Overview

| Attribute | Purpose |
|-----------|---------|
| `#[Table]` | Defines table name (`name:`) or marks an extender (`extends:`) |
| `#[Column]` | Column configuration (name, primaryKey, autoIncrement, length, type, unique, default, references, onDelete, onUpdate) |
| `#[Index]` | Composite indexes |
| `#[Cast]` | Converts a property with a custom cast class (see [Casts](#casts)) |
| `#[Encrypted]` | Stores a property encrypted (see [Encrypted Columns](#encrypted-columns)) |
| `#[Timestamps]` | Fills `createdAt`/`updatedAt` automatically (see [Automatic Timestamps](#automatic-timestamps)) |
| `#[HasOne]` | Declares a has-one relationship to another entity |
| `#[HasMany]` | Declares a has-many relationship to another entity |
| `#[BelongsTo]` | Declares a belongs-to relationship to another entity |
| `#[BelongsToMany]` | Declares a many-to-many relationship through a pivot entity |

Property names are automatically converted from camelCase to snake_case for column names. For example, `$createdAt` maps to the `created_at` column. Use the `name` parameter to override this: `#[Column(name: 'custom_column')]`.

### Type Inference Rules

Marko infers database types from PHP types:

| PHP Type | Database Type |
|----------|---------------|
| `int` | INT (or SERIAL/BIGSERIAL if autoIncrement) |
| `string` | VARCHAR(255) by default, TEXT if type='text' |
| `bool` | BOOLEAN |
| `float` | DECIMAL or FLOAT |
| `?type` | Column is NULLABLE |
| `DateTimeImmutable` | VARCHAR unless declared --- use `type: 'timestamp'` or `type: 'datetime'` |
| `BackedEnum` | ENUM with cases as values |
| `array` or `?array` with `type: 'json'` | JSON (MySQL) / JSONB (PostgreSQL) |
| Union type (e.g. `int\|string`) | No inference — requires an explicit `type:` |
| Default values | From property initializers |

### Union-Typed Columns

A union type has no single PHP type to infer a column type from, so it must declare one explicitly. This is how polymorphic foreign keys are modeled — an attachment can point at entities whose primary keys are `int` or `string`:

```php
#[Column(type: 'varchar', length: 255)]
public int|string $attachableId = 0;
```

Without an explicit `type:`, metadata parsing throws `EntityException` — Marko will not guess which side of the union wins.

> A varchar-backed union always hydrates to a PHP `string`, even when the value was written as an `int`. Compare loosely or cast explicitly rather than strict-comparing against an integer primary key.

### String and UUID Primary Keys

Primary keys are not limited to integers. Any property marked `#[Column(primaryKey: true)]` serves as the primary key. `find()` and `findOrFail()` accept `int|string`.

> **Every entity must declare exactly one `#[Column(primaryKey: true)]` property.** Marko validates this at metadata-parse time and throws `MissingPrimaryKeyException` if none is found. There is no silent `id` fallback.

UUID primary keys work on both drivers:

```php title="app/blog/Entity/Article.php"
<?php

declare(strict_types=1);

namespace App\Blog\Entity;

use Marko\Database\Attributes\Column;
use Marko\Database\Attributes\Table;
use Marko\Database\Entity\Entity;

#[Table('articles')]
class Article extends Entity
{
    // PostgreSQL: uses gen_random_uuid() natively
    #[Column(primaryKey: true, type: 'uuid', default: 'gen_random_uuid()')]
    public string $id;

    #[Column(length: 255)]
    public string $title;
}
```

For MySQL, generate UUIDs in PHP before persisting:

```php
use Ramsey\Uuid\Uuid;

$article = new Article();
$article->id = Uuid::uuid4()->toString();
$article->title = 'Hello';
$articleRepository->save($article);
```

### JSON Columns

Store structured data directly in a column using `#[Column(type: 'json')]`. The property type must be `array` or `?array`. MySQL uses the native `JSON` type; PostgreSQL uses `JSONB`.

```php title="app/blog/Entity/Post.php"
<?php

declare(strict_types=1);

namespace App\Blog\Entity;

use Marko\Database\Attributes\Column;
use Marko\Database\Attributes\Table;
use Marko\Database\Entity\Entity;

#[Table('posts')]
class Post extends Entity
{
    #[Column(primaryKey: true, autoIncrement: true)]
    public int $id;

    #[Column(length: 255)]
    public string $title;

    #[Column(type: 'json')]
    public array $metadata = [];

    #[Column(type: 'json')]
    public ?array $settings = null;
}
```

JSON columns serialize on write and deserialize on read automatically using `JSON_THROW_ON_ERROR`. The value must be an array or null --- top-level JSON scalars are out of scope.

JSON is the pragmatic alternative to EAV tables and to running a separate document store. For structured-but-variable attributes (e.g., product options, user preferences, webhook payloads), a JSON column keeps everything in one place without the overhead of a second data layer.

### JSON query operators

Query inside JSON columns using arrow-path syntax in `where()` and `select()`, or the dedicated JSON methods:

```php
// Arrow path in where() — driver translates to JSON_EXTRACT (MySQL) or -> / ->> (PostgreSQL)
$this->query()->where('data->user->name', '=', 'Alice')->getEntities();

// Select a nested value
$this->query()->select('id', 'data->>name as display_name')->get();

// whereJsonContains — value is present in a JSON array
$this->query()->whereJsonContains('tags', 'php')->getEntities();

// whereJsonExists / whereJsonMissing — check for key presence
$this->query()->whereJsonExists('settings->notifications')->getEntities();
$this->query()->whereJsonMissing('profile->avatar')->getEntities();
```

**Path syntax:**
- `data->user->name` --- extract a nested value (returns JSON on MySQL, typed value on PostgreSQL)
- `data->>name` --- extract as text (unquoted string)

**JSON indexing** is done via raw DDL in your migration or schema setup --- the query builder does not generate index DDL for you:

```sql
-- PostgreSQL: GIN index for containment queries
CREATE INDEX idx_posts_metadata ON posts USING gin(metadata jsonb_path_ops);

-- MySQL: generated column + B-tree index
ALTER TABLE posts
    ADD COLUMN metadata_status VARCHAR(50)
        GENERATED ALWAYS AS (JSON_UNQUOTE(JSON_EXTRACT(metadata, '$.status'))) STORED,
    ADD INDEX idx_posts_metadata_status (metadata_status);
```

## Value Conversion

Every value moving between an entity property and a column goes through one pipeline in `EntityHydrator`: `toPhpValue()` on read, `toDatabaseValue()` on write. Inserts, batch inserts, updates and `findBy()`/`findOneBy()`/`existsBy()` criteria all use it, so an insert and an update of the same value always bind the same database value.

Each property is converted by a cast, chosen in this order:

| Property | Cast |
|----------|------|
| `#[Cast(SomeCast::class)]` | `SomeCast` |
| `array` (or `type: 'json'`) | `JsonCast` --- JSON encode/decode |
| `BackedEnum` | `EnumCast` --- stored as the backing value |
| `DateTimeImmutable` | `DateTimeCast` --- see [Datetimes and Timezones](#datetimes-and-timezones) |
| `int`, `float`, `bool`, `string` | `ScalarCast` --- coerced to the declared type on read |

`NULL` never reaches a cast: SQL `NULL` hydrates to PHP `null`, and `null` is written as `NULL`.

### Datetimes and Timezones

`DateTimeImmutable` values are converted to the database timezone before they are formatted, and read back in that timezone. The stored instant never depends on the PHP default timezone or on the timezone of the object you assigned:

```php
$appointment->startsAt = new DateTimeImmutable('2026-07-04 09:30:00', new DateTimeZone('America/New_York'));
$appointmentRepository->save($appointment); // stored as '2026-07-04 13:30:00'

$loaded = $appointmentRepository->find($appointment->id);
$loaded->startsAt->getTimestamp() === $appointment->startsAt->getTimestamp(); // true
$loaded->startsAt->getTimezone()->getName(); // 'UTC'
```

The database timezone defaults to UTC. Set the optional `timezone` key to change it:

```php title="config/database.php"
return [
    // ...driver, host, port, database, username, password
    'timezone' => 'UTC',
];
```

An invalid identifier fails loudly (a `ConfigurationException` naming the value) the first time a datetime is converted.

> **Behaviour change:** before casts were introduced, datetimes were formatted in whatever timezone the object carried and read back in the PHP default timezone, so a value saved from a non-UTC object came back as a different instant. Rows written that way by a non-UTC application hold local wall-clock times; convert them to UTC (or set `timezone` to the zone they were written in) when upgrading.

Datetimes are stored to the second (`Y-m-d H:i:s`). Declare the column type explicitly --- `#[Column(type: 'timestamp')]` or `#[Column(type: 'datetime')]` --- because a `DateTimeImmutable` property does not infer one.

### Casts

A cast converts a value object to and from its column. Implement `CastInterface` and point `#[Cast]` at it:

```php title="app/billing/Cast/MoneyCast.php"
<?php

declare(strict_types=1);

namespace App\Billing\Cast;

use App\Billing\Money;
use Marko\Database\Entity\Cast\CastInterface;
use Marko\Database\Entity\PropertyMetadata;

class MoneyCast implements CastInterface
{
    public function toPhp(
        mixed $value,
        PropertyMetadata $meta,
    ): Money {
        return Money::fromCents((int) $value);
    }

    public function toDatabase(
        mixed $value,
        PropertyMetadata $meta,
    ): int {
        return $value->cents;
    }
}
```

```php title="app/billing/Entity/Invoice.php"
use App\Billing\Cast\MoneyCast;
use App\Billing\Money;
use Marko\Database\Attributes\Cast;
use Marko\Database\Attributes\Column;

#[Column(type: 'integer')]
#[Cast(MoneyCast::class)]
public Money $total;
```

- **Container-resolved.** Casts are built through the container, so they can take constructor dependencies and can be replaced with a Preference. The class must implement `CastInterface`, or metadata parsing throws `EntityException`.
- **Column type.** A cast property uses the `#[Column(type:)]` you declare, otherwise the type inferred from the PHP type (`varchar` for value objects). A cast may also back a `type: 'json'` column with a non-array property.
- **Dirty checking.** A cast property is dirty when its database representation changes, so assigning an equal value object --- or mutating a mutable one in place --- is handled correctly. To decide equality yourself, implement `EquatableCastInterface::equals(mixed $a, mixed $b, PropertyMetadata $meta): bool`.
- **Criteria.** `findBy(['total' => Money::fromCents(500)])` converts the value through the cast before binding it.

### Automatic Timestamps

`#[Timestamps]` on an entity fills its creation and update times:

```php title="app/blog/Entity/Comment.php"
use DateTimeImmutable;
use Marko\Database\Attributes\Column;
use Marko\Database\Attributes\Table;
use Marko\Database\Attributes\Timestamps;
use Marko\Database\Entity\Entity;

#[Table('comments')]
#[Timestamps]
class Comment extends Entity
{
    #[Column(primaryKey: true, autoIncrement: true)]
    public ?int $id = null;

    #[Column(type: 'text')]
    public string $body;

    #[Column(type: 'timestamp')]
    public ?DateTimeImmutable $createdAt = null;

    #[Column(type: 'timestamp')]
    public ?DateTimeImmutable $updatedAt = null;
}
```

- **Insert** (`save()` on a new entity and `insertBatch()`) sets both properties. A value you set yourself is kept.
- **Update** sets `updatedAt` only when something else changed. Saving an unchanged entity writes nothing, and an `updatedAt` you changed yourself is kept.
- **Property names** default to `createdAt` and `updatedAt`. Rename them, or pass `null` to manage one yourself: `#[Timestamps(createdAt: 'publishedAt', updatedAt: null)]`. Each named property must be a `DateTimeImmutable` `#[Column]`, or metadata parsing throws `EntityException`. Passing `null` for both, or using `#[Timestamps]` on a `#[Table(extends:)]` extender, also throws.
- The time comes from the protected `Repository::now()` (UTC); override it in a repository to supply a different clock.

### Encrypted Columns

`#[Encrypted]` stores a property encrypted with [`marko/encryption`](/docs/packages/encryption/) and decrypts it on hydration:

```php
use Marko\Database\Attributes\Column;
use Marko\Database\Attributes\Encrypted;

#[Column]
#[Encrypted]
public ?string $refreshToken = null;
```

`marko/database` does not require `marko/encryption`. Install a driver first:

```bash
composer require marko/encryption-openssl
```

- **Any PHP type.** The property is converted by its normal cast first (an `array` becomes JSON, an enum its backing value, a datetime its database-timezone string), then the string is encrypted. Reads decrypt first, then convert. `#[Encrypted]` cannot be combined with `#[Cast]` on the same property (`EntityException`).
- **Schema.** The column is always `text`, whatever the PHP type. Declaring any other `type:` throws `EntityException`.
- **Loud setup errors.** `#[Encrypted]` without `marko/encryption` installed, or without an `EncryptorInterface` binding, throws `EntityException` when the entity metadata is parsed --- not on the first save.
- **Not queryable.** Ciphertext changes on every write, so encrypted columns cannot be searched, indexed or unique. `findBy()`/`findOneBy()`/`existsBy()` on an encrypted property throws `EntityException`, as does marking an encrypted column as a primary key, `unique: true`, or including it in an `#[Index]` (thrown when the entity metadata is parsed).
- **`NULL` is not encrypted.** A nullable encrypted column reveals whether a value is set.
- **Decryption failures** (wrong key, a row written before encryption was enabled) throw `EntityException` naming the entity, property and column.

## Table Extension

Any module can add columns to another module's entity table without touching the original entity class. Declare a plain `Entity` subclass with `#[Table(extends: ParentEntity::class)]` — the framework merges its columns/indexes/foreign-keys into the parent's table schema and hydrates the extender from the same row as a *companion* on the parent entity.

```php title="vendor/marko/auth/src/Entity/User.php"
#[Table(name: 'users')]
class User extends Entity
{
    #[Column(primaryKey: true, autoIncrement: true)]
    public ?int $id = null;

    #[Column]
    public string $email = '';
}
```

```php title="app/billing/Entity/UserBilling.php"
#[Table(extends: User::class)]
class UserBilling extends Entity
{
    #[Column]
    public ?string $stripeCustomerId = null;

    #[Column]
    public ?string $plan = null;
}
```

### Reading and writing companions

```php
// Attach a companion before saving
$user = new User();
$user->email = 'a@b.com';
$user->attachCompanion(new UserBilling(
    stripeCustomerId: 'cus_abc',
    plan: 'pro',
));
$userRepo->save($user); // single INSERT with parent + extender columns

// Read back — companions hydrate from the same SELECT
$loaded = $userRepo->find(1);
$billing = $loaded->companion(UserBilling::class); // typed via @template
echo $billing?->plan;

// Update — both parent and companion fields in a single UPDATE
$loaded->email = 'new@b.com';
$loaded->companion(UserBilling::class)->plan = 'enterprise';
$userRepo->save($loaded);
```

### Rules

- Specify exactly one of `name:` or `extends:` on `#[Table]`.
- An extender may not redeclare the parent's primary key, may not set `autoIncrement` on any column, and may not declare its own `name:`.
- The parent itself may not be an extender — chained extension is not supported in v1.
- Two extenders may not add the same column name or index name to a table. This fails loudly at registration with both class-strings in the error.
- Extenders have no primary key of their own and cannot have a standalone `Repository`. Use the parent's `Repository`.
- `insertBatch()` does not support entities with companions attached in v1.

### Schema merging

`SchemaRegistry::registerEntities()` is two-pass: it parses all entity classes, separates parents from extenders, then for each parent merges every linked extender's columns, indexes, and foreign keys into the parent's `Table` value object. `migrate:diff` sees the merged table — no extra code or configuration to make schema migrations aware of extender columns.

### Rolling-deploy safety

If an extender's columns are not yet present in the database (the deploy that adds the module has shipped but its migration hasn't run yet), hydration silently skips that extender. No exception, no companion attached. Once the migration runs, hydration begins populating the companion automatically.

## Data Mapper Pattern

Entities are plain PHP objects. They don't save themselves or know about the database. Repositories handle all persistence.

```php title="app/blog/Repository/PostRepository.php"
<?php

declare(strict_types=1);

namespace App\Blog\Repository;

use App\Blog\Entity\Post;
use Marko\Database\Entity\EntityCollection;
use Marko\Database\Repository\Repository;

class PostRepository extends Repository
{
    protected const ENTITY_CLASS = Post::class;

    public function findBySlug(string $slug): ?Post
    {
        return $this->findOneBy(['slug' => $slug]);
    }

    public function findPublished(): EntityCollection
    {
        return $this->query()
            ->where('status', '=', 'published')
            ->orderBy('created_at', 'desc')
            ->getEntities();
    }
}
```

### Entity Not Found

`findOrFail()` throws `EntityNotFoundException` when no row matches. It extends `RepositoryException`, so existing `catch (RepositoryException $e)` blocks keep working, and it carries `$e->entityClass` and `$e->id` for logging.

It also implements `Marko\Core\Exceptions\HttpExceptionInterface`: if it escapes a controller, the routing pipeline renders a **`404`** with the body `{"message": "Not found."}`. The entity class and ID are never sent to the client. See [Errors and HTTP Exceptions](/docs/packages/routing/#errors-and-http-exceptions).

```php
#[Get('/posts/{id}')]
public function show(int $id): Response
{
    $post = $this->postRepository->findOrFail($id); // 404 if missing

    return Response::json(['title' => $post->title]);
}
```

### Why Data Mapper?

- **Testability**: Entities are plain objects, easy to construct in tests
- **Separation**: Business logic stays in entities, persistence in repositories
- **Flexibility**: Switch databases without changing entity code
- **Clarity**: No hidden magic, explicit saves via repository

### Custom Queries with `query()`

The base `Repository` provides three ways to query, each suited to a different use case:

| Method                              | When to use                                                                 |
|-------------------------------------|-----------------------------------------------------------------------------|
| `findBy(array $criteria)`           | Simple equality matches on columns                                          |
| `matching(QuerySpecification ...)`  | Reusable, composable query fragments shared across repositories             |
| `query()`                           | One-off custom queries --- joins, raw conditions, ordering, limits, offsets |

`query()` returns a `RepositoryQueryBuilder` pre-configured with the repository's table name. It implements the full `QueryBuilderInterface` and adds entity hydration.

```php
public function findPublished(int $limit = 10): EntityCollection
{
    return $this->query()
        ->where('status', '=', 'published')
        ->whereNotNull('published_at')
        ->orderBy('published_at', 'desc')
        ->limit($limit)
        ->getEntities();
}
```

#### Returning entities vs arrays

| Method            | Returns                                 |
|-------------------|-----------------------------------------|
| `getEntities()`   | `EntityCollection<TEntity>` --- hydrated, supports eager loading |
| `firstEntity()`   | `?TEntity` --- hydrated, or `null` if no match |
| `get()`           | `array<array<string, mixed>>` --- raw rows |
| `first()`         | `?array<string, mixed>` --- raw row, or `null` |
| `count()`         | `int`                                   |

Use `getEntities()` / `firstEntity()` for typed domain objects. Drop to `get()` / `first()` only for reports or aggregates where building entities adds no value.

#### Available filters

`where`, `whereIn`, `whereNull`, `whereNotNull`, `orWhere`, `whereRaw`, `join`, `leftJoin`, `rightJoin`, `orderBy`, `orderByRaw`, `limit`, `offset`, `select`, `selectRaw`. All return `static` for chaining. The escape hatch is `raw(string $sql, array $bindings = [])` for queries the builder can't express.

#### Raw expressions

Use `selectRaw` and `whereRaw` when the structured builder methods cannot express the SQL you need. Both accept a raw expression string and an optional array of positional `?` bindings. A denylist rejects expressions containing `;`, `--`, `/*`, `*/`, or backticks --- use `?` placeholders for user-supplied values instead of interpolating them directly.

```php
// Compute a derived column inline
$rows = $this->query()
    ->select('id', 'title')
    ->selectRaw('COALESCE(published_at, created_at) AS display_date')
    ->get();

// Filter on an expression that where() cannot express
$rows = $this->query()
    ->whereRaw('COALESCE(price, base_price) > ?', [100])
    ->orderBy('title')
    ->get();

// Both can be combined freely with structured methods
$rows = $this->query()
    ->select('status')
    ->selectRaw('COUNT(*) AS total')
    ->whereRaw('EXTRACT(YEAR FROM created_at) = ?', [2024])
    ->groupBy('status')
    ->get();
```

`whereRaw` conditions are AND-combined with all other `where*` conditions and are also honoured by aggregate methods (`count`, `min`, `max`, `sum`, `avg`).

#### Aggregate functions

```php
$count  = $this->query()->where('status', '=', 'published')->count();
$count  = $this->query()->count('id');         // COUNT(id)
$total  = $this->query()->sum('amount');
$avg    = $this->query()->avg('score');
$min    = $this->query()->min('price');
$max    = $this->query()->max('price');
```

All aggregates return `int|float`. `count()` accepts an optional column name; omitting it produces `COUNT(*)`.

#### GROUP BY and HAVING

```php
$this->query()
    ->select('status', 'COUNT(*) as total')
    ->groupBy('status')
    ->having('COUNT(*) > ?', [5])
    ->get();
```

#### DISTINCT and UNION

```php
// DISTINCT rows
$rows = $this->query()->select('country')->distinct()->get();

// UNION (deduplicates) and UNION ALL (keeps duplicates)
$active   = $this->query()->where('status', '=', 'active');
$featured = $this->query()->where('featured', '=', 1);

$results = $active->union($featured)->get();
$results = $active->unionAll($featured)->get();
```

`union()` and `unionAll()` throw `UnionShapeMismatchException` if the two builders have different column counts.

#### Column aliasing

Use standard SQL `AS` syntax inside `select()`:

```php
$this->query()
    ->select('users.name as author_name', 'COUNT(*) as post_count')
    ->join('posts', 'posts.user_id', '=', 'users.id')
    ->groupBy('users.id')
    ->get();
```

#### Eager loading

Chain `->with('comments', 'author')` before `getEntities()` to load relationships in a single round trip:

```php
return $this->query()
    ->where('status', '=', 'published')
    ->with('author', 'comments.author')
    ->getEntities();
```

Dot-notation loads nested relationships.

#### Query builder requirement

`query()` depends on `QueryBuilderFactoryInterface` being injected into the repository. When a driver package (`marko/database-mysql`, `marko/database-pgsql`) is installed, the container wires this automatically. If you construct a repository manually without providing a factory, `query()` throws `RepositoryException::queryBuilderNotConfigured`.

## Relationships

Define relationships between entities using property attributes. Marko loads related entities explicitly — there is no lazy loading.

### HasOne

A user has one profile. The `foreignKey` is the property name on the **related** entity pointing back to this entity.

```php title="app/blog/Entity/User.php"
<?php

declare(strict_types=1);

namespace App\Blog\Entity;

use Marko\Database\Attributes\Column;
use Marko\Database\Attributes\HasOne;
use Marko\Database\Attributes\Table;
use Marko\Database\Entity\Entity;

#[Table('users')]
class User extends Entity
{
    #[Column(primaryKey: true, autoIncrement: true)]
    public int $id;

    #[Column(length: 255)]
    public string $name;

    #[HasOne(entityClass: Profile::class, foreignKey: 'userId')]
    public ?Profile $profile = null;
}
```

### HasMany

A post has many comments. The `foreignKey` is the property name on the **related** entity pointing back to this entity.

```php title="app/blog/Entity/Post.php"
<?php

declare(strict_types=1);

namespace App\Blog\Entity;

use Marko\Database\Attributes\Column;
use Marko\Database\Attributes\HasMany;
use Marko\Database\Attributes\Table;
use Marko\Database\Entity\Entity;
use Marko\Database\Entity\EntityCollection;

#[Table('posts')]
class Post extends Entity
{
    #[Column(primaryKey: true, autoIncrement: true)]
    public int $id;

    #[Column(length: 255)]
    public string $title;

    #[HasMany(entityClass: Comment::class, foreignKey: 'postId')]
    public EntityCollection $comments;
}
```

### BelongsTo

A comment belongs to a post. The `foreignKey` is the property name on **this** entity pointing to the related entity.

```php title="app/blog/Entity/Comment.php"
<?php

declare(strict_types=1);

namespace App\Blog\Entity;

use Marko\Database\Attributes\BelongsTo;
use Marko\Database\Attributes\Column;
use Marko\Database\Attributes\Table;
use Marko\Database\Entity\Entity;

#[Table('comments')]
class Comment extends Entity
{
    #[Column(primaryKey: true, autoIncrement: true)]
    public int $id;

    #[Column(name: 'post_id')]
    public int $postId;

    #[Column(type: 'text')]
    public string $body;

    #[BelongsTo(entityClass: Post::class, foreignKey: 'postId')]
    public ?Post $post = null;
}
```

### BelongsToMany

A post belongs to many tags through a pivot entity. The `foreignKey` is the pivot property pointing to **this** entity; `relatedKey` is the pivot property pointing to the related entity.

```php title="app/blog/Entity/Post.php"
#[BelongsToMany(
    entityClass: Tag::class,
    pivotClass: PostTag::class,
    foreignKey: 'postId',
    relatedKey: 'tagId',
)]
public EntityCollection $tags;
```

### Eager Loading

Use `with()` on the repository to load relationships without N+1 queries. Pass dot-notation strings for nested relationships.

```php
// Load posts with their comments
$posts = $postRepository->with('comments')->findAll();

// Load posts with comments and each comment's author
$posts = $postRepository->with('comments.author')->findAll();

// Multiple relationships
$posts = $postRepository->with('comments', 'tags')->findAll();
```

`with()` returns a cloned repository instance — the original is unchanged. Relationships are loaded in a single batch query per relationship level.

## EntityCollection

`findAll()` and `findBy()` return an `EntityCollection` instead of a plain array. `EntityCollection` is iterable, countable, and provides collection methods.

```php
use Marko\Database\Entity\EntityCollection;

$posts = $postRepository->findAll();

// Iterate
foreach ($posts as $post) { ... }

// Count
$posts->count();
$posts->isEmpty();

// Access
$posts->first();
$posts->last();

// Transform
$posts->filter(fn (Post $p): bool => $p->published);
$posts->map(fn (Post $p): string => $p->title);
$posts->each(fn (Post $p): void => ...);
$posts->pluck('title');          // array of property values

// Sort and group
$posts->sortBy('createdAt', descending: true);
$posts->groupBy('status');       // array<string, EntityCollection>
$posts->chunk(10);               // array<int, EntityCollection>

// Search
$posts->contains(fn (Post $p): bool => $p->id === 5);

// Convert
$posts->toArray();
```

## Query Specifications

`QuerySpecification` is an interface for encapsulating reusable query logic. Use `matching()` on the repository to apply one or more specifications.

```php title="app/blog/Query/PublishedSpec.php"
<?php

declare(strict_types=1);

namespace App\Blog\Query;

use Marko\Database\Query\EntityQueryBuilderInterface;
use Marko\Database\Query\QuerySpecification;

class PublishedSpec implements QuerySpecification
{
    public function apply(EntityQueryBuilderInterface $queryBuilder): void
    {
        $queryBuilder->where('status', '=', 'published');
    }
}
```

```php title="app/blog/Query/RecentSpec.php"
<?php

declare(strict_types=1);

namespace App\Blog\Query;

use Marko\Database\Query\EntityQueryBuilderInterface;
use Marko\Database\Query\QuerySpecification;

readonly class RecentSpec implements QuerySpecification
{
    public function __construct(
        private int $limit = 10,
    ) {}

    public function apply(EntityQueryBuilderInterface $queryBuilder): void
    {
        $queryBuilder->orderBy('created_at', 'desc')->limit($this->limit);
    }
}
```

Compose multiple specifications in a single `matching()` call:

```php
use App\Blog\Query\PublishedSpec;
use App\Blog\Query\RecentSpec;

$posts = $postRepository->matching(
    new PublishedSpec(),
    new RecentSpec(limit: 5),
);
```

### Specs with eager loading

`QuerySpecification::apply()` receives an `EntityQueryBuilderInterface`, which extends `QueryBuilderInterface` with `with()`. Specs can declare their own eager-loading needs:

```php title="app/blog/Query/PublishedWithAuthorSpec.php"
<?php

declare(strict_types=1);

namespace App\Blog\Query;

use Marko\Database\Query\EntityQueryBuilderInterface;
use Marko\Database\Query\QuerySpecification;

class PublishedWithAuthorSpec implements QuerySpecification
{
    public function apply(EntityQueryBuilderInterface $queryBuilder): void
    {
        $queryBuilder
            ->where('status', '=', 'published')
            ->with('author', 'tags');
    }
}
```

The caller does not need to know which relationships the spec requires --- they are encapsulated inside it.

## Transactions

The database driver (`marko/database-mysql` or `marko/database-pgsql`) registers `ConnectionInterface` as a shared instance: one connection, and one PDO handle, per request. Under a long-running worker such as `marko/roadrunner`, the same connection is reused for every request the worker serves. Every repository, the query builder, `marko/queue-database` and any service that injects `ConnectionInterface` all use that one connection. As a result, a transaction covers every write made through any of them.

Inject `TransactionInterface` to run a unit of work atomically. It resolves to the same shared connection:

```php title="app/billing/Service/CheckoutService.php"
<?php

declare(strict_types=1);

namespace App\Billing\Service;

use App\Billing\Entity\Invoice;
use App\Billing\Entity\Payment;
use App\Billing\Repository\InvoiceRepository;
use App\Billing\Repository\PaymentRepository;
use Marko\Database\Connection\TransactionInterface;

class CheckoutService
{
    public function __construct(
        private TransactionInterface $transaction,
        private InvoiceRepository $invoices,
        private PaymentRepository $payments,
    ) {}

    public function checkout(Invoice $invoice, Payment $payment): void
    {
        $this->transaction->transaction(function () use ($invoice, $payment): void {
            $this->invoices->save($invoice);
            $this->payments->save($payment);
        });
    }
}
```

`transaction()` commits when the callback returns and rolls back, then rethrows, when it throws. Both saves above are committed together or not at all. Use `beginTransaction()`, `commit()` and `rollback()` when you need to manage the boundaries yourself.

The entity hydrator is shared the same way. An entity loaded by one repository and saved by another is still recognised as an existing entity, and only its changed columns are written.

Under a long-running worker, the connection's `reset()` (from `ResettableInterface`) runs between requests. It rolls back any transaction a failed request left open, so the next request never inherits it.

:::caution
Depend on `ConnectionInterface` or `TransactionInterface`, never on `MySqlConnection` or `PgSqlConnection` directly. Only the interfaces are shared. Requesting a concrete connection class builds a new, separate connection that is outside every transaction. It also bypasses `marko/database-readwrite` when that package is enabled.
:::

## Query and Constraint Exceptions

When the database rejects a statement, the driver turns the `PDOException` into a typed exception from `Marko\Database\Exceptions`. This applies to repositories, the query builder, `ConnectionInterface::query()`/`execute()` and prepared statements alike. Unique and foreign key violations are normal events (double submits, races, deleting a row that is still referenced), and you can handle them without knowing which driver is installed:

| Exception | Raised when | PostgreSQL | MySQL / MariaDB | HTTP |
|-----------|-------------|------------|-----------------|------|
| `UniqueConstraintViolationException` | A duplicate value hits a unique column or index | `23505` | `1062` | `409` |
| `ForeignKeyConstraintViolationException` | A row references a missing parent, or a referenced row is deleted or updated | `23503` | `1451`, `1452` (and legacy `1216`, `1217`) | `409` |
| `NotNullConstraintViolationException` | `NULL` (or no value) is written to a `NOT NULL` column | `23502` | `1048`, `1364` | --- |
| `CheckConstraintViolationException` | A row fails a `CHECK` constraint | `23514` | `3819` (MySQL), `4025` (MariaDB) | --- |
| `QueryException` | Any other driver error | anything else | anything else | --- |

PostgreSQL is matched on the SQLSTATE and MySQL on the server error number, because MySQL reports every integrity violation as SQLSTATE `23000`.

The hierarchy is:

```
DatabaseException
└── QueryException                      sql(), bindings(), sqlState()
    └── ConstraintViolationException    constraintName(), table(), column()
        ├── UniqueConstraintViolationException
        ├── ForeignKeyConstraintViolationException
        ├── NotNullConstraintViolationException
        └── CheckConstraintViolationException
```

The original `PDOException` is always available from `getPrevious()`. `constraintName()`, `table()` and `column()` are parsed from the driver message and return `null` when the driver does not report them. `table()` is the table that owns the constraint. For a foreign key violation that is the referencing (child) table, whether the statement inserted the child or deleted the parent. For a unique violation on PostgreSQL, `column()` is the indexed column list as the server reports it, for example `email` or `tenant_id, email`.

### Handle a Duplicate Email

Catch `UniqueConstraintViolationException` and turn it into a validation error:

```php title="app/account/Service/RegistrationService.php"
<?php

declare(strict_types=1);

namespace App\Account\Service;

use App\Account\Entity\User;
use App\Account\Repository\UserRepository;
use Marko\Database\Exceptions\UniqueConstraintViolationException;
use Marko\Validation\Exceptions\ValidationException;
use Marko\Validation\Validation\ValidationErrors;

class RegistrationService
{
    public function __construct(
        private UserRepository $userRepository,
    ) {}

    /**
     * @throws ValidationException
     */
    public function register(string $email): User
    {
        $user = new User();
        $user->email = $email;

        try {
            $this->userRepository->save($user);
        } catch (UniqueConstraintViolationException $e) {
            if ($e->constraintName() !== 'users_email_unique') {
                throw $e;
            }

            throw ValidationException::withErrors(
                new ValidationErrors(['email' => ['This email address is already registered.']]),
            );
        }

        return $user;
    }
}
```

Checking with `existsBy(['email' => $email])` first gives a friendlier path for the common case. Keep the `catch` anyway: two requests can both pass the check before either one inserts, and only the database constraint catches that race.

:::caution
PostgreSQL aborts the current transaction after any error. If the `save()` above runs inside `transaction()`, every later statement in that transaction fails with SQLSTATE `25P02` until it is rolled back. Catch the violation outside the transaction (let `transaction()` roll back and rethrow), or roll back before you continue. MySQL does not abort the transaction, but keep the same shape so the code works on both drivers.
:::

### HTTP Status

`UniqueConstraintViolationException` and `ForeignKeyConstraintViolationException` implement `Marko\Core\Exceptions\HttpExceptionInterface`. If one escapes a controller, the routing pipeline renders a **`409 Conflict`** with the body `{"message": "Conflict."}`. The constraint name, SQL and bound values are never sent to the client. Not-null and check violations usually point at a missing validation rule, so they are not mapped and surface as a `500`. See [Errors and HTTP Exceptions](/docs/packages/routing/#errors-and-http-exceptions).

`marko/database` depends only on `marko/core` for this interface. It does not require `marko/routing` or an errors package.

### Messages and Redaction

Exception messages are built from the parsed constraint, table and column, never copied from the driver. PostgreSQL's `DETAIL:` line (`Key (email)=(ada@example.com) already exists`) and MySQL's `Duplicate entry 'ada@example.com'` both contain row data. For example:

```
Unique constraint 'users_email_unique' violated on table 'users'
```

The exception context holds the SQL with its placeholders. It never holds the bound values. A plain `QueryException` uses the first line of the driver message, with any string binding the driver echoes replaced by `[redacted]`. The raw values stay available to your own code through `bindings()`, and the untouched driver message through `getPrevious()`. Be careful what you log from either one.

### Upgrading From `catch (PDOException)`

Before this change the raw `PDOException` escaped from `query()`, `execute()` and the repositories. Code that catches `PDOException` around those calls no longer matches. Catch `QueryException` (or a constraint subclass) instead, and read `sqlState()` or `getPrevious()` where you used to inspect the PDO error. This includes retry loops for deadlocks and serialization failures (SQLSTATE `40001` / `40P01`, MySQL `1213`):

```php
use Marko\Database\Exceptions\QueryException;

try {
    $this->transaction->transaction($work);
} catch (QueryException $e) {
    if (!in_array($e->sqlState(), ['40001', '40P01'], true)) {
        throw $e;
    }

    // retry $work
}
```

## Bulk Insert

`Repository::insertBatch(array $entities): void` inserts multiple entities in a single multi-row `INSERT` statement, wrapped in a transaction. It fires `EntityCreating` and `EntityCreated` events for each entity.

```php
use App\Blog\Entity\Post;

$posts = [];
for ($i = 1; $i <= 1000; $i++) {
    $post = new Post();
    $post->title = "Post {$i}";
    $post->slug  = "post-{$i}";
    $posts[] = $post;
}

$postRepository->insertBatch($posts);
```

**Caveats:**

- Relationships are **not** auto-persisted. Persist related entities separately before calling `insertBatch()`.
- `EntityCreating` / `EntityCreated` events fire synchronously for every entity in the batch. For high-throughput imports, mark observers async via `marko/queue` or drop to the raw query builder to avoid the per-row overhead.
- A constraint violation on any row rolls back the whole batch and rethrows the typed exception (for example `UniqueConstraintViolationException`, with the `PDOException` as `getPrevious()`). It is not wrapped in `BatchInsertException`, so the same `catch` works for `save()` and `insertBatch()`. See [Query and Constraint Exceptions](#query-and-constraint-exceptions).
- All entities must be of the same type and have identical column sets. `BatchInsertException` is thrown for empty input, mixed types, mismatched columns, or (on PostgreSQL) when the number of rows returned by `RETURNING` does not equal the number of inserted entities.

**ID assignment after batch insert:**

- **MySQL** --- IDs are recovered from `lastInsertId()` plus sequential offset.
- **PostgreSQL** --- uses `INSERT ... RETURNING id` to retrieve each generated ID.

## Seeders

Seeders populate development/test databases with sample data. They're discovered via the `#[Seeder]` attribute.

Each seeder runs inside a database transaction. If a seeder fails partway through, all its changes are automatically rolled back — preventing partial data that would require manual cleanup.

```php title="app/blog/Seed/PostSeeder.php"
<?php

declare(strict_types=1);

namespace App\Blog\Seed;

use App\Blog\Entity\Post;
use App\Blog\Repository\PostRepositoryInterface;
use Marko\Database\Seed\Seeder;
use Marko\Database\Seed\SeederInterface;

#[Seeder(name: 'posts', order: 10)]
readonly class PostSeeder implements SeederInterface
{
    public function __construct(
        private PostRepositoryInterface $postRepository,
    ) {}

    public function run(): void
    {
        $post = new Post();
        $post->title = 'Hello World';
        $post->slug = 'hello-world';
        $post->content = 'Welcome to my blog!';
        $post->createdAt = date('Y-m-d H:i:s');

        $this->postRepository->save($post);
    }
}
```

> **Why `new Post()` instead of factories?** Entities are simple data objects without dependencies or complex construction logic. Direct instantiation is explicit — you see exactly what's being set. This aligns with Marko's "explicit over implicit" principle. If your tests need realistic fake data at scale, consider adding a test data builder for that specific need rather than a general factory abstraction.

> **IDE Note:** PhpStorm may report seeder classes as "unused" since they're discovered via attributes rather than direct instantiation. The `@noinspection PhpUnused` annotation suppresses this false positive.

Place seeders in your module's `Seed/` directory. The `order` parameter controls execution sequence — use spaced numbers (10, 20, 30) rather than sequential (1, 2, 3) to allow other modules to insert seeders between existing ones without renumbering.

## CLI Commands

| Command | Description |
|---------|-------------|
| `marko db:status` | Show migration status |
| `marko db:diff` | Preview changes between entities and database |
| `marko db:migrate` | Generate and apply migrations |
| `marko db:rollback` | Revert last migration batch (development only) |
| `marko db:reset` | Rollback all migrations (development only) |
| `marko db:rebuild` | Reset + re-run all migrations (development only) |
| `marko db:seed` | Run seeders (development only) |

### Development Workflow

```bash
# 1. Define/modify your entity
# 2. Preview what will change
marko db:diff

# 3. Generate migration and apply it
marko db:migrate

# 4. If mistake, rollback (development only)
marko db:rollback
```

### Production Workflow

```bash
# Deploy code (includes migration files)
# Apply existing migrations only
marko db:migrate
```

In production, `db:migrate` only applies existing migration files — it never generates new ones.

## Switching Database Drivers

Since entities are the single source of truth, switching between database systems is a config change — each driver's `SqlGenerator` translates entity attributes to native SQL automatically.

### Example: MySQL to PostgreSQL

1. Delete the migration files in `database/migrations/` — they contain MySQL-specific SQL:

```bash
rm database/migrations/*.php
```

2. Swap drivers:

```bash
composer remove marko/database-mysql
composer require marko/database-pgsql
```

3. Update your database config:

```php title="config/database.php"
return [
    'driver' => 'pgsql',
    'host' => '127.0.0.1',
    'port' => 5432,
    'database' => 'myapp',
    'username' => 'postgres',
    'password' => '',
];
```

4. Create the database and run migrations:

```bash
createdb myapp
marko db:migrate
marko db:seed
```

`db:migrate` diffs entity attributes against the empty database, generates new migration files with PostgreSQL-native SQL (e.g., `SERIAL` instead of `AUTO_INCREMENT`, `BOOLEAN` instead of `TINYINT(1)`), and applies them. Your entity code and application logic remain unchanged.

## Framework Comparison

| Feature | Laravel | Doctrine | Marko |
|---------|---------|----------|-------|
| Schema definition | Separate migration files | XML/YAML or attributes | Entity attributes (single source of truth) |
| Migration generation | Manual | `doctrine:schema:update` | `db:migrate` auto-generates |
| Entity persistence | Active Record (Eloquent) | Data Mapper | Data Mapper |
| Schema location | `database/migrations/` | Mapping files or entity | Entity only |

## Benefits of Entity as Single Source of Truth

1. **No schema drift** — Entity changes automatically sync to database
2. **Refactoring updates both** — Rename a property, schema updates automatically
3. **IDE support** — Full autocomplete and type checking for schema
4. **No context switching** — Everything about your model in one place
5. **Reduced cognitive load** — One file to understand, not entity + migration + mapping

## Wire-compatible database variants

Some databases speak an existing wire protocol (PostgreSQL or MySQL) but require different SQL dialect logic. CockroachDB, for example, accepts PostgreSQL connections but has its own DDL, introspection queries, and query-builder behaviour. A variant package can reuse the parent driver's connection and override only the four dialect interfaces.

### The 6-binding split

Every driver package binds six interfaces. They fall into two categories:

| Interface | Category | Role |
|-----------|----------|------|
| `ConnectionInterface` | **Wire** | PDO connection, DSN format, PostgreSQL/MySQL protocol; exposes `driverName(): string` (e.g. `'mysql'`, `'pgsql'`) so dialect-aware code can branch without a live connection |
| `ConnectionFactoryInterface` | **Wire** | Creates `ConnectionInterface` instances from a `DatabaseConfig` |
| `SqlGeneratorInterface` | Dialect | DDL generation for schema diffs |
| `IntrospectorInterface` | Dialect | Reading existing schema from `information_schema` etc. |
| `QueryBuilderInterface` | Dialect | SELECT/INSERT/UPDATE/DELETE SQL generation |
| `QueryBuilderFactoryInterface` | Dialect | Constructs query builder instances |

A wire-compatible variant inherits the parent's `ConnectionInterface` and `ConnectionFactoryInterface` bindings unchanged and overrides the four dialect interfaces.

### CockroachDB example

The following shows the complete wiring for a hypothetical third-party `acme/database-cockroachdb` package. The `acme/` vendor and class names are illustrative — the `marko/` namespace is reserved for core Marko packages, so variant packages must ship under their own vendor namespace.

**`composer.json`** — require the parent pgsql package (which transitively requires `marko/database`):

```json title="composer.json"
{
    "name": "acme/database-cockroachdb",
    "description": "CockroachDB variant for Marko (PostgreSQL wire protocol)",
    "type": "marko-module",
    "require": {
        "marko/database-pgsql": "^1.0"
    },
    "autoload": {
        "psr-4": {
            "Acme\\Database\\CockroachDb\\": "src/"
        }
    }
}
```

**`module.php`** — no static `bindings` for the dialect interfaces; a `boot` closure rebinds them after `marko/database-pgsql` has registered its own static bindings:

```php title="module.php"
<?php

declare(strict_types=1);

use Acme\Database\CockroachDb\Diff\CockroachDbGenerator;
use Acme\Database\CockroachDb\Introspection\CockroachDbIntrospector;
use Acme\Database\CockroachDb\Query\CockroachDbQueryBuilder;
use Acme\Database\CockroachDb\Query\CockroachDbQueryBuilderFactory;
use Marko\Core\Container\Container;
use Marko\Database\Diff\SqlGeneratorInterface;
use Marko\Database\Introspection\IntrospectorInterface;
use Marko\Database\Query\QueryBuilderFactoryInterface;
use Marko\Database\Query\QueryBuilderInterface;

// ConnectionInterface is intentionally omitted: CockroachDB speaks the
// PostgreSQL wire protocol, so PgSqlConnection from marko/database-pgsql
// connects and authenticates without modification.

return [
    'boot' => function (Container $container): void {
        $container->bind(SqlGeneratorInterface::class, CockroachDbGenerator::class);
        $container->bind(IntrospectorInterface::class, CockroachDbIntrospector::class);
        $container->bind(QueryBuilderInterface::class, CockroachDbQueryBuilder::class);
        $container->bind(QueryBuilderFactoryInterface::class, CockroachDbQueryBuilderFactory::class);
    },
];
```

Because `acme/database-cockroachdb` requires `marko/database-pgsql` in its `composer.json`, Marko automatically boots the variant after the parent — no `sequence` configuration is needed.

For the underlying `boot` callback mechanism, see [Overriding another module's bindings](/docs/concepts/dependency-injection/#overriding-another-modules-bindings).

## Available Drivers

- [marko/database-pgsql](/docs/packages/database-pgsql/) — PostgreSQL driver
- [marko/database-mysql](/docs/packages/database-mysql/) — MySQL driver

## Read/Write Splitting

To route reads to replicas and writes to a primary, see [marko/database-readwrite](/docs/packages/database-readwrite/). It wraps any existing driver connection using the decorator pattern — no changes to application code are required.
