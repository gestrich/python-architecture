# Python Code Style Guide

## Service Layer Organization

This document describes the organizational principles and patterns used for Python code in the ClaudeChain project, particularly for service classes.

## Method Organization Principles

All service classes follow a consistent organization pattern to improve readability and maintainability:

### 1. Public Before Private

Public methods (part of the API) appear before private/internal methods (prefixed with `_`). This allows developers to understand the public API of each service at a glance without scrolling through implementation details.

### 2. High-Level Before Low-Level

More abstract, higher-level operations come before detailed implementation helpers. This creates a natural reading flow from "what the service does" to "how it does it."

### 3. Logical Grouping

Related methods are grouped together with clear section comments. This helps developers quickly navigate to the functionality they need.

### 4. Standard Order

Methods follow this ordering:
1. Special methods (`__init__`, `__str__`, etc.)
2. Class methods (`@classmethod`)
3. Static methods (`@staticmethod`)
4. Instance methods (public, then private)

## Section Headers

Use clear section comments to separate different types of methods:

```python
class MyService:
    def __init__(self):
        """Constructor always comes first."""
        pass

    # Public API methods
    def high_level_operation(self):
        """Main public methods in order of abstraction level."""
        pass

    def mid_level_operation(self):
        """Supporting public methods."""
        pass

    # Static utility methods
    @staticmethod
    def utility_function():
        """Static utilities at the end."""
        pass

    # Private helper methods
    def _internal_helper(self):
        """Private implementation details last."""
        pass
```

For services with many methods, use more descriptive section headers with separators:

```python
class ComplexService:
    def __init__(self):
        pass

    # ============================================================
    # Core CRUD Operations
    # ============================================================

    def create_resource(self):
        pass

    def get_resource(self):
        pass

    # ============================================================
    # Query Operations
    # ============================================================

    def find_resources(self):
        pass

    # ============================================================
    # Utility Operations
    # ============================================================

    @staticmethod
    def parse_identifier(text: str):
        pass
```

## Dependency Injection and Optional Parameters

### Principle: Avoid Optional Dependencies with Default Factories

Optional parameters with default factory patterns (e.g., `param: Optional[Type] = None` with `self.param = param or DefaultType()`) are a code smell that hides dependencies and makes code harder to reason about.

### Anti-Pattern (❌ Avoid)

```python
# BAD: Optional dependency with default factory
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

### Recommended Pattern (✅ Use This)

```python
# GOOD: Required dependency - explicit and clear
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

## Fail Fast: Avoid Silent Failures

### Principle: Treat Abnormal Cases as Errors, Not Empty Data

When a value is missing, invalid, or unexpected, **raise an exception** rather than silently returning empty data structures or default values. Missing values are often errors that should be surfaced, not hidden.

### Anti-Pattern (❌ Avoid)

```python
# BAD: Silently returning empty data when something is wrong
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

# BAD: Using .get() with defaults to mask missing required fields
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

### Recommended Pattern (✅ Use This)

```python
# GOOD: Raise exceptions for abnormal cases
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

# GOOD: Validate required fields explicitly
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

### Checklist

When writing code that handles potentially missing data:

- [ ] Would an empty result indicate an error or a valid state?
- [ ] Will the caller be able to proceed meaningfully with empty/default data?
- [ ] Could this silent failure cause problems downstream?
- [ ] Is the default value truly sensible, or just convenient?
- [ ] Would I want to know immediately if this value was missing?

**When in doubt, raise an exception.** It's easier to catch and handle an exception than to debug silent failures.

## Configuration Defaults and Flow

### Principle: Configuration Should Flow Downward, Avoid Defaults

Configuration values should flow explicitly from the entry point (CLI, web handler) down through the layers. Defaults in the middle of the call stack are dangerous because callers may unintentionally rely on them, leading to bugs.

### Anti-Pattern (❌ Avoid)

```python
# BAD: Default configuration deep in the call stack
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

### Recommended Pattern (✅ Use This)

```python
# GOOD: Configuration flows from constructor (set once) or is required per-call

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

## Environment Variables and Configuration

### Principle: Services Should Not Read Environment Variables

Service classes and their methods should **never** read environment variables directly using `os.environ.get()`. All environment variable access should happen at the **entry point layer** (CLI commands, web handlers, etc.) and be passed explicitly as constructor or method parameters.

### Anti-Pattern (❌ Avoid)

```python
# BAD: Service reads environment variables directly
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

### Recommended Pattern (✅ Use This)

```python
# GOOD: Service receives all configuration explicitly
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
# OK: Infrastructure layer for GitHub API connections
class GitHubMetadataStore:
    def __init__(self, repo: str, token: Optional[str] = None):
        self.repo = repo
        # ✅ OK here: Infrastructure layer connecting to external system
        self.token = token or os.environ.get("GITHUB_TOKEN")
```

Even here, prefer explicit parameters with environment variables as fallback defaults.

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

## Module-Level Code

For modules with functions rather than classes (like `artifact_operations_service.py`), use this order:

1. **Dataclasses and models** (public before private)
2. **Public API functions** (high-level to low-level)
3. **Module utilities** (helper functions used by the public API)
4. **Private helper functions** (prefixed with `_`)

Example:

```python
# Public models
@dataclass
class ProjectArtifact:
    """Public dataclass."""
    pass

# Public API functions
def find_artifacts():
    """Highest-level public function."""
    pass

def get_artifact_details():
    """Mid-level public function."""
    pass

# Module utilities
def parse_artifact_name(name: str):
    """Utility function."""
    pass

# Private helper functions
def _fetch_from_api():
    """Private implementation detail."""
    pass
```

## Benefits

This organizational approach provides:

- **Easier onboarding**: New developers can quickly understand what a service does by reading public methods first
- **Better maintainability**: Clear separation between public contracts and implementation details
- **Intuitive navigation**: Developers can find methods more quickly with consistent structure
- **Clearer API boundaries**: Public vs. private methods are visually distinct

## Examples

See the following services for reference implementations:

- [task_management_service.py](../../src/claudechain/application/services/task_management_service.py) - Simple service with public API and static utilities
- [metadata_service.py](../../src/claudechain/application/services/metadata_service.py) - Complex service with multiple logical sections
- [artifact_operations_service.py](../../src/claudechain/application/services/artifact_operations_service.py) - Module-level functions and dataclasses

## Domain Models and Data Parsing

### Principle: Parse Once Into Well-Formed Models

When working with structured data (YAML files, JSON responses, markdown files, etc.), **parse the data once into a well-formed domain model** rather than passing raw strings or dictionaries around and parsing them repeatedly.

### Anti-Pattern (❌ Avoid)

```python
# BAD: Services parse strings directly
class StatisticsService:
    def collect_project_stats(self, project_name: str):
        # String-based path construction
        config_path = f"claude-chain/{project_name}/configuration.yml"
        spec_path = f"claude-chain/{project_name}/spec.md"

        # Fetch raw strings from API
        config_content = get_file_from_branch(repo, branch, config_path)
        spec_content = get_file_from_branch(repo, branch, spec_path)

        # Parse YAML into raw dictionary
        config = yaml.safe_load(config_content)

        # String-based dictionary access (no type safety)
        reviewers_config = config.get("reviewers", [])
        reviewers = [r.get("username") for r in reviewers_config if "username" in r]

        # Regex parsing in service layer
        total = len(re.findall(r"^\s*- \[[xX \]]", spec_content, re.MULTILINE))
        completed = len(re.findall(r"^\s*- \[[xX]\]", spec_content, re.MULTILINE))

        # Return primitive types
        return (total, completed, reviewers)
```

**Problems with this approach:**
- **Parsing logic scattered**: Different services duplicate regex patterns and YAML parsing
- **No type safety**: Dictionary access with string keys can fail silently
- **Brittle**: Changes to file format require updates in multiple places
- **Wrong layer**: Business logic (parsing) mixed with orchestration (service layer)
- **Hard to test**: Must test parsing logic alongside business logic
- **Poor reusability**: Can't reuse parsing logic across services

### Recommended Pattern (✅ Use This)

```python
# GOOD: Domain models encapsulate parsing and provide type-safe APIs

# 1. Domain Model (domain/project.py)
@dataclass
class Project:
    """Domain model representing a ClaudeChain project"""
    name: str

    @property
    def config_path(self) -> str:
        """Centralized path construction"""
        return f"claude-chain/{self.name}/configuration.yml"

    @property
    def spec_path(self) -> str:
        return f"claude-chain/{self.name}/spec.md"

# 2. Configuration Model (domain/project_configuration.py)
@dataclass
class Reviewer:
    """Type-safe reviewer model"""
    username: str
    max_open_prs: int = 2

@dataclass
class ProjectConfiguration:
    """Parsed configuration with validation"""
    project: Project
    reviewers: List[Reviewer]

    @classmethod
    def from_yaml_string(cls, project: Project, content: str) -> 'ProjectConfiguration':
        """Parse once, validate, return type-safe model"""
        config = yaml.safe_load(content)

        # Validate structure
        if "reviewers" not in config:
            raise ConfigurationError("Missing reviewers field")

        # Parse into typed objects
        reviewers = [
            Reviewer(
                username=r["username"],
                max_open_prs=r.get("maxOpenPRs", 2)
            )
            for r in config["reviewers"]
            if "username" in r
        ]

        return cls(project=project, reviewers=reviewers)

    def get_reviewer_usernames(self) -> List[str]:
        """Type-safe API for common operations"""
        return [r.username for r in self.reviewers]

# 3. Spec Model (domain/spec_content.py)
@dataclass
class SpecTask:
    """Parsed task from spec.md"""
    index: int
    description: str
    is_completed: bool

class SpecContent:
    """Parsed spec.md with task extraction"""

    def __init__(self, project: Project, content: str):
        self.project = project
        self.content = content
        self._tasks: Optional[List[SpecTask]] = None

    @property
    def tasks(self) -> List[SpecTask]:
        """Parse once, cache results"""
        if self._tasks is None:
            self._tasks = self._parse_tasks()
        return self._tasks

    def _parse_tasks(self) -> List[SpecTask]:
        """Centralized regex parsing logic"""
        tasks = []
        task_index = 1

        for line in self.content.split('\n'):
            match = re.match(r'^\s*- \[([xX ])\]\s*(.+)$', line)
            if match:
                tasks.append(SpecTask(
                    index=task_index,
                    description=match.group(2).strip(),
                    is_completed=match.group(1).lower() == 'x'
                ))
                task_index += 1

        return tasks

    @property
    def total_tasks(self) -> int:
        """Clean API for statistics"""
        return len(self.tasks)

    @property
    def completed_tasks(self) -> int:
        return sum(1 for task in self.tasks if task.is_completed)

# 4. Repository Pattern (infrastructure/repositories/project_repository.py)
class ProjectRepository:
    """Infrastructure layer: Fetch and parse into domain models"""

    def __init__(self, repo: str):
        self.repo = repo

    def load_configuration(
        self, project: Project, branch: str = "main"
    ) -> Optional[ProjectConfiguration]:
        """Fetch from API, parse into domain model"""
        content = get_file_from_branch(self.repo, branch, project.config_path)
        if not content:
            return None

        return ProjectConfiguration.from_yaml_string(project, content)

    def load_spec(
        self, project: Project, branch: str = "main"
    ) -> Optional[SpecContent]:
        """Fetch from API, parse into domain model"""
        content = get_file_from_branch(self.repo, branch, project.spec_path)
        if not content:
            return None

        return SpecContent(project, content)

# 5. Service Layer (services/statistics_service.py)
class StatisticsService:
    """Service uses domain models - no parsing logic"""

    def __init__(
        self,
        repo: str,
        metadata_service: MetadataService,
        project_repository: ProjectRepository
    ):
        self.repo = repo
        self.metadata_service = metadata_service
        self.project_repository = project_repository

    def collect_project_stats(self, project_name: str) -> ProjectStats:
        """Clean, type-safe service logic"""
        project = Project(project_name)

        # Load parsed domain models
        config = self.project_repository.load_configuration(project)
        spec = self.project_repository.load_spec(project)

        if not config or not spec:
            return None

        # Use type-safe APIs (no parsing, no string keys)
        stats = ProjectStats(project_name)
        stats.total_tasks = spec.total_tasks
        stats.completed_tasks = spec.completed_tasks
        stats.reviewers = config.get_reviewer_usernames()

        return stats
```

### Benefits

✅ **Parse once**: Data is parsed into models at the boundary (repository/infrastructure layer)

✅ **Type safety**: Domain models provide strongly-typed properties and methods
```python
# Autocomplete works, typos caught at runtime
reviewers = config.reviewers  # List[Reviewer]
username = reviewers[0].username  # str (typed!)
```

✅ **Centralized parsing**: Regex patterns and parsing logic in one place
```python
# All spec parsing happens in SpecContent
# Services just use the clean API
total = spec.total_tasks
pending = spec.pending_tasks
```

✅ **Clear layering**:
- **Infrastructure**: Fetches raw data from external systems
- **Domain**: Parses into validated, type-safe models
- **Service**: Orchestrates domain models (no parsing!)

✅ **Testability**: Test parsing separately from business logic
```python
# Test parsing independently
def test_spec_content_parsing():
    spec = SpecContent(project, "- [ ] Task 1\n- [x] Task 2")
    assert spec.total_tasks == 2
    assert spec.completed_tasks == 1

# Test service with mocked domain models
def test_statistics_service():
    mock_spec = Mock(total_tasks=10, completed_tasks=5)
    mock_repo.load_spec.return_value = mock_spec
    # Test service logic without parsing concerns
```

✅ **Validation at boundaries**: Models validate on construction
```python
@classmethod
def from_yaml_string(cls, project: Project, content: str):
    config = yaml.safe_load(content)

    # Validate early, fail fast
    if "reviewers" not in config:
        raise ConfigurationError("Missing reviewers")

    return cls(...)
```

✅ **Reusability**: Models can be used across different services
```python
# Multiple services can use the same parsed models
class StatisticsService:
    def collect_stats(self, project: Project):
        config = self.repo.load_configuration(project)

class ReviewerService:
    def assign_reviewer(self, project: Project):
        config = self.repo.load_configuration(project)  # Same model!
```

### Architecture Pattern

```
┌─────────────────────────────────────────────────────────────┐
│ External System (GitHub API, File System)                   │
│ - Returns: Raw strings (YAML, JSON, Markdown)              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Infrastructure Layer (Repository Pattern)                   │
│ - Fetches raw data from external systems                   │
│ - Delegates parsing to domain model factories              │
│ - Returns: Fully-parsed domain models                      │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Domain Layer (Models with Factory Methods)                 │
│ - Project, ProjectConfiguration, SpecContent               │
│ - Encapsulates parsing logic (regex, YAML, validation)    │
│ - Provides type-safe APIs                                  │
│ - Single source of truth for data structure                │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────────┐
│ Service Layer (Business Logic)                             │
│ - Receives parsed domain models                            │
│ - Uses type-safe model APIs                                │
│ - No string parsing or dictionary access                   │
│ - Focuses on orchestration and business rules              │
└─────────────────────────────────────────────────────────────┘
```

### Key Principles

1. **Parse at boundaries**: Convert raw data to domain models as soon as it enters your system
2. **Parse once**: Never re-parse the same data in multiple places
3. **Domain owns parsing**: Parsing logic belongs in domain models, not services
4. **Type-safe APIs**: Models expose strongly-typed properties and methods
5. **Validate early**: Domain model constructors/factories validate structure
6. **No leaky abstractions**: Services don't know about YAML, regex, or file formats

### Related Patterns

This approach aligns with:
- **Repository Pattern**: Infrastructure fetches, domain parses
- **Factory Pattern**: `from_yaml_string()`, `from_branch_name()` constructors
- **Domain-Driven Design**: Rich domain models with behavior
- **Dependency Inversion**: Services depend on domain abstractions, not infrastructure

## Datetime and Timezone Handling

### Principle: Always Use Timezone-Aware Datetimes

**All datetime objects in ClaudeChain must be timezone-aware.** Naive datetimes (without timezone information) are not allowed and will raise validation errors.

### Anti-Pattern (❌ Avoid)

```python
# BAD: Naive datetime - no timezone information
from datetime import datetime

now = datetime.now()  # ❌ Naive datetime
timestamp = datetime(2025, 1, 15, 10, 30, 0)  # ❌ Naive datetime
utc_now = datetime.utcnow()  # ❌ Deprecated and naive
```

**Problems with naive datetimes:**
- **Ambiguous**: Which timezone does this represent?
- **Comparison errors**: Cannot compare naive and timezone-aware datetimes
- **Data bugs**: Serialized without timezone, leading to misinterpretation
- **Not interoperable**: Different systems may interpret differently

### Recommended Pattern (✅ Use This)

```python
# GOOD: Always use timezone-aware datetimes
from datetime import datetime, timezone

# Creating new timestamps - always use UTC
now = datetime.now(timezone.utc)  # ✅ Timezone-aware
timestamp = datetime(2025, 1, 15, 10, 30, 0, tzinfo=timezone.utc)  # ✅ Timezone-aware

# Parsing ISO 8601 timestamps with timezone
from claudechain.domain.models import parse_iso_timestamp

# Parse with helper function (handles both "Z" and "+00:00" formats)
parsed_dt = parse_iso_timestamp("2025-01-15T10:30:00Z")  # ✅ Returns timezone-aware
parsed_dt2 = parse_iso_timestamp("2025-01-15T10:30:00+00:00")  # ✅ Also works

# Serializing to ISO 8601 with timezone
timestamp_str = now.isoformat()  # ✅ Produces "2025-01-15T10:30:00+00:00"
```

### Timezone Convention

**Always use UTC** for internal operations and storage:

```python
# ✅ GOOD: Use UTC for all internal timestamps
created_at = datetime.now(timezone.utc)
last_updated = datetime.now(timezone.utc)

# Store in metadata as ISO 8601 with timezone
metadata = {
    "created_at": created_at.isoformat(),  # "2025-01-15T10:30:00+00:00"
    "last_updated": last_updated.isoformat()
}
```

**Why UTC:**
- **Unambiguous**: UTC has no daylight saving time
- **Standard**: Industry best practice for system timestamps
- **Interoperable**: Works across all timezones
- **Comparable**: All timestamps in same timezone can be directly compared

### Domain Model Validation

All domain models with datetime fields include automatic validation:

```python
from dataclasses import dataclass
from datetime import datetime, timezone

@dataclass
class MyModel:
    created_at: datetime

    def __post_init__(self):
        """Validate that datetime fields are timezone-aware."""
        if self.created_at.tzinfo is None:
            raise ValueError(
                f"created_at must be timezone-aware. "
                f"Use datetime.now(timezone.utc) or parse_iso_timestamp()"
            )
```

This validation prevents naive datetimes from being created, catching timezone bugs immediately.

### Common Pitfalls to Avoid

❌ **Don't use `datetime.now()` without timezone:**
```python
# BAD
timestamp = datetime.now()  # ❌ Naive datetime
```

✅ **Use `datetime.now(timezone.utc)` instead:**
```python
# GOOD
timestamp = datetime.now(timezone.utc)  # ✅ Timezone-aware UTC
```

❌ **Don't use deprecated `datetime.utcnow()`:**
```python
# BAD
timestamp = datetime.utcnow()  # ❌ Deprecated and naive
```

✅ **Use `datetime.now(timezone.utc)` instead:**
```python
# GOOD
timestamp = datetime.now(timezone.utc)  # ✅ Modern and timezone-aware
```

❌ **Don't parse ISO 8601 strings directly with `fromisoformat()`:**
```python
# BAD - fromisoformat() doesn't handle "Z" suffix
timestamp = datetime.fromisoformat("2025-01-15T10:30:00Z")  # ❌ Raises ValueError
```

✅ **Use the `parse_iso_timestamp()` helper:**
```python
# GOOD
from claudechain.domain.models import parse_iso_timestamp
timestamp = parse_iso_timestamp("2025-01-15T10:30:00Z")  # ✅ Handles "Z" and "+00:00"
```

### ISO 8601 Serialization Format

When serializing datetimes to JSON or strings, always include timezone information:

```python
# ✅ GOOD: ISO 8601 with timezone
timestamp = datetime.now(timezone.utc)
json_value = timestamp.isoformat()  # "2025-01-15T10:30:00+00:00"

# The isoformat() method automatically includes the timezone suffix
# - UTC timezone: "+00:00" suffix
# - "Z" suffix is equivalent to "+00:00" (both represent UTC)
```

**Acceptable formats:**
- `2025-01-15T10:30:00+00:00` (preferred - Python's default)
- `2025-01-15T10:30:00Z` (also valid - common in APIs)

**Invalid formats:**
- `2025-01-15T10:30:00` ❌ (no timezone - ambiguous)

### Helper Function: parse_iso_timestamp()

Use the centralized helper function for parsing ISO 8601 timestamps:

```python
from claudechain.domain.models import parse_iso_timestamp
from datetime import datetime, timezone

# Handles both "Z" and "+00:00" timezone formats
dt1 = parse_iso_timestamp("2025-01-15T10:30:00Z")
dt2 = parse_iso_timestamp("2025-01-15T10:30:00+00:00")

# Both return timezone-aware datetime objects
assert dt1.tzinfo is not None  # ✅ Always timezone-aware
assert dt2.tzinfo is not None  # ✅ Always timezone-aware

# For backward compatibility, also handles naive timestamps (adds UTC)
# But all new code should only produce timezone-aware timestamps
dt3 = parse_iso_timestamp("2025-01-15T10:30:00")  # Adds timezone.utc
```

### Testing with Datetimes

In tests, always create timezone-aware datetimes:

```python
# ✅ GOOD: Timezone-aware test fixtures
from datetime import datetime, timezone

def test_my_feature():
    # Create test timestamp with timezone
    test_time = datetime(2025, 1, 15, 10, 30, 0, tzinfo=timezone.utc)

    # Use in assertions
    result = my_function(test_time)
    assert result.created_at == test_time
```

### Benefits

✅ **Prevents comparison errors**: No more "can't compare offset-naive and offset-aware datetimes"

✅ **Unambiguous data**: Timestamps clearly indicate their timezone

✅ **ISO 8601 compliant**: Standard format works everywhere

✅ **Interoperable**: Data works across systems and timezones

✅ **Self-describing**: Data itself indicates timezone, not relying on implicit conventions

✅ **Validated**: Domain models prevent naive datetimes at construction time

### Checklist

When working with datetimes:

- [ ] Use `datetime.now(timezone.utc)` for new timestamps (never `datetime.now()`)
- [ ] Use `parse_iso_timestamp()` when parsing ISO 8601 strings
- [ ] Always create test datetimes with `tzinfo=timezone.utc`
- [ ] Verify `.isoformat()` includes timezone suffix in output
- [ ] Never use deprecated `datetime.utcnow()`
- [ ] Check that domain models validate timezone-aware datetimes in `__post_init__`

## Avoiding Circular Imports

### Principle: Design One-Way Dependency Graphs

Circular imports indicate architectural problems. Fix the dependency structure rather than working around it.

### Anti-Patterns (❌ Avoid)

```python
# BAD: TYPE_CHECKING guard
if TYPE_CHECKING:
    from module_b import SomeClass  # Only imported during type checking
open_items: List["SomeClass"]  # String annotation workaround

# BAD: Using Any
open_items: List[Any] = []  # "Any to avoid circular import" ❌
```

Both approaches hide the architectural issue rather than solving it. `TYPE_CHECKING` means your code works at runtime only because the import is skipped. `Any` throws away type safety entirely.

### Fix: Establish One-Way Dependencies

1. **Identify which module is more foundational** (lower-level)
2. **Ensure the lower-level module never imports from higher-level ones**
3. **Verify with grep**: `grep "from higher_module" lower_module.py` should return nothing

```python
# lower_level.py - No imports from higher_level.py
class GitHubPullRequest: ...

# higher_level.py - Can import from lower_level.py
from lower_level import GitHubPullRequest  # ✅ One-way dependency
class ProjectStats:
    open_prs: List[GitHubPullRequest]  # ✅ Proper typing
```

If you have a genuine cycle, either move shared code to a third module or refactor so the lower-level module doesn't need higher-level types.

## Type Annotations

### Use `Self` for Factory Methods

When a classmethod or factory returns an instance of its own class, use `Self` from `typing` instead of quoted string annotations:

```python
# ❌ Avoid: Quoted string annotation
@classmethod
def from_string(cls, value: str) -> "MyClass":
    ...

# ✅ Use: Self type (Python 3.11+)
from typing import Self

@classmethod
def from_string(cls, value: str) -> Self:
    ...
```

`Self` is cleaner, works correctly with subclasses, and avoids forward reference strings.

### Use `from __future__ import annotations` for Self-Referential Types

For classes that reference themselves in type hints (e.g., tree structures, containers with nested elements), use `from __future__ import annotations` to enable deferred evaluation:

```python
# ✅ Use: Deferred annotations for self-referential types
from __future__ import annotations

from dataclasses import dataclass
from typing import List, Optional

@dataclass
class Section:
    elements: List[Section]  # ✅ No quotes needed - deferred evaluation
    parent: Optional[Section] = None

    def add(self, element: Section) -> Section:  # ✅ Clean type hints
        self.elements.append(element)
        return self
```

Without `from __future__ import annotations`, you'd need quoted strings (`"Section"`) for forward references, which is less readable.

## Related Documentation

- See [docs/completed/2025-12-30-reorganize-service-methods.md](../completed/2025-12-30-reorganize-service-methods.md) for the history of applying these principles to the codebase
- See [docs/proposed/2025-12-27-projectrefactor.md](../proposed/2025-12-27-projectrefactor.md) for the plan to introduce Project, ProjectConfiguration, and SpecContent domain models
- See [docs/proposed/2025-12-27-refactor-statistics-service-architecture.md](../proposed/2025-12-27-refactor-statistics-service-architecture.md) for the complete StatisticsService refactoring process
