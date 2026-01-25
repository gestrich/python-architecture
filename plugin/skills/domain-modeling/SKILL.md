---
name: domain-modeling
description: Design domain models following parse-once principle with type-safe APIs, factory methods, and Repository pattern. Use when creating domain models, parsing data structures, or organizing business logic in models.
user-invocable: true
argument-hint: "[model-name]"
---

# Domain Modeling

## When to Use This Skill

Activate this skill when:
- Creating domain models that encapsulate parsing logic
- Working with structured data (YAML, JSON, XML, markdown)
- Converting raw strings/dictionaries into type-safe objects
- Implementing the Repository pattern
- Centralizing data validation and parsing

## Quick Reference

### Parse-Once Pattern
```python
# Domain model with factory method
@dataclass
class ProjectConfiguration:
    """Parsed configuration with validation"""
    project: Project
    reviewers: List[Reviewer]

    @classmethod
    def from_yaml_string(cls, project: Project, content: str) -> 'ProjectConfiguration':
        """Parse once, validate, return type-safe model"""
        config = yaml.safe_load(content)

        if "reviewers" not in config:
            raise ConfigurationError("Missing reviewers field")

        reviewers = [
            Reviewer(username=r["username"], max_open_prs=r.get("maxOpenPRs", 2))
            for r in config["reviewers"]
            if "username" in r
        ]

        return cls(project=project, reviewers=reviewers)

# Repository fetches and parses
class ProjectRepository:
    def load_configuration(self, project: Project) -> Optional[ProjectConfiguration]:
        content = get_file_from_branch(self.repo, project.config_path)
        return ProjectConfiguration.from_yaml_string(project, content)

# Service uses type-safe API
config = repository.load_configuration(project)
usernames = [r.username for r in config.reviewers]  # Type-safe!
```

## Principle: Parse Once Into Well-Formed Models

When working with structured data (YAML files, JSON responses, markdown files, etc.), **parse the data once into a well-formed domain model** rather than passing raw strings or dictionaries around and parsing them repeatedly.

## Anti-Pattern (❌ Avoid)

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

## Recommended Pattern (✅ Use This)

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

## Benefits

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

## Architecture Pattern

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

## Key Principles

1. **Parse at boundaries**: Convert raw data to domain models as soon as it enters your system
2. **Parse once**: Never re-parse the same data in multiple places
3. **Domain owns parsing**: Parsing logic belongs in domain models, not services
4. **Type-safe APIs**: Models expose strongly-typed properties and methods
5. **Validate early**: Domain model constructors/factories validate structure
6. **No leaky abstractions**: Services don't know about YAML, regex, or file formats

## Related Patterns

This approach aligns with:
- **Repository Pattern**: Infrastructure fetches, domain parses
- **Factory Pattern**: `from_yaml_string()`, `from_branch_name()` constructors
- **Domain-Driven Design**: Rich domain models with behavior
- **Dependency Inversion**: Services depend on domain abstractions, not infrastructure

## Related Skills

- **creating-services**: Learn how to use domain models in service classes
- **testing-services**: Understand how to test domain models and services that use them
- **identifying-layer-placement**: Learn where domain models fit in the architecture

## Further Reading

- [Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html) by Martin Fowler
- [Repository Pattern](https://martinfowler.com/eaaCatalog/repository.html) by Martin Fowler
- [Parse, Don't Validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) by Alexis King
