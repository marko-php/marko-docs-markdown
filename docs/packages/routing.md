---
title: marko/routing
description: Attribute-based routing with automatic conflict detection — define routes on controller methods, not in separate files.
---

Routes live on the methods they handle. Conflicts are caught at boot time with clear error messages. Override vendor routes cleanly via [Preferences](/docs/packages/core/), or disable them explicitly with `#[DisableRoute]`.

## Installation

```bash
composer require marko/routing
```

## Usage

### Defining Routes

Add route attributes to controller methods:

```php title="ProductController.php"
use Marko\Routing\Attributes\Get;
use Marko\Routing\Attributes\Post;
use Marko\Routing\Http\Response;

class ProductController
{
    #[Get('/products')]
    public function index(): Response
    {
        return new Response('Product list');
    }

    #[Get('/products/{id}')]
    public function show(
        int $id,
    ): Response {
        return new Response("Product $id");
    }

    #[Post('/products')]
    public function store(): Response
    {
        return new Response('Created', 201);
    }
}
```

Route parameters, POST body values, and query string values are automatically resolved and passed to method arguments. Typed scalar parameters (`int`, `float`, `bool`, `string`) are cast to the declared type. If a required typed scalar parameter cannot be found in the route, POST body, or query string, the router returns a `400` response (via `InvalidRouteParameterException`) instead of throwing a `TypeError`.

### Available Methods

```php
#[Get('/path')]
#[Post('/path')]
#[Put('/path')]
#[Patch('/path')]
#[Delete('/path')]
#[Head('/path')]
#[Options('/path')]
```

You rarely need `#[Head]` or `#[Options]` --- the router answers both automatically (see [HEAD and OPTIONS](#head-and-options)). Declare them only when a route needs custom behaviour.

### Route Precedence

Which route handles a URL never depends on registration order, module order or file layout. For each HTTP method the router tries routes in this order:

1. **Static paths** (no parameters), matched by exact lookup. `/shows/live` always beats `/shows/{id}`.
2. **Dynamic paths with more static segments.** `/a/{x}/c` (2 static segments) beats `/a/{x}/{y}` (1).
3. **Dynamic paths with a longer static prefix** (the text before the first `{`). `/api/{version}/list` beats `/{tenant}/users/list`.
4. **Registration order** breaks any remaining tie.

A trailing slash is ignored (`/shows/live/` matches `/shows/live`). `marko route:list` prints routes in this effective order.

### Unmatched Requests: 404 and 405

A request that matches no route still runs through every **global** middleware (session, CORS, security headers, logging, ...), then the router responds with:

- **`405 Method Not Allowed`** when the path matches a route of another method. The `Allow` header lists the methods that would work, e.g. `Allow: GET, HEAD, OPTIONS`.
- **`404 Not Found`** otherwise.

Both are thrown as `HttpException` and rendered by `ExceptionRenderer` --- JSON when the client asks for it, a minimal HTML page otherwise (see [Errors and HTTP Exceptions](#errors-and-http-exceptions)). Global middleware can decorate them like any other response.

Route middleware (`#[Middleware]`) only runs when a route matched. In global middleware, `$request->controller()` and `$request->action()` are `null` for unmatched requests --- handle that case if your middleware reads them.

### HEAD and OPTIONS

- **HEAD**: when no `#[Head]` route matches, the GET route for the same path handles the request. The response keeps its status, headers and cookies, but the router always removes the body of a response to a HEAD request (including 404/405 pages). `Response::withoutBody()` does this and preserves the concrete response class; a `StreamingResponse` sends its headers and never opens the stream.
- **OPTIONS**: when no `#[Options]` route matches but the path matches other routes, the router answers `204 No Content` with an `Allow` header (always including `HEAD` when `GET` is allowed, and `OPTIONS`). This automatic response goes through global middleware, so [`marko/cors`](/docs/packages/cors/) turns a browser preflight into a full CORS response. An OPTIONS request to an unknown path gets a 404.

### Adding Middleware

```php title="AdminController.php"
use Marko\Routing\Attributes\Middleware;

class AdminController
{
    #[Get('/admin/dashboard')]
    #[Middleware(AuthMiddleware::class)]
    public function dashboard(): Response
    {
        return new Response('Admin dashboard');
    }
}
```

Middleware classes implement `MiddlewareInterface`:

```php title="AuthMiddleware.php"
use Marko\Routing\Http\Request;
use Marko\Routing\Http\Response;
use Marko\Routing\Middleware\MiddlewareInterface;

class AuthMiddleware implements MiddlewareInterface
{
    public function handle(
        Request $request,
        callable $next,
    ): Response {
        if (!$this->isAuthenticated($request)) {
            return new Response('Unauthorized', 401);
        }

        return $next($request);
    }
}
```

### Decorating Responses

Middleware that adds headers, cookies, or changes the status code should decorate the response returned by `$next()` rather than construct a new one:

```php title="SecurityHeadersMiddleware.php"
use Marko\Routing\Http\Request;
use Marko\Routing\Http\Response;
use Marko\Routing\Middleware\MiddlewareInterface;

class SecurityHeadersMiddleware implements MiddlewareInterface
{
    public function handle(
        Request $request,
        callable $next,
    ): Response {
        $response = $next($request);

        return $response->withHeader('X-Frame-Options', 'DENY');
    }
}
```

`withHeader()`, `withHeaders()`, `withStatus()`, and `withCookie()` each return a clone of the response, preserving its concrete class. Rebuilding a response instead, e.g. `new Response($response->body(), $response->statusCode(), $headers)`, silently discards subclass identity --- a `StreamingResponse` returned by an SSE endpoint would be downgraded to a plain `Response` and its stream would never send. Always decorate, never rebuild.

### Setting Cookies

Attach a cookie to a response with `withCookie()`:

```php
use Marko\Routing\Attributes\Get;
use Marko\Routing\Http\Cookie;
use Marko\Routing\Http\Response;

#[Get('/login')]
public function login(): Response
{
    return Response::json(['ok' => true])
        ->withCookie(new Cookie(
            name: 'session',
            value: $sessionId,
            expires: time() + 3600,
            path: '/',
            secure: true,
            httpOnly: true,
            sameSite: 'Lax',
        ));
}
```

`expires: null` or `expires: 0` omits the `Expires` attribute entirely, producing a browser-session cookie instead of a persistent one. Cookie values are `rawurlencode()`d automatically, so a value containing `;` cannot inject a second attribute. An invalid cookie name throws `CookieException`, as does `sameSite: 'None'` without `secure: true` --- browsers silently drop such cookies, so Marko fails loudly instead.

Read cookies sent by the client with `Request::cookie()`, which mirrors `query()` and `post()`:

```php
$sessionId = $request->cookie('session');
```

### Reading JSON Bodies

A request whose `Content-Type` is `application/json` (or any `+json` type, such as `application/vnd.api+json`) has its body decoded once, when the `Request` is built. Read it with `json()`, using dot-notation for nested keys:

```php
use Marko\Routing\Attributes\Post;
use Marko\Routing\Http\Request;
use Marko\Routing\Http\Response;

#[Post('/api/users')]
public function store(Request $request): Response
{
    $email = $request->json('user.email');
    $roles = $request->json('user.roles', []);

    return Response::json(['email' => $email], 201);
}
```

JSON fields also bind straight to typed controller parameters, the same way form fields do:

```php
#[Post('/api/shows')]
public function create(string $title, int $count): Response
{
    // POST {"title": "Live at Five", "count": 7}
    return Response::json(['title' => $title, 'count' => $count], 201);
}
```

`input()` reads from whichever body the request carries (the JSON body for JSON requests, form data otherwise) and falls back to the query string, so one controller can serve both kinds of client:

```php
$title = $request->input('title', 'Untitled');
```

A malformed JSON body throws `MalformedJsonException` when it is read, never silently returning an empty payload. It implements `HttpExceptionInterface`, so the pipeline renders it as a `400` wherever it is thrown --- while binding controller parameters or inside your controller. `wantsJson()` checks the `Accept` header, so you can pick a response format for the client:

```php
if ($request->wantsJson()) {
    return Response::json(['ok' => true]);
}
```

### Handling File Uploads

Uploaded files arrive as `Marko\Routing\Http\UploadedFile` objects, keyed like the form fields that sent them. `file()` returns one file, `files()` returns a list for multi-file inputs (`name="photos[]"`), and nested fields use dot-notation (`user.avatar`):

```php
use Marko\Routing\Attributes\Post;
use Marko\Routing\Http\Request;
use Marko\Routing\Http\Response;

#[Post('/profile/avatar')]
public function upload(Request $request): Response
{
    $avatar = $request->file('avatar');

    if ($avatar === null || !$avatar->isValid()) {
        return new Response('Upload an image', 422);
    }

    $extension = $avatar->guessExtension() ?? 'bin';
    $avatar->moveTo("/var/app/storage/avatars/" . bin2hex(random_bytes(16)) . ".$extension");

    return new Response('Saved', 201);
}
```

The client filename and media type are untrusted: `mimeType()` and `guessExtension()` inspect the file contents with `finfo` instead. Inputs submitted without a file are left out, so `hasFile()` is `false` for them. An upload that failed (for example, larger than `upload_max_filesize`) is still present but not valid; moving it, or moving any file twice, throws `UploadedFileException` naming the reason. Calling `file()` on a multi-file input also throws --- use `files()` there.

To store an upload through [`marko/media`](/docs/packages/media/), build a `Marko\Media\Value\UploadedFile` from it:

```php
use Marko\Media\Value\UploadedFile as MediaUpload;

$media = $mediaManager->upload(new MediaUpload(
    name: $avatar->clientFilename(),
    tmpPath: $avatar->tempPath(),
    mimeType: $avatar->mimeType(),
    size: $avatar->size(),
    extension: $avatar->guessExtension() ?? 'bin',
));
```

### Overriding Vendor Routes

Use [Preferences](/docs/packages/core/) to replace a vendor's controller:

```php title="MyPostController.php"
use Marko\Core\Attributes\Preference;
use Marko\Routing\Attributes\Get;
use Vendor\Blog\PostController;

#[Preference(replaces: PostController::class)]
class MyPostController extends PostController
{
    #[Get('/blog')]  // Your route takes over
    public function index(): Response
    {
        return new Response('My custom blog');
    }
}
```

### Disabling Routes

Explicitly remove an inherited route:

```php title="MyPostController.php"
use Marko\Routing\Attributes\DisableRoute;

#[Preference(replaces: PostController::class)]
class MyPostController extends PostController
{
    #[DisableRoute]  // Removes /blog/{slug} route
    public function show(
        string $slug,
    ): Response {
        // Method still exists but has no route
    }
}
```

### Route Conflicts

If two modules define the same route, Marko throws `RouteConflictException` at boot:

```
Route conflict detected for GET /products

Defined in:
  - Vendor\Catalog\ProductController::index()
  - App\Store\ProductController::list()

Resolution: Use #[Preference] to extend one controller,
or use #[DisableRoute] to remove one route.
```

### Errors and HTTP Exceptions

Throw `HttpException` from a controller or middleware to respond with an error status. There is no global `abort()` helper --- throw the exception explicitly:

```php title="app/blog/src/Controller/PostController.php"
use Marko\Routing\Exceptions\HttpException;

#[Get('/posts/{id}')]
public function show(int $id): Response
{
    $post = $this->postRepository->findById($id)
        ?? throw HttpException::notFound('Post not found.');

    return Response::json(['title' => $post->title]);
}
```

Named constructors cover the common cases: `badRequest()`, `unauthorized()`, `forbidden()`, `notFound()`, `methodNotAllowed(['GET', 'HEAD'])` (sets `Allow`), `conflict()`, and `tooManyRequests(retryAfter: 60)` (sets `Retry-After`). For anything else use the constructor, which accepts any `400`--`599` status plus headers and extra client-safe data:

```php
throw new HttpException(
    statusCode: 409,
    message: 'That slug is already taken.',
    data: ['field' => 'slug'],
);
```

The message is client-facing: it is sent as `message` in the response body. When omitted it defaults to the status reason phrase (`Not Found`, `Conflict`, ...).

#### Where errors are rendered

The middleware pipeline catches any exception implementing `Marko\Core\Exceptions\HttpExceptionInterface` **at the depth it was thrown** and turns it into a `Response`. Every middleware outside that point still runs, so CORS headers, security headers, session saving and similar decorations apply to error responses too. This works the same under PHP-FPM and [RoadRunner](/docs/packages/roadrunner/).

Any other throwable is not caught: it propagates out of `Router::handle()` to the installed error handler ([`marko/errors-simple`](/docs/packages/errors-simple/) or [`marko/errors-advanced`](/docs/packages/errors-advanced/)), which renders a `500`.

#### Framework exceptions

These framework exceptions implement `HttpExceptionInterface`, so they render without any glue code:

| Exception | Status | Body |
|---|---|---|
| `Marko\Routing\Exceptions\InvalidRouteParameterException` | `400` | `{"message": "Missing required parameter 'id' of type 'int'"}` |
| `Marko\Security\Exceptions\CsrfTokenMismatchException` | `419` | `{"message": "CSRF token mismatch."}` |
| `Marko\Validation\Exceptions\ValidationException` | `422` | `{"message": "The given data was invalid.", "errors": {"email": ["..."]}}` |
| `Marko\Database\Exceptions\EntityNotFoundException` | `404` | `{"message": "Not found."}` (never the entity class or ID) |
| `Marko\Database\Exceptions\UniqueConstraintViolationException` | `409` | `{"message": "Conflict."}` (never the constraint, SQL or values) |
| `Marko\Database\Exceptions\ForeignKeyConstraintViolationException` | `409` | `{"message": "Conflict."}` (never the constraint, SQL or values) |

Your own exceptions can implement the interface too --- useful for domain exceptions in packages that should not depend on `marko/routing`:

```php
use Marko\Core\Exceptions\HttpExceptionInterface;

class SubscriptionExpiredException extends RuntimeException implements HttpExceptionInterface
{
    public function getStatusCode(): int
    {
        return 402;
    }

    public function getHeaders(): array
    {
        return [];
    }

    public function getResponseData(): array
    {
        return ['message' => 'Your subscription has expired.'];
    }
}
```

Only `getResponseData()` reaches the client. The exception message, file, and trace are never rendered, in any environment.

#### JSON or HTML

`ExceptionRenderer` renders JSON when the request's `Accept` header contains `application/json` or a `+json` type (for example `application/problem+json`), or when there is no `Accept` header and the `Content-Type` is JSON. The body is the response data with a `message` key guaranteed. Otherwise it renders a minimal HTML page showing the status and message. The exception's headers are added in both cases.

#### Custom error pages

Replace `ExceptionRenderer` with a `#[Preference]` to render branded pages. Override `renderHtml()` to change only the HTML output, `renderJson()` for JSON, or `render()` for both:

```php title="app/web/src/Http/BrandedExceptionRenderer.php"
use Marko\Core\Attributes\Preference;
use Marko\Routing\Http\ExceptionRenderer;
use Marko\Routing\Http\Response;

#[Preference(replaces: ExceptionRenderer::class)]
class BrandedExceptionRenderer extends ExceptionRenderer
{
    protected function renderHtml(
        int $statusCode,
        array $data,
    ): Response {
        return Response::html(
            "<h1>Oops ($statusCode)</h1><p>" . htmlspecialchars($data['message']) . '</p>',
            $statusCode,
        );
    }
}
```

## CLI

Requires [`marko/cli`](/docs/packages/cli/) for the `marko` binary.

### Listing Routes

See all registered routes:

```bash
marko route:list
```

```
METHOD  PATH            ACTION                    MIDDLEWARE
GET     /               HelloController::index
GET     /blog           PostController::index
GET     /products       ProductController::index
GET     /blog/{id}      PostController::show
GET     /products/{id}  ProductController::show
POST    /products       ProductController::store
```

Routes are grouped by method (`GET`, `HEAD`, `POST`, `PUT`, `PATCH`, `DELETE`, `OPTIONS`, then any others) and listed in the order the router tries them --- see [Route Precedence](#route-precedence). Automatic HEAD and OPTIONS responses are not listed.

Filter by HTTP method or path:

```bash
marko route:list --method=POST
marko route:list --path=products
marko route:list --method=GET --path=blog
```

## API Reference

### Route Attributes

```php
#[Get(path: '/path', middleware: [])]
#[Post(path: '/path')]
#[Put(path: '/path')]
#[Patch(path: '/path')]
#[Delete(path: '/path')]
#[Head(path: '/path')]
#[Options(path: '/path')]
#[DisableRoute]
#[Middleware(MiddlewareClass::class)]
```

### Request

```php
class Request
{
    public function method(): string;
    public function path(): string;
    public function query(?string $key = null, mixed $default = null): mixed;
    public function post(?string $key = null, mixed $default = null): mixed;
    public function cookie(?string $key = null, mixed $default = null): mixed;
    public function body(): string;
    public function isJson(): bool;
    public function wantsJson(): bool;
    public function json(?string $key = null, mixed $default = null): mixed;
    public function input(?string $key = null, mixed $default = null): mixed;
    public function file(string $key): ?UploadedFile;
    public function files(?string $key = null): array;
    public function hasFile(string $key): bool;
    public function header(string $name, ?string $default = null): ?string;
    public function headers(): array;
    public function server(string $key): ?string;
    public function ip(): ?string;
    public function withRoute(string $controller, string $action): self;
    public function controller(): ?string;
    public function action(): ?string;
    public static function fromGlobals(): self;
}
```

`ip()` returns `REMOTE_ADDR` from the server bag (equivalent to `server('REMOTE_ADDR')`). `cookie()` reads from the request's `$_COOKIE` bag and mirrors the signature of `query()` and `post()`. `withRoute()` returns a new immutable `Request` with the matched controller class and action method attached; `controller()` and `action()` retrieve them. The router attaches route context before invoking middleware, which allows middleware (such as `AdminAuthMiddleware`) to inspect which controller method is handling the request. `withRoute()` carries the uploaded files and the decoded JSON body through unchanged.

| Method | Description |
| --- | --- |
| `isJson()` | `true` when `Content-Type` is `application/json` or ends in `+json` (e.g. `application/vnd.api+json`) |
| `wantsJson()` | `true` when the `Accept` header lists `application/json` or a `+json` type |
| `json($key, $default)` | The decoded JSON body, or one value from it by dot-notation key (`user.email`). Returns `[]` / `$default` for non-JSON requests and empty bodies. Throws `MalformedJsonException` for a malformed body |
| `input($key, $default)` | The JSON body for JSON requests, form data otherwise, then the query string. With no key, the query string merged with the body (body values win) |
| `file($key)` | One `UploadedFile`, or `null`. Dot-notation for nested fields. Throws `UploadedFileException` when the field holds several files |
| `files($key)` | With no key, every uploaded file keyed like its form field. With a key, that field's files as a list |
| `hasFile($key)` | Whether at least one file was uploaded for the field (check `isValid()` before using it) |

The constructor accepts a `files` argument (`array<string, UploadedFile|array>`), so a request with uploads can be built without superglobals. `fromGlobals()` normalizes `$_FILES`, including PHP's inverted `name[]` / `name[key]` layout for multi-file and nested inputs, and leaves out inputs submitted without a file (`UPLOAD_ERR_NO_FILE`).

### UploadedFile

```php
use Marko\Routing\Http\UploadedFile;

public function __construct(
    string $tempPath,
    string $clientFilename,
    string $clientMediaType,
    int $size,
    int $error = UPLOAD_ERR_OK,
)

public function clientFilename(): string;
public function clientMediaType(): string;
public function size(): int;
public function error(): int;
public function tempPath(): string;
public function isValid(): bool;
public function isMoved(): bool;
public function moveTo(string $targetPath): void;
public function stream(): mixed; // resource
public function contents(): string;
public function mimeType(): string;
public function guessExtension(): ?string;
```

`clientFilename()` and `clientMediaType()` are whatever the client sent --- never use them as a storage path or to decide what a file is. `mimeType()` detects the real type from the contents with `finfo`, and `guessExtension()` maps it to an extension (`null` when unknown). `isValid()` is `true` only for a successful upload (`UPLOAD_ERR_OK`) that has not been moved yet. `moveTo()` uses `move_uploaded_file()` under a web SAPI (PHP-FPM, Apache) and `rename()` elsewhere (CLI, tests, RoadRunner workers), and can be called once. Moving twice, moving a failed upload, moving into a missing directory, or reading a moved file throws `UploadedFileException` with the reason and a fix (for a failed upload, the `UPLOAD_ERR_*` name and the `php.ini` setting to change).

### Response

```php
class Response
{
    public function __construct(
        string $body = '',
        int $statusCode = 200,
        array $headers = [],
    );

    public function body(): string;
    public function statusCode(): int;
    public function headers(): array;
    public function cookies(): array;
    public function headerLines(): array;
    public function send(): void;
    public static function json(mixed $data, int $statusCode = 200): self;
    public static function html(string $html, int $statusCode = 200): self;
    public static function redirect(string $url, int $statusCode = 302): self;
    public function withHeader(string $name, string $value): static;
    public function withHeaders(array $headers): static;
    public function withStatus(int $statusCode): static;
    public function withCookie(Cookie $cookie): static;
    public function withoutBody(): static;
    public function isBodyOmitted(): bool;
}
```

`withoutBody()` empties the body and marks it omitted, keeping status, headers, cookies and the concrete class; the router applies it to every HEAD response. A subclass that writes its own output in `send()` should check `isBodyOmitted()` and send headers only.

`cookies()` returns the `Cookie` instances attached to the response; `headerLines()` returns the raw `"Name: value"` lines followed by one `Set-Cookie:` line per cookie, without making any SAPI calls --- `send()` uses it internally, and it's also useful for testing. `withHeader()` and `withHeaders()` merge into the existing headers (`withHeaders()` merges its argument over the current set). `withCookie()` replaces an existing cookie that matches on `(name, path, domain)`, or appends a new one otherwise. Every `with*()` method is marked `#[\NoDiscard]` and returns a clone via PHP's `clone` operator rather than `new static(...)`, so a `Response` subclass such as `StreamingResponse` survives decoration intact --- see [Decorating Responses](#decorating-responses). `Response` is deliberately not a `readonly class` for this reason: immutability is enforced by API design (private properties, no setters) rather than the `readonly` keyword.

`json()`, `html()`, and `redirect()` construct with `new self()`, not `new static()`, since a subclass such as `StreamingResponse` has a different constructor signature and cannot accept `(body:, statusCode:, headers:)`.

### Cookie

```php
use Marko\Routing\Http\Cookie;

public function __construct(
    string $name,
    string $value = '',
    ?int $expires = null,
    ?string $path = null,
    ?string $domain = null,
    bool $secure = false,
    bool $httpOnly = false,
    ?string $sameSite = null,
)

public function name(): string;
public function value(): string;
public function expires(): ?int;
public function path(): ?string;
public function domain(): ?string;
public function secure(): bool;
public function httpOnly(): bool;
public function sameSite(): ?string;
public function toSetCookieString(): string;
```

`value()` returns the raw value, before the URL-encoding applied in the `Set-Cookie` line.

`expires: null` or `expires: 0` omits the `Expires` attribute, producing a browser-session cookie. The value passed to `toSetCookieString()` is `rawurlencode()`d. The constructor throws `CookieException` for an invalid cookie name (control characters, whitespace, or separator characters such as `( ) < > @ , ; : \ " / [ ] ? = { }`), and also throws when `sameSite` is `'None'` without `secure: true`.

### MiddlewareInterface

```php
interface MiddlewareInterface
{
    public function handle(Request $request, callable $next): Response;
}
```

### HttpException

```php
use Marko\Routing\Exceptions\HttpException;

public function __construct(
    int $statusCode,             // 400-599, otherwise InvalidArgumentException
    string $message = '',        // defaults to the reason phrase
    array $headers = [],
    array $data = [],            // extra client-safe body fields
    string $context = '',
    string $suggestion = '',
    ?Throwable $previous = null,
)

public static function badRequest(string $message = ''): self;
public static function unauthorized(string $message = ''): self;
public static function forbidden(string $message = ''): self;
public static function notFound(string $message = ''): self;
public static function methodNotAllowed(array $allowedMethods, string $message = ''): self;
public static function conflict(string $message = ''): self;
public static function tooManyRequests(?int $retryAfter = null, string $message = ''): self;
```

`HttpException` implements `Marko\Core\Exceptions\HttpExceptionInterface`:

```php
interface HttpExceptionInterface extends Throwable
{
    public function getStatusCode(): int;
    public function getHeaders(): array;       // array<string, string>
    public function getResponseData(): array;  // client-safe; `message` is the human-readable text
}
```

### ExceptionRenderer

```php
use Marko\Routing\Http\ExceptionRenderer;

public function render(HttpExceptionInterface $exception, Request $request): Response;
public function wantsJson(Request $request): bool;
protected function renderJson(int $statusCode, array $data): Response;
protected function renderHtml(int $statusCode, array $data): Response;
```

`Marko\Routing\Http\HttpStatus::reasonPhrase(int $statusCode): string` returns the standard reason phrase (`419` → `Page Expired`).

### Parameter Resolution

The router resolves controller method parameters in priority order: route path params → request body (the JSON body for JSON requests, form data otherwise, via `Request::input()`) → query string → default value. Typed scalars (`int`, `float`, `bool`, `string`) are automatically cast. A required typed scalar with no matching source throws `InvalidRouteParameterException`, and a malformed JSON body throws `MalformedJsonException`; both implement `HttpExceptionInterface` and the pipeline renders them as a `400` response (see [Errors and HTTP Exceptions](#errors-and-http-exceptions)). Route path literals containing dots or other regex metacharacters are matched literally (via `preg_quote`). URL-encoded path segments are decoded once before matching.
