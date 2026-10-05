---
title: marko/cli
description: Global command-line tool for running commands in any Marko project directory.
---

Global command-line tool --- run `marko` from any project directory to execute commands. Install once globally, use in any Marko project. The CLI finds your project root, boots the application, and runs commands registered by your modules. No per-project CLI setup needed.

## Installation

```bash
composer global require marko/cli
```

Ensure Composer's global bin directory is in your PATH.

## Usage

### Running Commands

From any directory within a Marko project:

```bash
marko list                  # Show all available commands
marko module:list           # List installed modules
marko discovery:cache       # Compile the discovery cache (run on deploy)
marko discovery:clear       # Remove the discovery cache
marko cache:clear           # Clear cache (if cache module installed)
marko db:migrate            # Run migrations (if database module installed)
```

### Creating Commands

Register commands in your modules using the `#[Command]` attribute:

```php
use Marko\Core\Attributes\Command;
use Marko\Core\Command\CommandInterface;
use Marko\Core\Command\Input;
use Marko\Core\Command\Output;

/** @noinspection PhpUnused */
#[Command(name: 'greet', description: 'Say hello')]
class GreetCommand implements CommandInterface
{
    public function execute(Input $input, Output $output): int
    {
        $name = $input->getArgument(0) ?? 'World';
        $output->writeLine("Hello, $name!");

        return 0; // Exit code
    }
}
```

> **IDE Note:** PhpStorm may report command classes as "unused" since they're discovered via attributes rather than direct instantiation. The `@noinspection PhpUnused` annotation suppresses this false positive.

Run it:

```bash
marko greet
# Hello, World!

marko greet Mark
# Hello, Mark!
```

### Arguments and Options

`Input` parses the command line once into **positional arguments** and **options**. Options can appear before, between, or after arguments --- `getArgument()` only ever sees the positionals.

| Syntax | Result |
|---|---|
| `--queue=emails` | `getOption('queue')` returns `'emails'` |
| `--queue emails` | `getOption('queue')` returns `'emails'` (the next token is the value when it doesn't start with `-`) |
| `--queue` at the end, or followed by another option | `getOption('queue')` returns `'true'` |
| `--queue a --queue b` | `getOption('queue')` returns `'b'` (the last value); `getOptionValues('queue')` returns `['a', 'b']` |
| `-p=8000`, `-p 8000` | `getOption('p')` returns `'8000'` |
| `-d` | `getOption('d')` returns `'true'` |
| `--` | Ends option parsing --- every token after it is a positional argument, even if it starts with `-` |

Single-character names refer to short options (`-d`), longer names to long options (`--detach`). `hasOption('d')` does not match `--detach`, so check both when a command offers both forms.

### Declaring Flags

A bare option followed by a non-dash token takes that token as its value. For a value-less boolean flag that is wrong: in `marko queue:retry --force 5`, an undeclared `--force` would swallow `5`. Declare boolean flags on the `#[Command]` attribute so they never consume the next token:

```php
use Marko\Core\Attributes\Command;
use Marko\Core\Command\CommandInterface;
use Marko\Core\Command\Input;
use Marko\Core\Command\Output;

#[Command(name: 'report:send', description: 'Send a report', flags: ['force', 'f'])]
class SendReportCommand implements CommandInterface
{
    public function execute(Input $input, Output $output): int
    {
        $reportId = $input->getArgument(0);
        $force = $input->hasOption('force') || $input->hasOption('f');
        $recipients = $input->getOptionValues('to');

        // ...

        return 0;
    }
}
```

With `force` declared, `marko report:send --force 42 --to a@example.com --to b@example.com` and `marko report:send 42 --force --to=a@example.com --to=b@example.com` behave the same: argument `0` is `'42'`, `force` is set, and `to` has both addresses. Every option that is not declared as a flag is treated as a value option.

Declared flags are applied by the command runner, so they take effect when the command runs through `marko` or the `CommandRunner`. In unit tests that call `execute()` directly, pass the same flags to `Input`: `new Input(['marko', 'report:send', '--force', '42'], ['force', 'f'])`.

### Command Namespacing

Group related commands with colons:

```bash
marko db:migrate
marko db:rollback
marko db:seed
marko cache:clear
marko cache:warmup
```

### How It Works

1. CLI searches upward for a Marko project (looks for `vendor/marko/core`)
2. Loads the project's autoloader
3. Boots the application
4. Runs the requested command

The CLI itself has no commands --- all commands come from modules in your project.

## API Reference

### CommandInterface

```php
use Marko\Core\Command\CommandInterface;
use Marko\Core\Command\Input;
use Marko\Core\Command\Output;

interface CommandInterface
{
    public function execute(Input $input, Output $output): int;
}
```

### Input

```php
use Marko\Core\Command\Input;

class Input
{
    /**
     * @param array<int, string> $arguments Raw argv (index 0 = script, index 1 = command)
     * @param list<string> $flags Option names that never take a value
     */
    public function __construct(array $arguments, array $flags = []);
    public function withFlags(array $flags): self;
    public function getCommand(): ?string;
    public function getArgument(int $index): ?string;   // Positional arguments only
    public function hasArgument(int $index): bool;
    public function getArguments(): array;              // list<string> of positional arguments
    public function getOption(string $name): ?string;   // Last value, 'true' for a bare flag, null if absent
    public function getOptionValues(string $name): array; // list<string> of every value given
    public function hasOption(string $name): bool;
}
```

### Output

```php
use Marko\Core\Command\Output;

class Output
{
    public function write(string $text): void;
    public function writeLine(string $text): void;
}
```

### Command Attribute

```php
use Marko\Core\Attributes\Command;

#[Command(
    name: 'namespace:name',
    description: 'What it does',
    aliases: ['alias'],          // Optional alternative names
    flags: ['force', 'f'],       // Optional value-less options that never consume the next token
)]
```

### Exceptions

The CLI provides rich error messages with context and suggestions via `CliException` and its subclasses:

- `ProjectNotFoundException` --- thrown when no Marko project is found in the current directory or any parent directory.
- `CommandNotFoundException` --- thrown when the requested command does not exist. Suggests running `marko list`.
- `BootstrapException` --- thrown when the application fails to boot.

All CLI exceptions extend `CliException`, which includes `getContext()` and `getSuggestion()` methods for helpful error output.
