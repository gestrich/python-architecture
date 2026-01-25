---
name: creating-services
description: Creates new Python service classes following Martin Fowler's Service Layer pattern with constructor-based dependency injection, proper layering (Core vs Composite), and static vs instance method conventions. Use when adding new services or refactoring business logic into service classes.
user-invocable: true
argument-hint: "[service-name] [core|composite]"
---

# Creating Services

## When to Use This Skill

Activate this skill when:
- Creating a new service class to encapsulate business logic
- Refactoring business logic from commands/controllers into services
- Need guidance on Core vs Composite service distinction
- Unclear whether to use instance vs static methods

## Quick Reference

### Constructor Pattern
```python
class ServiceName:
    def __init__(self, dependency1: Type1, dependency2: Type2):
        """Initialize with required dependencies.

        Args:
            dependency1: Description of what this provides
            dependency2: Description of what this provides
        """
        self.dependency1 = dependency1
        self.dependency2 = dependency2
```

**Key rules**:
- All dependencies via constructor parameters
- NO optional dependencies with default factories
- NO reading environment variables or config files inside services
- Dependencies are either other services or infrastructure components

**Note**: For detailed patterns on dependency injection, see the **dependency-injection** skill.

### Method Organization

Order methods: Constructor → Public → Static → Private (high-level before low-level)

```python
class ServiceName:
    def __init__(self, ...): ...

    def main_operation(self): ...  # Public high-level

    @staticmethod
    def helper_function(param: str) -> str: ...  # Pure functions

    def _internal_helper(self): ...  # Private implementation
```

## Core vs Composite Services

### Core Services
**Purpose**: Single responsibility, focused operations

**Characteristics**:
- Perform ONE type of business operation
- Depend on infrastructure components (repos, APIs, file systems)
- Minimal dependencies on other services
- Directly interact with domain models

**Examples**: `TaskService`, `EmailService`, `ValidationService`, `ReportGenerator`

### Composite Services
**Purpose**: Coordinate multiple Core services for complex workflows

**Characteristics**:
- Orchestrate multiple Core services
- Implement cross-cutting workflows
- Little to no direct infrastructure access (delegates to Core services)

**Examples**: `StatisticsService`, `OnboardingService`, `CheckoutService`

### Decision Guide

1. Does this service do ONE thing? → Core
2. Does it coordinate multiple services? → Composite
3. Does it directly use infrastructure (DB, API, filesystem)? → Likely Core
4. Does it delegate all infrastructure work to other services? → Likely Composite

## Service Templates

**Note**: Services often work with domain models. For guidance on creating the models used by services, see the **domain-modeling** skill.

### Core Service Template

```python
from typing import List, Optional


class EntityService:
    """Handles business operations for Entity domain.

    Core service with single responsibility.
    """

    def __init__(self, repository: EntityRepository, validator: EntityValidator):
        """Initialize with required dependencies.

        Args:
            repository: Provides Entity persistence operations
            validator: Validates Entity business rules
        """
        self.repository = repository
        self.validator = validator

    def create_entity(self, name: str, value: int) -> Entity:
        """Create a new entity with validation.

        Args:
            name: Entity name
            value: Entity value

        Returns:
            The created Entity

        Raises:
            ValidationError: If entity data is invalid
        """
        entity_data = {"name": name, "value": value}

        if not self.validator.is_valid(entity_data):
            raise ValidationError(f"Invalid entity data: {entity_data}")

        entity = Entity(name=name, value=value)
        self.repository.save(entity)
        return entity

    def find_by_criteria(self, min_value: int) -> List[Entity]:
        """Find entities matching criteria."""
        all_entities = self.repository.list_all()
        return self._filter_by_value(all_entities, min_value)

    @staticmethod
    def _filter_by_value(entities: List[Entity], min_value: int) -> List[Entity]:
        """Filter entities by minimum value.

        Static because it's a pure function with no state dependency.
        """
        return [e for e in entities if e.value >= min_value]
```

### Composite Service Template

```python
class WorkflowService:
    """Coordinates multiple services for complex workflow.

    Composite service that orchestrates Core services.
    """

    def __init__(
        self,
        entity_service: EntityService,
        notification_service: NotificationService,
        audit_service: AuditService,
    ):
        """Initialize with Core service dependencies."""
        self.entity_service = entity_service
        self.notification_service = notification_service
        self.audit_service = audit_service

    def execute_workflow(self, name: str, value: int, user_id: str) -> WorkflowResult:
        """Execute complex workflow across multiple services.

        Args:
            name: Entity name
            value: Entity value
            user_id: User executing workflow

        Returns:
            WorkflowResult with status and details

        Raises:
            WorkflowError: If workflow fails at any step
        """
        # Step 1: Create entity (delegates to Core service)
        try:
            entity = self.entity_service.create_entity(name, value)
        except ValidationError as e:
            self.audit_service.log_failure(user_id, "create_entity", str(e))
            raise WorkflowError(f"Entity creation failed: {e}")

        # Step 2: Send notification
        self.notification_service.notify_entity_created(entity.id, user_id)

        # Step 3: Audit log
        self.audit_service.log_success(user_id, "execute_workflow", entity.id)

        return WorkflowResult(
            success=True,
            entity_id=entity.id,
            message=f"Workflow completed for entity {entity.id}",
        )

    @staticmethod
    def calculate_priority(value: int, user_tier: str) -> int:
        """Calculate workflow priority.

        Static because it's pure business logic with no state dependency.
        """
        base_priority = value // 100
        tier_multiplier = {"premium": 2, "standard": 1, "basic": 1}
        return base_priority * tier_multiplier.get(user_tier, 1)
```

## Static vs Instance Methods

### Use Instance Methods When:
- Need to access `self.dependency` (injected dependencies)
- Perform I/O operations (database, API calls, file system)
- Modify or read instance state

### Use Static Methods When:
- Pure function with NO state dependency
- Helper that operates only on parameters
- Business logic calculation that doesn't need services
- Can be tested without instantiating the service

### Example

```python
class ReportService:
    def __init__(self, database: Database):
        self.database = database

    # INSTANCE: Needs database dependency
    def generate_report(self, user_id: str) -> Report:
        data = self.database.fetch_user_data(user_id)
        return self._build_report(data)

    # STATIC: Pure calculation, no dependencies
    @staticmethod
    def calculate_score(correct: int, total: int) -> float:
        return (correct / total * 100) if total > 0 else 0.0

    # INSTANCE: Calls other instance methods
    def _build_report(self, data: dict) -> Report:
        score = self.calculate_score(data["correct"], data["total"])
        return Report(score=score, data=data)
```

**Common mistake**: Making static methods that need dependencies

```python
# WRONG
class UserService:
    def __init__(self, database: Database):
        self.database = database

    @staticmethod  # ❌ Needs database!
    def find_user(database: Database, user_id: str) -> User:
        return database.fetch_user(user_id)

# RIGHT
class UserService:
    def __init__(self, database: Database):
        self.database = database

    def find_user(self, user_id: str) -> User:  # ✓
        return self.database.fetch_user(user_id)
```

## Service Instantiation

Services are instantiated at entry points (CLI commands, API handlers) using constructor injection.

**Note**: For detailed command-level service instantiation patterns in CLI applications, see the **cli-architecture** skill.

### In CLI Commands

```python
def cmd_generate_report(args):
    # 1. Get configuration
    repo = os.environ.get("GITHUB_REPOSITORY", "")

    # 2. Initialize infrastructure
    database = DatabaseClient(repo)
    file_system = FileSystemWriter()

    # 3. Initialize Core services
    data_service = DataService(database)
    formatter_service = FormatterService()

    # 4. Initialize Composite service
    report_service = ReportService(data_service, formatter_service, file_system)

    # 5. Use service
    report = report_service.generate_report(args.report_type)

    # 6. Handle output (CLI layer responsibility)
    print(f"Report generated: {report.path}")
```

### In API Handlers

```python
@app.route('/api/entities', methods=['POST'])
def create_entity():
    # 1. Parse request
    data = request.get_json()

    # 2. Initialize dependencies
    repository = EntityRepository(db_connection)
    validator = EntityValidator()
    service = EntityService(repository, validator)

    # 3. Use service
    try:
        entity = service.create_entity(name=data['name'], value=data['value'])
    except ValidationError as e:
        return jsonify({"error": str(e)}), 400

    # 4. Format response
    return jsonify({"id": entity.id, "name": entity.name}), 201
```

**Dependency Flow**: Entry Point → Infrastructure → Core Services → Composite Services

**Key principle**: Configuration flows from entry point to services. Services never read environment variables directly.

## Error Handling

Services should fail fast and raise exceptions for abnormal conditions.

```python
class TaskService:
    def find_task(self, task_id: str) -> Task:
        task = self.repository.get_by_id(task_id)

        # Raise exception for abnormal case
        if task is None:
            raise TaskNotFoundError(f"Task {task_id} not found")

        return task

    def list_tasks(self, filter_status: Optional[str] = None) -> List[Task]:
        tasks = self.repository.list_all()

        # Empty list is normal, not exceptional
        if filter_status:
            tasks = [t for t in tasks if t.status == filter_status]

        return tasks  # Can be empty
```

**Define domain exceptions** in domain layer, raise them in services for abnormal cases.

## Testing Services

Mock dependencies (Arrange-Act-Assert pattern), not internal logic.

```python
from unittest.mock import Mock
import pytest

def test_create_entity_success():
    # Arrange: Mock dependencies
    mock_repository = Mock(spec=EntityRepository)
    mock_validator = Mock(spec=EntityValidator)
    mock_validator.is_valid.return_value = True
    service = EntityService(mock_repository, mock_validator)

    # Act
    result = service.create_entity(name="Test", value=42)

    # Assert
    assert result.name == "Test"
    mock_repository.save.assert_called_once()
```

**Mock**: Dependencies (repositories, other services). **Don't mock**: Domain models, static methods, internal logic.


## Anti-Patterns to Avoid

### ❌ Default Factory Dependencies

```python
# WRONG
class BadService:
    def __init__(self, repository: Optional[Repository] = None):
        self.repository = repository or Repository()  # ❌

# RIGHT
class GoodService:
    def __init__(self, repository: Repository):
        self.repository = repository
```

### ❌ Reading Config Inside Services

```python
# WRONG
class BadService:
    def __init__(self):
        self.api_key = os.getenv("API_KEY")  # ❌

# RIGHT
class GoodService:
    def __init__(self, api_key: str):
        self.api_key = api_key
```

### ❌ Mixing Layers

```python
# WRONG: Service does CLI output
class BadService:
    def process_data(self, data: str):
        result = self._transform(data)
        print(f"Processed: {result}")  # ❌
        return result

# RIGHT: Service returns data
class GoodService:
    def process_data(self, data: str) -> str:
        return self._transform(data)

# CLI handles output
result = service.process_data(data)
print(f"Processed: {result}")  # ✓
```

### ❌ God Services

```python
# WRONG: One service does everything
class GodService:
    def create_user(self): ...
    def send_email(self): ...
    def process_payment(self): ...
    def generate_report(self): ...
    # ❌ Too many responsibilities

# RIGHT: Split into focused services
class UserService:
    def create_user(self): ...

class EmailService:
    def send_email(self): ...

class PaymentService:
    def process_payment(self): ...
```

## Related Skills

- **dependency-injection**: Detailed patterns for constructor-based dependency injection and configuration flow
- **domain-modeling**: Creating rich domain models that services use and orchestrate
- **cli-architecture**: Command-level service instantiation and CLI structure patterns
- **python-code-style**: Method organization conventions for service classes
- **testing-services**: Testing services with mocked dependencies
- **identifying-layer-placement**: Understanding where services fit in the overall architecture

## Further Reading

- [Martin Fowler's Service Layer Pattern](https://martinfowler.com/eaaCatalog/serviceLayer.html)
- [Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html)
- [Dependency Injection Principles](https://martinfowler.com/articles/injection.html)
