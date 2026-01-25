---
name: cli-architecture
description: Structure CLI applications using command dispatcher pattern with layered architecture. Covers entry points, command routing, service instantiation, and explicit parameter flow. Use when building CLI tools or refactoring command structure.
user-invocable: true
argument-hint: "[command-name]"
---

# CLI Architecture

## When to Use This Skill

Activate this skill when:
- Building a multi-command CLI tool
- Structuring command entry points and routing
- Organizing CLI code with proper separation of concerns
- Implementing service instantiation patterns in commands
- Ensuring explicit parameter passing (not environment variables)
- Setting up a command dispatcher pattern

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
    """Orchestrate statistics workflow - returns exit code"""
    # 1. Get dependencies
    # 2. Initialize infrastructure
    # 3. Initialize services
    # 4. Use services
    # 5. Format output
    # 6. Return exit code
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
    """Orchestrate statistics workflow.

    Args:
        gh: GitHub Actions helper instance
        repo: GitHub repository (owner/name)
        config_path: Optional path to configuration file
        days_back: Days to look back for statistics

    Returns:
        Exit code (0 for success, 1 for failure)
    """
    try:
        # 1. Get common dependencies
        metadata_store = GitHubMetadataStore(repo)

        # 2. Initialize infrastructure layer
        metadata_service = MetadataService(metadata_store)

        # 3. Initialize service layer
        project_service = ProjectService(repo)
        stats_service = StatisticsService(repo, metadata_service)

        # 4. Use services to execute business logic
        projects = project_service.discover_projects()
        stats = stats_service.collect_statistics(projects, days_back)

        # 5. Format output
        output = format_statistics_table(stats)
        gh.set_output("statistics", output)

        # 6. Return exit code
        return 0

    except Exception as e:
        print(f"Error: {e}")
        return 1
```

**Command responsibilities:**
- Orchestrate workflow steps
- Handle argument parsing (via dispatcher)
- Initialize infrastructure and services
- Call business logic functions
- Write outputs (GitHub Actions, files, stdout)
- Return exit codes

## Module Organization

### Layered Directory Structure

```
src/mypackage/
├── __init__.py
├── __main__.py              # Entry point - command dispatcher

├── domain/                  # Layer 1: Core business logic
│   ├── models.py            # Data models (dataclasses)
│   ├── exceptions.py        # Custom exceptions
│   └── config.py            # Configuration models

├── infrastructure/          # Layer 2: External dependencies
│   ├── git/
│   │   └── operations.py    # Git command wrappers
│   ├── github/
│   │   ├── operations.py    # GitHub CLI wrappers
│   │   └── actions.py       # GitHub Actions integration
│   └── filesystem/
│       └── operations.py    # File I/O operations

├── services/                # Layer 3: Business logic services
│   ├── __init__.py
│   ├── core/
│   │   ├── pr_service.py
│   │   ├── project_service.py
│   │   └── task_service.py
│   ├── composite/           # Higher-level orchestration
│   │   └── statistics_service.py
│   └── formatters/          # Service utilities
│       └── table_formatter.py

└── cli/                     # Layer 4: Presentation layer
    ├── commands/
    │   ├── prepare.py
    │   ├── finalize.py
    │   └── statistics.py
    └── parser.py            # Argument parser configuration
```

### Module Responsibilities

**Commands** (`cli/commands/`):
- Orchestrate workflow steps
- Initialize infrastructure and services
- Handle command-line arguments
- Read environment variables (adapter layer only)
- Write outputs and return exit codes
- **NO business logic**

**Domain Models** (`domain/`):
- Data structures (dataclasses, simple classes)
- Formatting methods (Slack, JSON, markdown)
- Properties and computed values
- **NO external dependencies** (no GitHub API, no file I/O)

**Services** (`services/`):
- Orchestrate business logic across infrastructure
- Gather and aggregate data from external sources
- Implement use cases required by CLI commands
- Process and transform data using domain models
- Return model instances

**Infrastructure** (`infrastructure/`):
- Low-level wrappers around external tools
- Git commands, GitHub CLI, filesystem operations
- Error handling for subprocess calls
- External API integrations

## CLI Command Pattern: Explicit Parameters

### Principle: Commands Use Explicit Parameters, Not Environment Variables

CLI command functions should receive **explicit parameters** and never read environment variables directly. The adapter layer in `__main__.py` is responsible for translating CLI arguments and environment variables into parameters.

### Architecture Layers

```
GitHub Actions (env vars) → __main__.py (adapter) → commands (params) → services (params)
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
- ✅ **Works everywhere**: Both GitHub Actions and local development
- ✅ **Discoverable**: `--help` shows all options

## Service Instantiation Pattern

### Three-Section Pattern

All CLI commands follow this pattern for service instantiation:

```python
def cmd_prepare(
    gh: GitHubActionsHelper,
    repo: str,
    pr_number: int
) -> int:
    # ============================================================
    # 1. Get Common Dependencies
    # ============================================================
    # Extract configuration and shared values

    # ============================================================
    # 2. Initialize Infrastructure Layer
    # ============================================================
    metadata_store = GitHubMetadataStore(repo)
    metadata_service = MetadataService(metadata_store)

    # ============================================================
    # 3. Initialize Service Layer
    # ============================================================
    project_service = ProjectService(repo)
    task_service = TaskService(repo, metadata_service)
    reviewer_service = ReviewerService(repo, metadata_service)
    pr_service = PRService(repo)

    # ============================================================
    # 4. Use Services Throughout Command
    # ============================================================
    project = project_service.detect_project_from_pr(pr_number)
    task = task_service.find_next_available_task(spec_content)
    reviewer = reviewer_service.find_available_reviewer(reviewers, label, project)
    branch = pr_service.format_branch_name(project, task_index)

    # ... rest of command logic
    return 0
```

**Benefits of this pattern:**
- ✅ **Eliminates redundancy**: Services instantiated once, not per method call
- ✅ **Explicit dependencies**: All service dependencies visible upfront
- ✅ **Testable**: Easy to mock services for testing
- ✅ **Consistent structure**: All commands follow same pattern
- ✅ **Easy extension**: Adding new services is straightforward

## Class-Based Services

### Convention: Services Are Classes with Dependency Injection

Services are classes where:
- **Dependencies injected** via constructor
- **Services encapsulate** business logic for specific domain
- **Services instantiated once** per CLI command execution
- **Services share state** through instance variables

```python
class ProjectService:
    """Service for handling project operations"""

    def __init__(self, repo: str):
        """Initialize service with required dependencies"""
        self.repo = repo

    def detect_project_from_pr(self, pr_number: int) -> Project:
        """Instance method uses self.repo"""
        labels = get_pr_labels(self.repo, pr_number)
        return self._parse_project_from_labels(labels)

    @staticmethod
    def parse_branch_name(branch: str) -> tuple[str, int]:
        """Static method for pure functions with no state dependency"""
        match = re.match(r"^([^/]+)/task-(\d+)", branch)
        return match.groups() if match else (None, None)
```

### Static vs Instance Methods

**Use instance methods when:**
- Method needs access to injected dependencies (`self.repo`, `self.metadata_service`)
- Method performs I/O or makes API calls
- Method needs to share state with other methods

**Use static methods when:**
- Method is a pure function with no state dependency
- Method performs pure computation or string manipulation
- Method can be called without instantiating the service

## Design Principles

### Separation of Concerns

Each layer has a single, clear responsibility:

1. **CLI Layer**: Argument parsing, environment variables, command routing
2. **Service Layer**: Business logic orchestration
3. **Infrastructure Layer**: External system integration
4. **Domain Layer**: Core business entities and rules

### Dependency Direction

Higher layers depend on lower layers, never the reverse:

```
CLI → Services → Infrastructure → Domain
```

Commands depend on services, services depend on infrastructure, but infrastructure never depends on services.

### Testability

- **Domain models**: Test independently with no mocks
- **Infrastructure**: Mock external systems (GitHub API, filesystem)
- **Services**: Mock infrastructure dependencies
- **Commands**: Mock services for unit tests, use real services for integration tests

### Error Handling

Each layer handles errors appropriately:
- **Commands**: Catch exceptions, log errors, return exit codes
- **Services**: Raise domain exceptions with clear messages
- **Infrastructure**: Raise infrastructure exceptions for external failures
- **Domain**: Validate data in constructors/factories

## Related Patterns

- **Command Pattern**: Encapsulates requests as objects
- **Dependency Injection**: Constructor-based dependency management
- **Service Layer Pattern**: Business logic orchestration
- **Repository Pattern**: Data access abstraction

## Related Skills

- **creating-services**: Detailed guidance on implementing service classes
- **dependency-injection**: Constructor patterns and configuration flow
- **identifying-layer-placement**: Understanding which layer code belongs in
- **domain-modeling**: Creating rich domain models used by services

## Further Reading

- [Martin Fowler - Service Layer](https://martinfowler.com/eaaCatalog/serviceLayer.html)
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Python Application Layouts](https://realpython.com/python-application-layouts/)
