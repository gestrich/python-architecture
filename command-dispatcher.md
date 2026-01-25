# Command Dispatcher Pattern

## Entry Point: `__main__.py`

ClaudeChain uses a **command dispatcher** pattern with a single Python entry point:

```
python3 -m claudechain <command>
```

## Available Commands

| Command | Description | Used By |
|---------|-------------|---------|
| `prepare` | Setup before Claude Code execution | Main action |
| `finalize` | Commit changes and create PR | Main action |
| `prepare-summary` | Generate prompt for PR summary | Main action |
| `statistics` | Generate reports and statistics | Statistics action |

## Command Structure

**Dispatcher** (`scripts/claudechain/__main__.py`):
```python
def main():
    parser = argparse.ArgumentParser()
    subparsers = parser.add_subparsers(dest="command")

    parser_statistics = subparsers.add_parser("statistics")
    # ... other commands

    if args.command == "statistics":
        return cmd_statistics(args, gh)
    # ... route to other commands
```

**Command Implementation** (`scripts/claudechain/commands/statistics.py`):
```python
def cmd_statistics(args: argparse.Namespace, gh: GitHubActionsHelper) -> int:
    """Command logic - returns exit code (0 = success)"""
    # 1. Read environment variables
    # 2. Call business logic functions
    # 3. Write outputs via GitHub Actions helper
    # 4. Return exit code
```

## Benefits of Command Dispatcher

1. **Single entry point** - Easy to understand and debug
2. **Consistent interface** - All commands have same signature
3. **Shared utilities** - GitHubActionsHelper, config loading, etc.
4. **Easy extension** - Add new commands without touching existing ones
5. **Local testing** - Run commands outside GitHub Actions:
   ```bash
   PYTHONPATH=scripts python3 -m claudechain statistics
   ```

## Module Organization

### Python Package Structure

**Modern Structure** (`src/claudechain/` - Layered Architecture):
```
src/claudechain/
├── __init__.py
├── __main__.py              # Entry point
│
├── domain/                  # Layer 1: Core business logic
│   ├── models.py            # Data models
│   ├── exceptions.py        # Custom exceptions
│   └── config.py            # Configuration models
│
├── infrastructure/          # Layer 2: External dependencies
│   ├── git/
│   │   └── operations.py    # Git command wrappers
│   ├── github/
│   │   ├── operations.py    # GitHub CLI wrappers
│   │   └── actions.py       # GitHub Actions integration
│   └── filesystem/
│       └── operations.py    # File I/O operations
│
├── services/                # Layer 3: Business logic services
│   ├── __init__.py
│   ├── core/                # Foundational services
│   │   ├── __init__.py
│   │   ├── pr_service.py
│   │   ├── project_service.py
│   │   ├── reviewer_service.py
│   │   └── task_service.py
│   ├── composite/           # Higher-level orchestration
│   │   ├── __init__.py
│   │   ├── artifact_service.py
│   │   └── statistics_service.py
│   └── formatters/          # Service layer utilities
│       └── table_formatter.py
│
└── cli/                     # Layer 4: Presentation layer
    ├── commands/
    │   ├── prepare.py
    │   ├── finalize.py
    │   ├── prepare_summary.py
    │   └── statistics.py
    └── parser.py
```

**Legacy Structure** (`scripts/claudechain/` - For backward compatibility):
- Maintained during migration
- Will be removed in future releases
- Contains compatibility shims that import from new structure

## Module Responsibilities

**Commands** (`commands/`):
- Orchestrate workflow steps
- Handle argument parsing
- Read environment variables
- Call business logic functions
- Write GitHub Actions outputs
- Return exit codes

**Models** (`models.py`):
- Data structures (dataclasses, simple classes)
- Formatting methods (Slack, JSON, markdown)
- Properties and computed values
- No external dependencies (GitHub API, file I/O)

**Services** (`*_service.py`, `*_management.py`):
- Orchestrate business logic across multiple infrastructure components
- Gather and aggregate data from external sources (GitHub API, metadata storage)
- Implement use cases required by CLI commands
- Process and transform data using domain models
- Return model instances

**Operations** (`*_operations.py`):
- Low-level wrappers around external tools
- Git commands
- GitHub CLI (`gh`)
- Error handling for subprocess calls

**Utilities** (`config.py`, `github_actions.py`):
- Cross-cutting concerns
- Configuration loading
- GitHub Actions environment integration
- Template substitution

## Design Principles

1. **Separation of Concerns**: Each module has a single, clear responsibility
2. **Dependency Direction**: Commands depend on utilities, not vice versa
3. **Testability**: Models and collectors can be tested independently
4. **Reusability**: Multiple commands can use the same utilities
5. **Error Handling**: Each layer handles its own errors appropriately

## CLI Command Pattern

### Principle: Commands Use Explicit Parameters, Not Environment Variables

CLI command functions should receive explicit parameters and never read environment variables directly. The adapter layer in `__main__.py` is responsible for translating CLI arguments and environment variables into parameters.

### Architecture Layers

```
GitHub Actions (env vars) → __main__.py (adapter) → commands (params) → services (params)
```

Only `__main__.py` reads environment variables in the CLI layer.

### Anti-Pattern (❌ Avoid)

```python
# BAD: Command reads environment variables
def cmd_statistics(args: argparse.Namespace, gh: GitHubActionsHelper) -> int:
    repo = os.environ.get("GITHUB_REPOSITORY", "")  # Don't do this!
    config_path = args.config_path  # Mixing args and env is confusing
```

**Problems:**
- Hidden dependencies on environment variables
- Awkward local usage (must set env vars)
- Poor type safety with Namespace
- Harder to test

### Recommended Pattern (✅ Use This)

```python
# In cli/parser.py
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
    ...

# In __main__.py - Adapter layer
elif args.command == "statistics":
    return cmd_statistics(
        gh=gh,
        repo=args.repo or os.environ.get("GITHUB_REPOSITORY", ""),
        config_path=args.config_path or os.environ.get("CONFIG_PATH"),
        days_back=args.days_back or int(os.environ.get("STATS_DAYS_BACK", "30"))
    )
```

**Benefits:**
- ✅ Explicit dependencies: Function signature shows exactly what's needed
- ✅ Type safety: IDEs can autocomplete and type-check
- ✅ Easy testing: Just pass parameters, no environment mocking
- ✅ Works for both GitHub Actions and local development
- ✅ Discoverable: `--help` shows all options

## Services: Class-Based Architecture

### Convention: Class-Based Services with Dependency Injection

ClaudeChain uses a **class-based service architecture** where:

- **Services are classes** with constructor-based dependency injection
- **Services encapsulate** business logic for a specific domain area
- **Services are instantiated** once per CLI command execution
- **Services share state** through instance variables (repo, metadata_service, etc.)

### Why Class-Based Services?

**Benefits**:
1. **Consistency** - All services follow the same pattern
2. **Dependency Injection** - Dependencies injected via constructor, not passed as function parameters
3. **Testability** - Easier to mock dependencies and control test setup
4. **Reduced Repetition** - Eliminates redundant object creation (e.g., GitHubMetadataStore)
5. **State Management** - Services cache configuration and avoid redundant API calls
6. **Future Flexibility** - Easier to add methods or refactor without changing signatures everywhere

### Service Architecture Pattern

All services follow this design:

```python
class ServiceName:
    """Service for handling domain-specific operations"""

    def __init__(self, repo: str, metadata_service: MetadataService):
        """Initialize service with required dependencies"""
        self.repo = repo
        self.metadata_service = metadata_service

    def instance_method(self, param: str) -> Result:
        """Instance methods use self.repo and self.metadata_service"""
        # Use injected dependencies
        data = self.metadata_service.get_data()
        return process(data, param)

    @staticmethod
    def static_method(param: str) -> Result:
        """Static methods for pure functions with no state dependency"""
        return pure_computation(param)
```

### Service Instantiation in CLI Commands

Services are instantiated **once** at the beginning of each command execution:

```python
def cmd_prepare(args: argparse.Namespace, gh: GitHubActionsHelper) -> int:
    # === Get common dependencies ===
    repo = os.environ.get("GITHUB_REPOSITORY", "")

    # === Initialize infrastructure ===
    metadata_store = GitHubMetadataStore(repo)
    metadata_service = MetadataService(metadata_store)

    # === Initialize services ===
    project_service = ProjectService(repo)
    task_service = TaskService(repo, metadata_service)
    reviewer_service = ReviewerService(repo, metadata_service)
    pr_service = PRService(repo)

    # === Use services throughout command ===
    project = project_service.detect_project_from_pr(pr_number)
    task = task_service.find_next_available_task(spec_content)
    reviewer = reviewer_service.find_available_reviewer(reviewers, label, project)
    branch = pr_service.format_branch_name(project, task_index)

    # ... rest of command logic
```

### Service Instantiation Pattern

All CLI commands follow this three-section pattern:

1. **Get common dependencies** - Extract `repo` and other config from environment
2. **Initialize infrastructure** - Create `GitHubMetadataStore` and `MetadataService`
3. **Initialize services** - Instantiate all needed services with their dependencies

This pattern:
- Eliminates redundant service creation
- Makes dependencies explicit and testable
- Provides consistent structure across all commands
- Enables easy addition of new services

### Static vs Instance Methods

**Use instance methods when:**
- The method needs access to `self.repo` or `self.metadata_service`
- The method performs I/O or makes API calls
- The method needs to share state with other methods

**Use static methods when:**
- The method is a pure function with no state dependency
- The method performs pure computation or string manipulation
- The method can be called without instantiating the service

**Example**:
```python
class PRService:
    def __init__(self, repo: str):
        self.repo = repo

    def get_project_prs(self, project: str) -> List[dict]:
        """Instance method - needs self.repo"""
        return fetch_prs_from_github(self.repo, project)

    @staticmethod
    def parse_branch_name(branch: str) -> tuple[str, int]:
        """Static method - pure function, no state needed"""
        match = re.match(r"^([^/]+)/task-(\d+)", branch)
        return match.groups() if match else (None, None)
```
