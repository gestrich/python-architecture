---
name: dependency-injection
description: Apply constructor-based dependency injection with explicit configuration flow. Covers required dependencies, fail-fast principles, and avoiding optional parameters with default factories. Use when designing service constructors or managing dependencies.
user-invocable: true
argument-hint: ""
---

# Dependency Injection

## When to Use This Skill

Activate this skill when:
- Designing service constructors with dependencies
- Managing configuration flow from entry points to services
- Deciding between required vs optional parameters
- Handling missing or invalid values (fail-fast vs silent defaults)
- Structuring code to avoid reading environment variables in services

## Quick Reference

### Constructor-Based Dependency Injection Pattern

```python
# ✅ GOOD: Required dependencies via constructor
class StatisticsService:
    def __init__(
        self,
        repo: str,
        metadata_service: MetadataService,
        project_repository: ProjectRepository,  # ✅ Required, no default
        base_branch: str = "main"
    ):
        """Initialize the statistics service

        Args:
            repo: GitHub repository (owner/name)
            metadata_service: MetadataService instance for accessing metadata
            project_repository: ProjectRepository instance for loading project data
            base_branch: Base branch to fetch specs from (default: "main")
        """
        self.repo = repo
        self.metadata_service = metadata_service
        self.project_repository = project_repository  # ✅ Direct assignment
        self.base_branch = base_branch

# ✅ Explicit dependency creation at call site
metadata_store = GitHubMetadataStore(repo)
metadata_service = MetadataService(metadata_store)
project_repository = ProjectRepository(repo)
statistics_service = StatisticsService(
    repo,
    metadata_service,
    project_repository,  # ✅ Explicitly passed
    base_branch
)
```

## Principle: Avoid Optional Dependencies with Default Factories

Optional parameters with default factory patterns (e.g., `param: Optional[Type] = None` with `self.param = param or DefaultType()`) are a code smell that hides dependencies and makes code harder to reason about.

## Anti-Pattern: Optional Dependencies (❌ Avoid)

```python
# ❌ BAD: Optional dependency with default factory
class StatisticsService:
    def __init__(
        self,
        repo: str,
        metadata_service: MetadataService,
        base_branch: str = "main",
        project_repository: Optional[ProjectRepository] = None  # ❌ Optional
    ):
        self.repo = repo
        self.metadata_service = metadata_service
        self.base_branch = base_branch
        # ❌ Hidden object creation - magic happens here
        self.project_repository = project_repository or ProjectRepository(repo)
```

**Problems with this approach:**
- **Hidden dependencies**: Not obvious that `ProjectRepository` is required
- **Configuration ambiguity**: Unclear if `None` means "use default" or "not needed"
- **Testing confusion**: Tests still need to pass mocks explicitly, so optionality provides no benefit
- **Dangerous defaults**: If caller forgets to pass it, they get an unintended default silently
- **Hard to reason about**: "Is this actually optional or just convenient?"

## Recommended Pattern: Required Dependencies (✅ Use This)

```python
# ✅ GOOD: Required dependency - explicit and clear
class StatisticsService:
    def __init__(
        self,
        repo: str,
        metadata_service: MetadataService,
        project_repository: ProjectRepository,  # ✅ Required, no default
        base_branch: str = "main"
    ):
        """Initialize the statistics service

        Args:
            repo: GitHub repository (owner/name)
            metadata_service: MetadataService instance for accessing metadata
            project_repository: ProjectRepository instance for loading project data
            base_branch: Base branch to fetch specs from (default: "main")
        """
        self.repo = repo
        self.metadata_service = metadata_service
        self.project_repository = project_repository  # ✅ Direct assignment
        self.base_branch = base_branch
```

**In production code:**
```python
# ✅ Explicit dependency creation at call site
metadata_store = GitHubMetadataStore(repo)
metadata_service = MetadataService(metadata_store)
project_repository = ProjectRepository(repo)  # ✅ Caller creates it
statistics_service = StatisticsService(
    repo,
    metadata_service,
    project_repository,  # ✅ Explicitly passed
    base_branch
)
```

**In tests:**
```python
# ✅ Tests explicitly pass mocks
mock_metadata_service = Mock()
mock_project_repo = Mock()  # ✅ Must create mock
service = StatisticsService(
    "owner/repo",
    mock_metadata_service,
    mock_project_repo,  # ✅ Explicitly mocked
    base_branch="main"
)
```

### Benefits of Required Dependencies

✅ **Explicit dependencies**: Constructor signature shows exactly what the class needs

✅ **No hidden behavior**: No magic object creation - what you see is what you get

✅ **Clear ownership**: Caller is responsible for creating dependencies, not the class

✅ **Better testing**: Forces explicit mocks, making tests clearer

✅ **Fail fast**: Missing dependency causes immediate error, not silent defaults

✅ **Self-documenting**: Code clearly shows all required collaborators

### When Optional Parameters Are Acceptable

Optional parameters are acceptable only when they represent:

1. **Truly optional behavior**: Feature flags or optional enhancements
   ```python
   def send_notification(message: str, priority: Optional[str] = None):
       # Priority is genuinely optional - default is "normal"
   ```

2. **Optional filters or constraints**: Narrowing a query
   ```python
   def collect_statistics(config_path: Optional[str] = None):
       # None means "all projects", not "use default config"
   ```

3. **Backward compatibility**: When adding new parameters to existing APIs
   ```python
   def legacy_method(required: str, new_param: Optional[str] = None):
       # Added for backward compatibility with existing callers
   ```

**Never use optional for:**
- Dependencies/services (use required parameters)
- Configuration that should flow from the caller
- Cases where `None` has ambiguous meaning

## Principle: Fail Fast - Avoid Silent Failures

When a value is missing, invalid, or unexpected, **raise an exception** rather than silently returning empty data structures or default values. Missing values are often errors that should be surfaced, not hidden.

## Anti-Pattern: Silent Failures (❌ Avoid)

```python
# ❌ BAD: Silently returning empty data when something is wrong
def get_user_config(user_id: str) -> dict:
    result = database.query(f"SELECT * FROM configs WHERE user_id = '{user_id}'")
    if not result:
        return {}  # ❌ Caller won't know if user doesn't exist or has no config

def parse_task_list(content: str) -> list[Task]:
    if not content:
        return []  # ❌ Was the file empty, or did we fail to read it?

def get_reviewer(pr_number: int) -> Optional[str]:
    pr = api.get_pull_request(pr_number)
    if pr is None:
        return None  # ❌ Did PR not exist, or was there an API error?
    return pr.get("reviewer")  # ❌ Returns None if key missing - is that valid?

# ❌ BAD: Using .get() with defaults to mask missing required fields
def process_config(config: dict) -> Settings:
    return Settings(
        timeout=config.get("timeout", 30),  # ❌ Is 30 the right default, or should timeout be required?
        retries=config.get("retries", 3),   # ❌ Masks missing configuration
        api_key=config.get("api_key", ""),  # ❌ Empty string will fail later with confusing error
    )
```

**Problems with silent failures:**
- **Hidden bugs**: Caller proceeds with empty/default data, causing failures elsewhere
- **Difficult debugging**: Error surfaces far from the root cause
- **Ambiguous meaning**: Can't distinguish "no data" from "error fetching data"
- **False confidence**: Code appears to work but produces incorrect results
- **Lost context**: By the time the error manifests, the original cause is unknown

## Recommended Pattern: Fail Fast (✅ Use This)

```python
# ✅ GOOD: Raise exceptions for abnormal cases
def get_user_config(user_id: str) -> dict:
    result = database.query(f"SELECT * FROM configs WHERE user_id = '{user_id}'")
    if not result:
        raise UserNotFoundError(f"No configuration found for user: {user_id}")
    return result

def parse_task_list(content: str) -> list[Task]:
    if not content:
        raise ValueError("Cannot parse empty content - file may be missing or unreadable")
    # Parse and return tasks...

def get_reviewer(pr_number: int) -> str:
    pr = api.get_pull_request(pr_number)
    if pr is None:
        raise PullRequestNotFoundError(f"PR #{pr_number} not found")
    if "reviewer" not in pr:
        raise InvalidPRDataError(f"PR #{pr_number} has no reviewer assigned")
    return pr["reviewer"]

# ✅ GOOD: Validate required fields explicitly
def process_config(config: dict) -> Settings:
    required_fields = ["timeout", "retries", "api_key"]
    missing = [f for f in required_fields if f not in config]
    if missing:
        raise ConfigurationError(f"Missing required config fields: {missing}")

    if not config["api_key"]:
        raise ConfigurationError("api_key cannot be empty")

    return Settings(
        timeout=config["timeout"],
        retries=config["retries"],
        api_key=config["api_key"],
    )
```

### When Empty/Default Values Are Acceptable

Empty or default values are appropriate only when they represent **valid business states**, not error conditions:

```python
# ✅ OK: Empty list means "no items match filter" (valid state)
def find_open_tasks(tasks: list[Task]) -> list[Task]:
    return [t for t in tasks if not t.is_completed]

# ✅ OK: Optional field that genuinely may not exist
def get_pr_description(pr: PullRequest) -> Optional[str]:
    return pr.description  # None means "no description provided" - valid state

# ✅ OK: Default for truly optional behavior
def format_output(data: dict, include_timestamps: bool = False) -> str:
    # include_timestamps is optional enhancement, False is sensible default
    ...
```

### Distinguishing Valid Empty States from Errors

Ask yourself: **"Would an empty result surprise the caller and cause problems downstream?"**

| Scenario | Empty/Default OK? | Recommended Approach |
|----------|------------------|---------------------|
| Query returns no matching records | ✅ Yes | Return empty list |
| Required config file is missing | ❌ No | Raise `FileNotFoundError` |
| API call fails | ❌ No | Raise exception with details |
| Optional field not provided | ✅ Yes | Return `None` or default |
| Required field missing from response | ❌ No | Raise `ValueError` |
| User has no assigned tasks | ✅ Yes | Return empty list |
| User account doesn't exist | ❌ No | Raise `UserNotFoundError` |

### Benefits of Fail-Fast

✅ **Immediate feedback**: Errors caught at the source, not downstream

✅ **Clear error messages**: Exception describes exactly what went wrong

✅ **Easier debugging**: Stack trace points to the actual problem

✅ **Explicit contracts**: Function signature and behavior are unambiguous

✅ **No hidden state**: Caller always knows if operation succeeded

✅ **Prevents data corruption**: Invalid states don't propagate through the system

### Let Exceptions Propagate in Services

Service methods should **not** catch exceptions and continue with default values. Let exceptions propagate to fail the calling workflow:

```python
# ❌ BAD: Catch and continue with empty data
def collect_stats(self, project: str) -> Stats:
    try:
        prs = self.pr_service.get_open_prs(project)
    except Exception as e:
        print(f"Error: {e}")
        prs = []  # ❌ Silent failure - stats will be wrong
    return Stats(prs=prs)

# ✅ GOOD: Let exceptions propagate
def collect_stats(self, project: str) -> Stats:
    prs = self.pr_service.get_open_prs(project)  # ✅ Fails if API errors
    return Stats(prs=prs)
```

If the GitHub API fails, the workflow should fail - not continue with incomplete data.

### Custom Exception Classes

Define specific exceptions to make error handling clear:

```python
# domain/exceptions.py
class ClaudeChainError(Exception):
    """Base exception for ClaudeChain errors."""
    pass

class ConfigurationError(ClaudeChainError):
    """Raised when configuration is invalid or missing."""
    pass

class ProjectNotFoundError(ClaudeChainError):
    """Raised when a project doesn't exist."""
    pass

class TaskNotFoundError(ClaudeChainError):
    """Raised when a task doesn't exist."""
    pass

# Usage
def get_project(name: str) -> Project:
    project = self.repository.find_by_name(name)
    if project is None:
        raise ProjectNotFoundError(f"Project '{name}' not found")
    return project
```

### Fail-Fast Checklist

When writing code that handles potentially missing data:

- [ ] Would an empty result indicate an error or a valid state?
- [ ] Will the caller be able to proceed meaningfully with empty/default data?
- [ ] Could this silent failure cause problems downstream?
- [ ] Is the default value truly sensible, or just convenient?
- [ ] Would I want to know immediately if this value was missing?

**When in doubt, raise an exception.** It's easier to catch and handle an exception than to debug silent failures.

## Principle: Configuration Should Flow Downward

Configuration values should flow explicitly from the entry point (CLI, web handler) down through the layers. Defaults in the middle of the call stack are dangerous because callers may unintentionally rely on them, leading to bugs.

## Anti-Pattern: Default Configuration Deep in Call Stack (❌ Avoid)

```python
# ❌ BAD: Default configuration deep in the call stack
class StatisticsService:
    def __init__(self, repo: str, metadata_service: MetadataService):
        self.repo = repo
        self.metadata_service = metadata_service

    def collect_all_statistics(
        self,
        config_path: Optional[str] = None,
        days_back: int = 30,  # ❌ Default here
        label: str = DEFAULT_PR_LABEL,  # ❌ Default here
        base_branch: str = "main"  # ❌ Default here
    ):
        # Uses defaults if caller doesn't specify
        ...
```

**Problems:**
- **Silent failures**: Caller forgets to pass `base_branch`, gets unexpected "main" instead of their intended branch
- **Configuration drift**: Different call sites may assume different defaults
- **Hard to trace**: Where did this value come from? Explicit parameter or default?
- **Testing issues**: Tests pass with defaults, production fails with actual values
- **Unclear intent**: Did caller want "main" or did they forget to specify?

## Recommended Pattern: Explicit Configuration Flow (✅ Use This)

```python
# ✅ GOOD: Configuration flows from constructor or is required per-call
class StatisticsService:
    def __init__(
        self,
        repo: str,
        metadata_service: MetadataService,
        project_repository: ProjectRepository,
        base_branch: str  # ✅ Required in constructor - set once for service lifetime
    ):
        """All configuration required at construction"""
        self.repo = repo
        self.metadata_service = metadata_service
        self.project_repository = project_repository
        self.base_branch = base_branch  # ✅ Stored, used by all methods

    def collect_all_statistics(
        self,
        config_path: Optional[str],  # ✅ Truly optional - None has meaning
        days_back: int,  # ✅ Required - caller must decide
        label: str  # ✅ Required - caller must decide
    ) -> StatisticsReport:
        """No defaults - uses instance variables or requires parameters"""
        # ✅ Uses instance variable set in constructor
        base_branch = self.base_branch

        # ✅ All parameters were required, no ambiguity
        return self._collect_statistics(config_path, days_back, label, base_branch)
```

**Configuration flows from the top:**
```python
# In __main__.py - Entry point sets defaults ONCE
def main():
    args = parse_args()

    # ✅ Defaults only at entry point, explicit about source
    repo = args.repo or os.environ.get("GITHUB_REPOSITORY", "")
    base_branch = args.base_branch or os.environ.get("BASE_BRANCH", "main")
    days_back = args.days_back or int(os.environ.get("STATS_DAYS_BACK", "30"))
    label = args.label or os.environ.get("PR_LABEL", "claudechain")

    # ✅ Pass everything explicitly down the stack
    return cmd_statistics(
        gh=gh,
        repo=repo,
        base_branch=base_branch,
        days_back=days_back,
        label=label
    )

# In commands/statistics.py - Receives explicit values
def cmd_statistics(
    gh: GitHubActionsHelper,
    repo: str,
    base_branch: str,  # ✅ Required - no default
    days_back: int,  # ✅ Required - no default
    label: str  # ✅ Required - no default
) -> int:
    """All configuration passed explicitly"""
    # ✅ Create service with configuration
    service = StatisticsService(repo, metadata_service, project_repo, base_branch)

    # ✅ Call with explicit parameters
    report = service.collect_all_statistics(
        config_path=None,
        days_back=days_back,
        label=label
    )
```

### Benefits of Explicit Configuration Flow

✅ **Single source of truth**: Defaults defined once at entry point

✅ **Traceable**: Easy to see where values come from (CLI arg, env var, or hardcoded default)

✅ **Intentional**: Every caller must consciously provide or accept values

✅ **Fail fast**: Missing configuration caught at entry point, not deep in call stack

✅ **Testable**: Tests must explicitly specify values, catching assumptions

✅ **No surprises**: What caller passes is what service uses - no hidden defaults

### When Defaults Are Acceptable

Use defaults only in these specific cases:

1. **Entry point only**: CLI argument parsers, main() functions
   ```python
   # ✅ OK: Default at entry point
   parser.add_argument("--days-back", type=int, default=30)
   ```

2. **True behavioral flags**: Optional features that are genuinely off/on
   ```python
   # ✅ OK: Feature flag with clear default
   def format_output(data: dict, include_debug: bool = False):
       # Debug output is optional, False is the natural default
   ```

3. **Backward compatibility**: When extending existing APIs
   ```python
   # ✅ OK: New parameter defaults to old behavior
   def legacy_api(required: str, new_feature: bool = False):
       # False preserves existing behavior
   ```

**Avoid defaults for:**
- Configuration values (branch names, timeouts, limits)
- Business logic parameters (user IDs, project names, dates)
- Any value where the caller should make a conscious choice

### Configuration Checklist

When adding a new parameter, ask:

- [ ] Should this be in the constructor (applies to all operations)?
- [ ] Should this be a method parameter (varies per call)?
- [ ] Does this need a default, or should it be required?
- [ ] If it has a default, is it truly optional or just convenient?
- [ ] Will callers understand what happens if they omit this?
- [ ] Can this lead to silent failures if the wrong default is used?

**Default to making parameters required** - only add defaults with clear justification.

## Principle: Services Should Not Read Environment Variables

Service classes and their methods should **never** read environment variables directly using `os.environ.get()`. All environment variable access should happen at the **entry point layer** (CLI commands, web handlers, etc.) and be passed explicitly as constructor or method parameters.

## Anti-Pattern: Services Reading Environment Variables (❌ Avoid)

```python
# ❌ BAD: Service reads environment variables directly
class StatisticsService:
    def __init__(self, repo: str, metadata_service: MetadataService):
        self.repo = repo
        self.metadata_service = metadata_service

    def collect_all_statistics(self, config_path: Optional[str] = None):
        # ❌ Hidden dependency on environment
        base_branch = os.environ.get("BASE_BRANCH", "main")
        label = "claudechain"
        # ... rest of implementation
```

**Problems with this approach:**
- **Hidden dependencies**: Not obvious from the API what environment variables are needed
- **Poor testability**: Tests must mock environment variables
- **Tight coupling**: Service is coupled to the deployment environment
- **Hard to reuse**: Can't easily use the service in different contexts
- **Inconsistent**: Mixes explicit parameters with implicit environment access

## Recommended Pattern: Explicit Parameters (✅ Use This)

```python
# ✅ GOOD: Service receives all configuration explicitly
class StatisticsService:
    def __init__(
        self,
        repo: str,
        metadata_service: MetadataService,
        base_branch: str = "main"
    ):
        """Initialize the statistics service

        Args:
            repo: GitHub repository (owner/name)
            metadata_service: MetadataService instance for accessing metadata
            base_branch: Base branch to fetch specs from (default: "main")
        """
        self.repo = repo
        self.metadata_service = metadata_service
        self.base_branch = base_branch  # ✅ Stored as instance variable

    def collect_all_statistics(
        self,
        config_path: Optional[str] = None,
        label: str = "claudechain"
    ):
        """Collect statistics for all projects

        Args:
            config_path: Optional path to specific config
            label: GitHub label for filtering (default: "claudechain")
        """
        # ✅ Uses instance variables, no environment access
        base_branch = self.base_branch
        # ... rest of implementation
```

**Adapter layer in `__main__.py` handles ALL environment variable reading:**

```python
# In __main__.py - Adapter layer reads environment variables and CLI args
elif args.command == "statistics":
    return cmd_statistics(
        gh=gh,
        repo=args.repo or os.environ.get("GITHUB_REPOSITORY", ""),
        base_branch=args.base_branch or os.environ.get("BASE_BRANCH", "main"),
        config_path=args.config_path or os.environ.get("CONFIG_PATH"),
        days_back=args.days_back or int(os.environ.get("STATS_DAYS_BACK", "30")),
        format_type=args.format or os.environ.get("STATS_FORMAT", "slack"),
        slack_webhook_url=os.environ.get("SLACK_WEBHOOK_URL", "")
    )

# In cli/commands/statistics.py - Command receives explicit parameters
def cmd_statistics(
    gh: GitHubActionsHelper,
    repo: str,
    base_branch: str = "main",
    config_path: Optional[str] = None,
    days_back: int = 30,
    format_type: str = "slack",
    slack_webhook_url: str = ""
) -> int:
    """Orchestrate statistics workflow."""
    # ✅ Passes everything explicitly to service
    metadata_store = GitHubMetadataStore(repo)
    metadata_service = MetadataService(metadata_store)
    statistics_service = StatisticsService(repo, metadata_service, base_branch)

    # ✅ Uses parameters directly - no environment access
    report = statistics_service.collect_all_statistics(
        config_path=config_path if config_path else None
    )
```

### Benefits

✅ **Explicit dependencies**: Constructor signature shows exactly what the service needs

✅ **Easy testing**: Just pass test values - no environment mocking needed
```python
# Testing is straightforward
service = StatisticsService("owner/repo", mock_metadata, base_branch="develop")
```

✅ **Separation of concerns**: CLI handles environment, service handles business logic

✅ **Reusability**: Service works in any context (CLI, web app, scripts, tests)

✅ **Type safety**: IDEs can autocomplete and type-check parameters

✅ **Self-documenting**: API clearly shows what configuration is needed

### When to Use Constructor vs Method Parameters

**Use constructor parameters for:**
- Configuration that applies to all operations (e.g., `repo`, `base_branch`)
- Dependencies/services that won't change (e.g., `metadata_service`)
- Settings that define the service's behavior globally

**Use method parameters for:**
- Operation-specific values that vary per call (e.g., `config_path`, `days_back`)
- Optional filters or constraints (e.g., `label`)
- Values that might differ between invocations

### Exception: Environment Variables in Infrastructure Layer

The **only** layer that should read environment variables is the **infrastructure layer** - specifically for connecting to external systems:

```python
# ✅ OK: Infrastructure layer for GitHub API connections
class GitHubMetadataStore:
    def __init__(self, repo: str, token: Optional[str] = None):
        self.repo = repo
        # ✅ OK here: Infrastructure layer connecting to external system
        self.token = token or os.environ.get("GITHUB_TOKEN")
```

Even here, prefer explicit parameters with environment variables as fallback defaults.

## Design Principles

1. **Explicit over implicit**: Make all dependencies visible in constructor signatures
2. **Required over optional**: Use required parameters unless truly optional
3. **Fail fast over silent failures**: Raise exceptions for abnormal cases
4. **Top-down configuration**: Flow configuration from entry points down
5. **No environment reads in services**: Keep services environment-agnostic

## Related Patterns

- **Constructor Injection**: Standard DI pattern where dependencies are passed via constructor
- **Poor Man's DI**: Manual dependency wiring at composition root (vs DI containers)
- **Dependency Inversion Principle**: Depend on abstractions, not concretions
- **Fail Fast**: Validate early and raise exceptions immediately
- **Configuration as Code**: Explicit configuration objects vs scattered defaults

## Related Skills

- **creating-services**: Apply DI patterns when creating service classes (plugin/skills/creating-services)
- **cli-architecture**: Understand how configuration flows from CLI entry points to services (plugin/skills/cli-architecture)
- **testing-services**: Testing with explicit dependencies makes mocking straightforward (plugin/skills/testing-services)
- **identifying-layer-placement**: Understand which layer should handle environment variables and configuration (plugin/skills/identifying-layer-placement)

## Further Reading

- Clean Architecture by Robert C. Martin (Uncle Bob)
- Dependency Injection Principles, Practices, and Patterns by Steven van Deursen and Mark Seemann
- Python documentation on dataclasses and type hints
- Martin Fowler's article on Dependency Injection
