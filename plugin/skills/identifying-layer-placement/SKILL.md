---
name: identifying-layer-placement
description: Determines the correct architectural layer (Entry Point/Service/Domain/Infrastructure) for code placement in Python applications using Service Layer pattern. Prevents layer violations and maintains separation of concerns. Use when adding new functionality, refactoring code, or unclear where code should live.
user-invocable: true
argument-hint: "[code-description]"
---

# Identifying Layer Placement

## Quick Decision Tree

```
Where should this code go?
    │
    ▼
Does it interact with users/callers?
(Parse args, format output, routes)
    │
    ├─ YES → ENTRY POINT (CLI, API routes, scripts)
    │
    └─ NO → Does it call external systems? (API, DB, FS)
            │
            ├─ YES → INFRASTRUCTURE (API clients, DB, file I/O, git)
            │
            └─ NO → Is it pure data or business rules?
                    │
                    ├─ YES → DOMAIN (models, dataclasses, parsing, validation)
                    │
                    └─ NO → SERVICE (business logic, coordination)
```

## Layer Architecture

```
┌─────────────────────────────────────────┐
│          Entry Point Layer              │  ← User/caller interface
│  (CLI, API routes, scripts, handlers)   │    (orchestration only)
├─────────────────────────────────────────┤
│         Service Layer                   │  ← Business logic
│  Core: Single-responsibility services   │    (coordinates operations)
│  Composite: Multi-service coordination  │
├─────────────────────────────────────────┤
│       Infrastructure Layer              │  ← External integrations
│  (API clients, DB, filesystem, git)     │    (I/O boundaries)
├─────────────────────────────────────────┤
│         Domain Layer                    │  ← Pure business models
│  (dataclasses, models, parsing, rules)  │    (no I/O, no dependencies)
└─────────────────────────────────────────┘

Dependency flow:
Entry Point → Service → Infrastructure
Entry Point → Service → Domain
Domain: No dependencies (pure)
```

## Layer Responsibilities

### Entry Point Layer
**Purpose**: Interface between users/callers and the application

**Belongs here**:
- Argument parsing (CLI flags, query params)
- Response formatting (JSON, text, exit codes)
- HTTP route definitions
- Error handling for user-facing messages
- Orchestrating service calls (no business logic)

**File locations**: `cli/`, `api/`, `scripts/`, `main.py`, `__main__.py`

**Note**: For detailed command dispatcher patterns and CLI architecture, see the **cli-architecture** skill.

**Example**:
```python
def cmd_prepare(args, gh):
    """CLI command orchestrates services."""
    # 1. Get configuration
    repo = os.environ.get("GITHUB_REPOSITORY")

    # 2. Initialize infrastructure
    metadata_store = GitHubMetadataStore(repo)

    # 3. Initialize services
    metadata_service = MetadataService(metadata_store)
    task_service = TaskService(repo, metadata_service)

    # 4. Use services
    task = task_service.find_next_available_task(spec_content)

    # 5. Format output for user
    print(f"Next task: {task.title}")
```

### Service Layer
**Purpose**: Encapsulate business logic and coordinate operations

**Belongs here**:
- Business operations (create, update, process, validate)
- Workflow coordination across multiple components
- Transaction boundaries and business rule enforcement

**File locations**: `services/`, `business/`

**Note**: For service constructor patterns and dependency management, see the **dependency-injection** skill.

**Example - Core service**:
```python
class TaskService:
    def __init__(self, repo: str, metadata_service: MetadataService):
        self.repo = repo
        self.metadata_service = metadata_service

    def find_next_available_task(self, spec: SpecFile) -> Task:
        """Business logic for finding next task."""
        completed = self.metadata_service.get_completed_tasks()
        return spec.find_next_pending(completed)
```

**Example - Composite service**:
```python
class StatisticsService:
    def __init__(self, task_service: TaskService, pr_service: PRService):
        self.task_service = task_service
        self.pr_service = pr_service

    def generate_project_stats(self) -> ProjectStats:
        """Coordinate across services."""
        tasks = self.task_service.get_all_tasks()
        prs = self.pr_service.get_all_prs()
        return ProjectStats(tasks, prs)
```

### Domain Layer
**Purpose**: Pure business models and rules with no external dependencies

**Belongs here**:
- Dataclasses and models
- Data parsing from raw formats (YAML, JSON, Markdown)
- Business rule validation (pure functions)
- Value objects, enums, domain exceptions

**File locations**: `models/`, `domain/`, `entities/`, `config/`

**Note**: For comprehensive domain modeling guidance and the parse-once principle, see the **domain-modeling** skill.

**Example**:
```python
@dataclass
class Task:
    """Pure domain model."""
    title: str
    status: TaskStatus
    assignee: Optional[str] = None

    @classmethod
    def from_markdown(cls, content: str) -> 'Task':
        """Parse from string (domain parsing)."""
        # ... parsing logic ...
        return cls(title=title, status=status)

    def is_available(self) -> bool:
        """Pure business rule."""
        return self.status == TaskStatus.PENDING and self.assignee is None

    def validate(self) -> None:
        """Domain validation."""
        if not self.title:
            raise DomainValidationError("Task must have a title")
```

**Key principle**: Domain models parse data once and provide type-safe APIs. No I/O.

### Infrastructure Layer
**Purpose**: Wrap external systems and I/O operations

**Belongs here**:
- API client wrappers
- Database connections and queries
- File system operations
- Git command execution
- HTTP requests, subprocess calls

**File locations**: `infrastructure/`, `adapters/`, `clients/`

**Example**:
```python
class GitHubMetadataStore:
    """Infrastructure for GitHub API operations."""

    def __init__(self, repo: str):
        self.repo = repo

    def get_completed_tasks(self) -> List[str]:
        """External I/O: Read from GitHub API."""
        result = subprocess.run(
            ["gh", "api", f"/repos/{self.repo}/issues"],
            capture_output=True, text=True
        )
        data = json.loads(result.stdout)
        return [issue['title'] for issue in data if issue['state'] == 'closed']
```

## Anti-Patterns

### Entry Point Anti-Patterns
❌ **DO NOT** put business logic in entry points
```python
# BAD: Business logic in CLI
def cmd_prepare(args):
    for task in spec.tasks:  # Business logic here
        if task.status == "pending":
            return task
```

✅ **DO** delegate to services
```python
# GOOD: Orchestrate only
def cmd_prepare(args):
    task = task_service.find_next_available_task(spec_content)
    print(f"Next task: {task.title}")
```

❌ **DO NOT** directly call infrastructure (e.g., `subprocess.run(["git", "status"])`)

✅ **DO** use services (`git_service.get_status()`)

### Service Anti-Patterns
❌ **DO NOT** parse arguments or access environment
```python
# BAD: Service reads environment
class TaskService:
    def __init__(self):
        self.repo = os.environ.get("GITHUB_REPOSITORY")
```

✅ **DO** receive configuration via constructor
```python
# GOOD: Injected configuration
class TaskService:
    def __init__(self, repo: str, metadata_service: MetadataService):
        self.repo = repo
        self.metadata_service = metadata_service
```

❌ **DO NOT** make direct subprocess/API calls (use injected infrastructure instead)

### Domain Anti-Patterns
❌ **DO NOT** perform I/O operations
```python
# BAD: Domain reads files
@dataclass
class Task:
    @classmethod
    def from_file(cls, path: str):
        with open(path) as f:  # I/O belongs in infrastructure
            return cls.from_yaml(f.read())
```

✅ **DO** parse from strings
```python
# GOOD: Parse string, not file
@dataclass
class Task:
    @classmethod
    def from_yaml(cls, content: str):  # String input
        data = yaml.safe_load(content)
        return cls(**data)
```

❌ **DO NOT** depend on services or infrastructure (keep domain pure with no dependencies)

### Infrastructure Anti-Patterns
❌ **DO NOT** contain business logic
```python
# BAD: Infrastructure makes business decisions
class GitHubMetadataStore:
    def get_next_task(self):
        tasks = self._fetch_all_tasks()
        for task in tasks:  # Business logic
            if task['status'] == 'pending':
                return task
```

✅ **DO** provide simple CRUD operations
```python
# GOOD: Just fetch data
class GitHubMetadataStore:
    def get_all_tasks(self) -> List[dict]:
        result = subprocess.run(["gh", "api", "..."])
        return json.loads(result.stdout)
```

## Common Scenarios

### Scenario 1: Report Generation
**Q**: "Generate a summary report of tasks and PRs. Where does this go?"

**A**: Service Layer - Coordinates multiple data sources with business logic

```python
# Service layer
class ReportService:
    def __init__(self, task_service: TaskService, pr_service: PRService):
        self.task_service = task_service
        self.pr_service = pr_service

    def generate_summary(self) -> Report:
        tasks = self.task_service.get_all_tasks()
        prs = self.pr_service.get_all_prs()
        return Report(
            total_tasks=len(tasks),
            completed_tasks=len([t for t in tasks if t.is_complete()])
        )

# Domain layer
@dataclass
class Report:
    total_tasks: int
    completed_tasks: int

    def completion_percentage(self) -> float:
        return (self.completed_tasks / self.total_tasks) * 100 if self.total_tasks else 0.0

# Entry point
def cmd_report(args):
    report = report_service.generate_summary()
    print(f"Completion: {report.completion_percentage():.1f}%")
```

### Scenario 2: User Model with JSON Formatting
**Q**: "User dataclass that converts to JSON. Where does it go?"

**A**: Domain Layer - Pure data model with serialization

```python
# Domain layer
@dataclass
class User:
    id: str
    username: str
    email: str
    password_hash: str

    def to_json_dict(self) -> dict:
        """Domain logic: what fields to expose."""
        return {
            "id": self.id,
            "username": self.username,
            "email": self.email
            # password_hash excluded
        }

# Entry point (API route)
@app.route('/users/<user_id>')
def get_user(user_id):
    user = user_service.get_user_by_id(user_id)
    return jsonify(user.to_json_dict())
```

### Scenario 3: Database Connection
**Q**: "Connect to PostgreSQL database. Where does this go?"

**A**: Infrastructure Layer - External system integration

```python
# Infrastructure: Repository pattern
class UserRepository:
    def __init__(self, db: PostgreSQLDatabase):
        self.db = db

    def find_by_id(self, user_id: str) -> Optional[User]:
        rows = self.db.execute_query("SELECT * FROM users WHERE id = %s", (user_id,))
        return User.from_db_row(rows[0]) if rows else None

# Service uses repository
class UserService:
    def __init__(self, user_repository: UserRepository):
        self.user_repository = user_repository
```

### Scenario 4: Email Validation
**Q**: "Validate email format. Where does this go?"

**A**: Domain Layer - Pure business rule

```python
@dataclass
class Email:
    address: str

    def __post_init__(self):
        if not self._is_valid_format(self.address):
            raise InvalidEmailError(f"Invalid: {self.address}")

    @staticmethod
    def _is_valid_format(email: str) -> bool:
        return bool(re.match(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$', email))
```

### Scenario 5: Git Operations
**Q**: "Commit, push, create branches. Where does this go?"

**A**: Infrastructure Layer - Wraps external git commands

```python
# Infrastructure wraps git commands
class GitCommandRunner:
    def commit(self, message: str) -> None:
        self._run(["commit", "-m", message])

# Service adds business logic
class GitService:
    def __init__(self, git_runner: GitCommandRunner):
        self.git_runner = git_runner

    def start_feature(self, feature_name: str) -> str:
        """Business logic: naming convention."""
        branch_name = f"feature/{feature_name}"
        self.git_runner.create_branch(branch_name)
        return branch_name
```

## Quick Reference

| Code Type | Layer | Example |
|-----------|-------|---------|
| CLI argument parsing | Entry Point | `argparse`, route definitions |
| Business workflow | Service | Coordinate operations |
| Data model | Domain | `@dataclass`, models |
| API client | Infrastructure | HTTP requests, `gh` CLI |
| JSON formatting | Domain | `to_json_dict()` |
| Database query | Infrastructure | SQL execution |
| Validation rule | Domain | `validate()`, `is_valid()` |
| File I/O | Infrastructure | `open()`, `write()` |
| Response formatting | Entry Point | `jsonify()`, `print()` |
| Multi-service coordination | Service (Composite) | Statistics, reports |

## Key Principles

1. **Entry points orchestrate** - Parse → Instantiate → Call → Format (no business logic)
2. **Services contain business logic** - Coordinate operations, use infrastructure for I/O
3. **Domain is pure** - No dependencies, no I/O, parse strings not files
4. **Infrastructure wraps external systems** - Simple CRUD, no business decisions
5. **Dependency flow is unidirectional** - Entry → Service → Infrastructure/Domain
6. **Configuration flows from entry point** - Read env vars once, inject down

## When Unsure

1. **Does it talk to users/callers?** → Entry Point
2. **Does it call external systems?** → Infrastructure
3. **Is it pure data/rules?** → Domain
4. **Does it coordinate business operations?** → Service

Default to Service layer if uncertain, refactor later if needed.

## Related Skills

- **creating-services**: Learn how to implement services following the Service Layer pattern
- **domain-modeling**: Understand how to create rich domain models with parsing logic
- **cli-architecture**: Detailed patterns for structuring CLI entry points and commands
- **dependency-injection**: Service constructor patterns and configuration flow
- **testing-services**: Layer-based testing strategies and what to mock at each layer
- **python-code-style**: Code organization conventions for each layer

## Further Reading

- [Martin Fowler's Service Layer Pattern](https://martinfowler.com/eaaCatalog/serviceLayer.html)
- [Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html)
- [Clean Architecture by Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
