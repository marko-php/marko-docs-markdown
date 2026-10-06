---
title: marko/validation
description: Input validation with built-in rules, string-based rule parsing, and custom rule support — validate data before it reaches your domain.
---

Input validation with built-in rules, string-based rule parsing, and custom rule support — validate data before it reaches your domain. Validation provides a `ValidatorInterface` for checking data against rules. Rules can be specified as pipe-delimited strings (`'required|email|max:255'`), arrays, or `RuleInterface` objects. Validation errors are collected into a `ValidationErrors` bag with per-field messages. Use `validateOrFail()` to throw on invalid data.

## Installation

```bash
composer require marko/validation
```

## Usage

### Basic Validation

Inject `ValidatorInterface` and pass data with rules:

```php
use Marko\Validation\Contracts\ValidatorInterface;

readonly class UserController
{
    public function __construct(
        private ValidatorInterface $validator,
    ) {}

    public function store(
        array $input,
    ): void {
        $errors = $this->validator->validate($input, [
            'name' => 'required|string|max:100',
            'email' => 'required|email',
            'age' => 'nullable|integer|min:18',
        ]);

        if ($errors->isNotEmpty()) {
            // handle errors
        }
    }
}
```

### Throw on Failure

Use `validateOrFail()` to throw `ValidationException` when data is invalid:

```php
$this->validator->validateOrFail($input, [
    'title' => 'required|string|max:200',
    'body' => 'required|string',
]);
// Throws ValidationException with errors() method
```

The `ValidationException` includes rich context --- call `errors()` to get the `ValidationErrors` bag, or `getContext()` and `getSuggestion()` for diagnostic details.

`ValidationException` implements `Marko\Core\Exceptions\HttpExceptionInterface`. When it escapes a controller, the routing pipeline renders it as **`422 Unprocessable Content`** with the field errors:

```json
{
    "message": "The given data was invalid.",
    "errors": {
        "email": ["The email field must be a valid email address."]
    }
}
```

API clients get JSON; browsers get a minimal HTML page with the message. See [Errors and HTTP Exceptions](/docs/packages/routing/#errors-and-http-exceptions). Redirecting back to an HTML form with errors and old input is not built in --- catch the exception in the controller for that.

### Quick Boolean Check

```php
if ($this->validator->passes($input, ['email' => 'required|email'])) {
    // data is valid
}

if ($this->validator->fails($input, ['email' => 'required|email'])) {
    // data is invalid
}
```

### Working with Errors

`ValidationErrors` provides methods for inspecting failures:

```php
$errors = $this->validator->validate($input, $rules);

$errors->has('email');        // true if email has errors
$errors->get('email');        // ['The email field must be a valid email.']
$errors->first('email');      // 'The email field must be a valid email.'
$errors->all();               // ['email' => [...], 'name' => [...]]
$errors->toFlatArray();       // ['email: ...', 'name: ...']
$errors->isEmpty();           // true if no errors
$errors->count();             // total error count across all fields
```

### Built-in Rules

| Rule | String Syntax | Description |
|------|--------------|-------------|
| Required | `required` | Must be present and non-empty |
| Nullable | `nullable` | Skips other rules if null/empty |
| String | `string` | Must be a string |
| Integer | `integer` | Must be an integer |
| Numeric | `numeric` | Must be numeric |
| Boolean | `boolean` | Must be boolean-like |
| Email | `email` | Must be a valid email |
| URL | `url` | Must be a valid URL |
| Alpha | `alpha` | Letters only |
| AlphaNumeric | `alpha_num` | Letters and numbers only |
| Min | `min:5` | Minimum value (numeric) or minimum length (string) or minimum count (array). Fails for a file --- use `min_size` |
| Max | `max:255` | Maximum value (numeric) or maximum length (string) or maximum count (array). Fails for a file --- use `max_size` |
| Between | `between:1,100` | Value range (numeric), length range (string), or count range (array). Fails for a file --- use `min_size`/`max_size` |
| In | `in:draft,published` | Must be one of the listed values; numeric strings are compared numerically |
| NotIn | `not_in:admin,root` | Must not be one of the listed values; numeric strings are compared numerically |
| Same | `same:other_field` | Must match another field |
| Different | `different:other_field` | Must differ from another field |
| Confirmed | `confirmed` | Must have a matching `_confirmation` field |
| Regex | `regex:/^\d{3}$/` | Must match the pattern |
| Date | `date` or `date:Y-m-d` | Must be a valid date |
| Array | `array` | Must be an array |
| File | `file` | Must be an uploaded file that arrived without an upload error |
| Image | `image` | Must be a JPEG, PNG, GIF or WebP image, judged from its contents (never SVG) |
| Mimes | `mimes:jpg,png,pdf` | The extension for the file's sniffed MIME type must be listed (`jpeg` and `jpg` are the same) |
| MimeTypes | `mimetypes:image/*,application/pdf` | The sniffed MIME type must be listed; `type/*` matches any subtype |
| MaxSize | `max_size:2048` | File size at most this many kilobytes (1 KB = 1024 bytes) |
| MinSize | `min_size:1` | File size at least this many kilobytes |

### Numeric-Aware Rules

`Min`, `Max`, and `Between` check the *value* numerically when the input is numeric (including numeric strings), and check *length* for non-numeric strings. This means `integer|min:18` accepts the string `"25"` as valid, and `max:100` rejects `"150"` even when sent as a string from a form:

```php
$errors = $this->validator->validate(
    ['age' => '25'],
    ['age' => 'required|integer|min:18'],
);
// No errors — "25" is numeric, 25.0 >= 18
```

`In` and `NotIn` compare numeric strings numerically: `in:1,2,3` accepts `"2"` even though it is not strictly identical to the integer `2`.

### Validating File Uploads

The validator checks an array, and uploaded files live apart from the form fields on the [request](/docs/packages/routing/#handling-file-uploads). Merge them into the data you validate:

```php title="app/profile/src/Controllers/AvatarController.php"
use Marko\Routing\Attributes\Post;
use Marko\Routing\Http\Request;
use Marko\Routing\Http\Response;
use Marko\Validation\Contracts\ValidatorInterface;

class AvatarController
{
    public function __construct(
        private ValidatorInterface $validator,
    ) {}

    #[Post('/profile/avatar')]
    public function upload(Request $request): Response
    {
        $this->validator->validateOrFail([...$request->input(), ...$request->files()], [
            'name' => 'required|string|max:100',
            'avatar' => 'required|file|image|max_size:2048',
            'resume' => 'nullable|file|mimes:pdf,docx|max_size:5120',
        ]);

        $avatar = $request->file('avatar');
        $avatar->moveTo('/var/www/storage/avatars/' . bin2hex(random_bytes(16)) . '.' . $avatar->guessExtension());

        return new Response('Saved');
    }
}
```

A failure throws `ValidationException`, which renders as a 422 with the file errors under their field names, exactly like any other rule. A file field overrides a form field of the same name in the merged array.

The type rules never trust what the client sent. `image`, `mimes` and `mimetypes` read the MIME type that `UploadedFile::mimeType()` detects from the file contents, so a text file named `avatar.png` with a `Content-Type` of `image/png` fails `image` and `mimes:png`. `image` accepts JPEG, PNG, GIF and WebP only: SVG can carry script, so allow it explicitly with `mimes:svg` or `mimetypes:image/svg+xml` when you serve it safely.

Every file rule fails a value that is not an uploaded file, and an upload that failed or was already moved. Failed uploads get a message that says why --- for example `The avatar file is larger than the server allows.` when the file exceeded `upload_max_filesize`.

Sizes are in kilobytes (1 KB = 1024 bytes) and the limits are inclusive. `max`, `min` and `between` keep their meaning for strings, numbers and arrays; given a file, they fail with a message that points to `max_size` / `min_size` (`The avatar field is a file: use max_size:2048 to limit its size in kilobytes.`). `max_size` and `min_size` without a number throw `InvalidArgumentException`, as do `mimes` and `mimetypes` without a list.

For a multi-file input (`name="photos[]"`), `array` and `max:5` check the list and count its files. Per-file rules (`photos.*`) are not supported yet.

The file rules depend only on `Marko\Core\Contracts\UploadedFileInterface`, which `Marko\Routing\Http\UploadedFile` implements, so `marko/validation` does not require `marko/routing`.

### Mixed Rule Formats

Rules can be strings, arrays, or `RuleInterface` instances:

```php
use Marko\Validation\Rules\Max;
use Marko\Validation\Rules\Required;

$this->validator->validate($input, [
    'name' => 'required|string',             // String format
    'email' => ['required', 'email'],        // Array of strings
    'bio' => [new Required(), new Max(500)], // Rule objects
]);
```

### Custom Rules

Implement `RuleInterface` for custom validation logic:

```php
use Marko\Validation\Contracts\RuleInterface;

class UniqueEmail implements RuleInterface
{
    public function passes(
        string $field,
        mixed $value,
        array $data,
    ): bool {
        // Check uniqueness against database
        return !$this->emailExists($value);
    }

    public function message(
        string $field,
        mixed $value,
    ): string {
        return "The $field has already been taken.";
    }
}

// Usage:
$this->validator->validate($input, [
    'email' => ['required', 'email', new UniqueEmail()],
]);
```

## API Reference

### ValidatorInterface

```php
interface ValidatorInterface
{
    public function validate(array $data, array $rules): ValidationErrors;
    public function validateOrFail(array $data, array $rules): void;
    public function passes(array $data, array $rules): bool;
    public function fails(array $data, array $rules): bool;
}
```

### RuleInterface

```php
interface RuleInterface
{
    public function passes(string $field, mixed $value, array $data): bool;
    public function message(string $field, mixed $value): string;
}
```

### ValidationErrors

```php
class ValidationErrors implements Countable, IteratorAggregate
{
    public function add(string $field, string $message): self;
    public function has(string $field): bool;
    public function get(string $field): array;
    public function first(string $field): ?string;
    public function all(): array;
    public function keys(): array;
    public function isEmpty(): bool;
    public function isNotEmpty(): bool;
    public function count(): int;
    public function toFlatArray(): array;
}
```

### ValidationException

```php
class ValidationException extends Exception implements HttpExceptionInterface
{
    public static function withErrors(ValidationErrors $errors): self;
    public function errors(): ValidationErrors;
    public function getContext(): string;
    public function getSuggestion(): string;
    public function getStatusCode(): int;      // 422
    public function getHeaders(): array;       // []
    public function getResponseData(): array;  // ['message' => ..., 'errors' => $errors->all()]
}
```
