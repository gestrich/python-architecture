---
name: cli-architecture
description: Structure CLI applications using command dispatcher pattern. Covers entry points, command routing, argument parsing, and explicit parameter flow. Use when building CLI tools or refactoring command structure.
user-invocable: true
argument-hint: "[command-name]"
---

# CLI Architecture

## When to Use This Skill

Activate this skill when:
- Building a multi-command CLI tool
- Structuring command entry points and routing
- Setting up argument parsing and command dispatching
- Ensuring explicit parameter passing to commands
- Organizing CLI command handlers

## Quick Reference

### Command Dispatcher Pattern
```python
# Entry point: python -m mypackage <command>

# __main__.py - Single entry point dispatcher
def main():
    parser = argparse.ArgumentParser()
    subparsers = parser.add_subparsers(dest="command")

    parser_statistics = subparsers.add_parser("statistics")
    parser_statistics.add_argument("--repo", help="GitHub repository")

    args = parser.parse_args()

    if args.command == "statistics":
        return cmd_statistics(
            gh=gh,
            repo=args.repo or os.environ.get("GITHUB_REPOSITORY", ""),
            days_back=args.days_back or 30
        )

# cli/commands/statistics.py - Command with explicit parameters
def cmd_statistics(
    gh: GitHubActionsHelper,
    repo: str,
    days_back: int = 30
) -> int:
    """Execute statistics command - returns exit code"""
    # Command orchestrates the workflow
    # Calls business logic functions/services
    # Formats and writes output
    # Returns exit code
    return 0
```

## Architecture Pattern

### Entry Point: Single Dispatcher

Use `__main__.py` as a single Python entry point that routes to command functions:

```
python3 -m mypackage <command>
```

**Benefits:**
- ✅ Single entry point - easy to understand and debug
- ✅ Consistent interface - all commands have same signature
- ✅ Shared utilities - config loading, helpers available to all
- ✅ Easy extension - add new commands without touching existing ones
- ✅ Local testing - run commands outside production environment

### Available Commands Table

Document your commands in a table format:

| Command | Description | Used By |
|---------|-------------|---------|
| `prepare` | Setup before execution | Main action |
| `finalize` | Commit changes and create PR | Main action |
| `statistics` | Generate reports | Statistics action |

## Command Structure

### Dispatcher Pattern

**`__main__.py`** - Routes to command implementations:

```python
def main():
    parser = argparse.ArgumentParser()
    subparsers = parser.add_subparsers(dest="command")

    # Define subparsers for each command
    parser_statistics = subparsers.add_parser("statistics")
    parser_statistics.add_argument("--repo", help="GitHub repository")
    parser_statistics.add_argument("--config-path", help="Config file path")
    parser_statistics.add_argument("--days-back", type=int, default=30)

    args = parser.parse_args()

    # Route to command implementations
    if args.command == "statistics":
        return cmd_statistics(args, gh)
    elif args.command == "prepare":
        return cmd_prepare(args, gh)
    # ... other commands
```

### Command Implementation Pattern

**Commands orchestrate workflow steps and return exit codes:**

```python
def cmd_statistics(
    gh: GitHubActionsHelper,
    repo: str,
    config_path: Optional[str] = None,
    days_back: int = 30
) -> int:
    """Execute statistics command.

    Args:
        gh: GitHub Actions helper instance
        repo: GitHub repository (owner/name)
        config_path: Optional path to configuration file
        days_back: Days to look back for statistics

    Returns:
        Exit code (0 for success, 1 for failure)
    """
    try:
        # 1. Initialize dependencies (services, helpers, etc.)
        stats_service = create_statistics_service(repo)

        # 2. Execute business logic
        stats = stats_service.collect_statistics(days_back)

        # 3. Format and write output
        output = format_statistics_table(stats)
        gh.set_output("statistics", output)

        # 4. Return exit code
        return 0

    except Exception as e:
        print(f"Error: {e}")
        return 1
```

**Command responsibilities:**
- Accept explicit parameters (not environment variables)
- Initialize dependencies needed for the command
- Execute business logic
- Handle errors and exceptions
- Write outputs (stdout, files, logs)
- Return exit codes (0 for success, non-zero for failure)

## Module Organization

### CLI Directory Structure

```
src/mypackage/
├── __init__.py
├── __main__.py              # Entry point - command dispatcher
│
└── cli/                     # CLI layer
    ├── __init__.py
    ├── commands/            # Command implementations
    │   ├── __init__.py
    │   ├── prepare.py       # cmd_prepare()
    │   ├── finalize.py      # cmd_finalize()
    │   └── statistics.py    # cmd_statistics()
    │
    └── parser.py            # Argument parser configuration (optional)
```

### Module Responsibilities

**`__main__.py`** (Entry point):
- Parse command-line arguments
- Route to command implementations
- Translate environment variables to parameters (adapter layer)
- Call command functions with explicit parameters

**`cli/commands/`** (Command implementations):
- Accept explicit parameters (typed function arguments)
- Initialize dependencies needed for execution
- Orchestrate workflow by calling business logic
- Handle errors and exceptions
- Write outputs (stdout, files, logs)
- Return exit codes
- **NO environment variable reading** (only in `__main__.py`)
- **NO business logic** (delegate to services/functions)

**`cli/parser.py`** (Optional):
- Define argument parsers for each command
- Configure subparsers and arguments
- Separate parser configuration from routing logic

## CLI Command Pattern: Explicit Parameters

### Principle: Commands Use Explicit Parameters, Not Environment Variables

CLI command functions should receive **explicit parameters** and never read environment variables directly. The adapter layer in `__main__.py` is responsible for translating CLI arguments and environment variables into parameters.

### Architecture Layers

```
Environment/Args → __main__.py (adapter) → commands (params) → business logic
```

**Only `__main__.py` reads environment variables in the CLI layer.**

### Anti-Pattern (❌ Avoid)

```python
# BAD: Command reads environment variables
def cmd_statistics(args: argparse.Namespace, gh: GitHubActionsHelper) -> int:
    repo = os.environ.get("GITHUB_REPOSITORY", "")  # Don't do this!
    config_path = args.config_path  # Mixing args and env is confusing

    metadata_store = GitHubMetadataStore(repo)
    # ... rest of command
```

**Problems:**
- ❌ Hidden dependencies on environment variables
- ❌ Awkward local usage (must set env vars manually)
- ❌ Poor type safety with Namespace objects
- ❌ Harder to test (must mock environment)
- ❌ Mixing argument sources (args vs env) is confusing

### Recommended Pattern (✅ Use This)

```python
# In cli/parser.py - Define arguments
parser_statistics = subparsers.add_parser("statistics")
parser_statistics.add_argument("--repo", help="GitHub repository (owner/name)")
parser_statistics.add_argument("--config-path", help="Path to configuration file")
parser_statistics.add_argument("--days-back", type=int, default=30)

# In cli/commands/statistics.py - Pure function with explicit parameters
def cmd_statistics(
    gh: GitHubActionsHelper,
    repo: str,
    config_path: Optional[str] = None,
    days_back: int = 30
) -> int:
    """Orchestrate statistics workflow.

    Args:
        gh: GitHub Actions helper instance
        repo: GitHub repository (owner/name)
        config_path: Optional path to configuration file
        days_back: Days to look back for statistics

    Returns:
        Exit code (0 for success, 1 for failure)
    """
    # Use parameters directly - no environment access!
    metadata_store = GitHubMetadataStore(repo)
    # ... rest of command
    return 0

# In __main__.py - Adapter layer translates env vars to parameters
elif args.command == "statistics":
    return cmd_statistics(
        gh=gh,
        repo=args.repo or os.environ.get("GITHUB_REPOSITORY", ""),
        config_path=args.config_path or os.environ.get("CONFIG_PATH"),
        days_back=args.days_back or int(os.environ.get("STATS_DAYS_BACK", "30"))
    )
```

**Benefits:**
- ✅ **Explicit dependencies**: Function signature shows exactly what's needed
- ✅ **Type safety**: IDEs can autocomplete and type-check
- ✅ **Easy testing**: Just pass parameters, no environment mocking
- ✅ **Works everywhere**: CI environments, scripts, and local development
- ✅ **Discoverable**: `--help` shows all options


## Design Principles

### Single Entry Point

Use `__main__.py` as the single entry point that routes to all commands:
- ✅ Easier to understand and debug
- ✅ Consistent interface across all commands
- ✅ Simpler to test (just call command functions)
- ✅ Works in any environment (local, CI, production)

### Explicit Parameters

Commands receive explicit typed parameters, not environment variables:
- ✅ Clear function signatures show what's needed
- ✅ IDE autocomplete and type checking
- ✅ Easy to test without mocking environment
- ✅ Self-documenting code

### Thin Command Layer

Commands orchestrate but don't implement business logic:
- ✅ Commands call services/functions for actual work
- ✅ Easy to test business logic independently
- ✅ Commands focus on CLI concerns (args, output, exit codes)
- ✅ Business logic reusable outside CLI context

### Error Handling

Commands are responsible for error handling:
- Catch exceptions from business logic
- Log errors with appropriate detail
- Return meaningful exit codes
- Provide user-friendly error messages

## Related Skills

- **creating-services**: How to implement services called by CLI commands
- **dependency-injection**: How commands receive and use dependencies
- **testing-services**: Testing strategies for CLI commands and business logic

## Related Patterns

- **Command Pattern**: Encapsulates requests as objects with execute methods
- **Adapter Pattern**: `__main__.py` adapts environment variables to parameters
- **Template Method**: Commands follow consistent structure (init, execute, output, return)

## Further Reading

- [Click - Python CLI Framework](https://click.palletsprojects.com/)
- [argparse - Command-line parsing](https://docs.python.org/3/library/argparse.html)
- [Python Application Layouts](https://realpython.com/python-application-layouts/)
