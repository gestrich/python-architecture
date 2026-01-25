---
name: testing-services
description: Writes unit tests for Python service classes using Arrange-Act-Assert pattern with proper mocking at boundaries. Tests behavior, not implementation. Mocks external systems only (API calls, file I/O, databases). Use when writing tests for services or fixing test coverage.
user-invocable: true
argument-hint: "[service-class-name]"
---

# Testing Services

## When to Use This Skill

Activate this skill when:
- Writing unit tests for service classes
- Need to improve test coverage for services
- Unclear what to mock vs what to test directly
- Writing integration tests that test service interactions

## Quick Reference

### AAA Pattern

```python
def test_service_operation():
    # Arrange: Set up dependencies and test data
    mock_dependency = Mock(spec=Dependency)
    mock_dependency.method.return_value = "expected"
    service = ServiceClass(mock_dependency)

    # Act: Execute the operation under test
    result = service.operation(param="value")

    # Assert: Verify behavior and outcomes
    assert result == "expected_outcome"
    mock_dependency.method.assert_called_once_with("value")
```

### Running Tests

```bash
pytest tests/ -v                              # Run all tests
pytest tests/unit/test_service.py -v          # Run specific file
pytest tests/ --cov=src --cov-report=term-missing  # With coverage
pytest tests/ -k "test_create" -v             # Pattern matching
```

### Coverage Guidelines

- Aim for 85%+ overall coverage
- Focus on testing behavior, not hitting targets
- Missing coverage acceptable for edge cases or trivial code

## Mock at Boundaries Rule

**The Golden Rule**: Mock external systems only, never internal logic.

### What to Mock ✅

**External systems and infrastructure**:
- HTTP API clients, database connections, repositories
- File system operations, subprocess calls
- External service clients (email, payment, etc.)

**Service dependencies**:
- Other services injected via constructor (enables isolation testing)

### What NOT to Mock ❌

**Internal logic**:
- Domain models and dataclasses
- Pure functions, helper methods, static methods
- Business logic calculations
- Exceptions and error classes

**Standard library basics**:
- datetime (use fixed times), string/list/dict operations

### Decision Guide

Ask: "Is this an external dependency or internal logic?"
- External dependency → Mock it
- Internal logic → Test it directly

## Service Test Templates

### Testing a Core Service

```python
from unittest.mock import Mock
import pytest
from src.services.entity_service import EntityService
from src.domain.entity import Entity
from src.domain.exceptions import ValidationError


class TestEntityService:
    """Tests for EntityService."""

    def test_create_entity_success(self):
        # Arrange: Mock external dependencies
        mock_repository = Mock()
        mock_validator = Mock()
        mock_validator.is_valid.return_value = True
        service = EntityService(mock_repository, mock_validator)

        # Act
        result = service.create_entity(name="Test Entity", value=42)

        # Assert: Verify behavior
        assert result.name == "Test Entity"
        assert result.value == 42
        mock_validator.is_valid.assert_called_once()
        mock_repository.save.assert_called_once()

    def test_create_entity_validation_failure(self):
        # Arrange
        mock_repository = Mock()
        mock_validator = Mock()
        mock_validator.is_valid.return_value = False
        service = EntityService(mock_repository, mock_validator)

        # Act & Assert
        with pytest.raises(ValidationError, match="Invalid entity data"):
            service.create_entity(name="Bad", value=-1)

        mock_repository.save.assert_not_called()

    def test_find_by_criteria_filters_correctly(self):
        # Arrange: Use real Entity objects (don't mock domain models!)
        mock_repository = Mock()
        entities = [
            Entity(name="Low", value=10),
            Entity(name="High", value=100),
        ]
        mock_repository.list_all.return_value = entities
        service = EntityService(mock_repository, Mock())

        # Act
        result = service.find_by_criteria(min_value=50)

        # Assert
        assert len(result) == 1
        assert result[0].name == "High"
```

### Testing a Composite Service

```python
from unittest.mock import Mock
import pytest
from src.services.workflow_service import WorkflowService
from src.domain.exceptions import WorkflowError, ValidationError


class TestWorkflowService:
    """Tests for WorkflowService."""

    def test_execute_workflow_success(self):
        # Arrange: Mock all Core service dependencies
        mock_entity_service = Mock()
        mock_notification_service = Mock()
        mock_audit_service = Mock()

        mock_entity = Mock(id="entity-123")
        mock_entity_service.create_entity.return_value = mock_entity

        service = WorkflowService(
            mock_entity_service,
            mock_notification_service,
            mock_audit_service
        )

        # Act
        result = service.execute_workflow(name="Test", value=100, user_id="user-456")

        # Assert: Verify orchestration
        assert result.success is True
        assert result.entity_id == "entity-123"
        mock_entity_service.create_entity.assert_called_once_with("Test", 100)
        mock_notification_service.notify_entity_created.assert_called_once()
        mock_audit_service.log_success.assert_called_once()

    def test_execute_workflow_handles_failure(self):
        # Arrange
        mock_entity_service = Mock()
        mock_entity_service.create_entity.side_effect = ValidationError("Invalid")
        mock_notification_service = Mock()
        mock_audit_service = Mock()

        service = WorkflowService(
            mock_entity_service,
            mock_notification_service,
            mock_audit_service
        )

        # Act & Assert
        with pytest.raises(WorkflowError, match="Entity creation failed"):
            service.execute_workflow(name="Bad", value=-1, user_id="user-456")

        mock_audit_service.log_failure.assert_called_once()
        mock_notification_service.notify_entity_created.assert_not_called()

    def test_calculate_priority_static_method(self):
        # Static methods are pure - test directly, no mocking needed
        result = WorkflowService.calculate_priority(value=200, user_tier="premium")
        assert result == 4

        result = WorkflowService.calculate_priority(value=150, user_tier="standard")
        assert result == 1
```

## Key Testing Patterns

### Pattern: Use Real Domain Models

```python
def test_process_order():
    mock_repository = Mock()
    service = OrderService(mock_repository)

    # Use real Order domain model, not a mock
    result = service.process_order(customer_id="c123", items=["item1"], total=99.99)

    assert isinstance(result, Order)
    assert result.customer_id == "c123"
    assert result.total == 99.99
```

### Pattern: Test Exception Handling

```python
def test_find_user_not_found():
    mock_repository = Mock()
    mock_repository.get_by_id.return_value = None
    service = UserService(mock_repository)

    with pytest.raises(UserNotFoundError, match="User u999 not found"):
        service.find_user("u999")
```

### Pattern: Verify Side Effects

```python
def test_send_notification():
    mock_email_client = Mock()
    service = NotificationService(mock_email_client)

    service.send_notification(user_email="user@example.com", message="Hello!")

    mock_email_client.send.assert_called_once_with(
        to="user@example.com",
        subject="Notification",
        body="Hello!"
    )
```

### Pattern: Parametrized Tests

```python
import pytest

@pytest.mark.parametrize("value,tier,expected", [
    (100, "premium", 2),
    (100, "standard", 1),
    (200, "premium", 4),
])
def test_calculate_priority(value, tier, expected):
    result = WorkflowService.calculate_priority(value, tier)
    assert result == expected
```

## Mock vs MagicMock vs Real Objects

### Use `Mock(spec=ClassName)`

For most scenarios - provides autocomplete and catches attribute errors.

```python
mock_repository = Mock(spec=UserRepository)
mock_repository.find_by_id.return_value = User(id="123")
```

### Use `MagicMock`

When you need magic methods (`__len__`, `__iter__`, `__getitem__`).

```python
from unittest.mock import MagicMock

mock_collection = MagicMock()
mock_collection.__len__.return_value = 5
```

### Use Real Objects

For domain models, value objects, exceptions.

```python
user = User(id="123", name="Alice")  # Real domain model
order = Order(items=[Item("book")])  # Real value objects

mock_service.process.side_effect = ValidationError("Bad data")  # Real exception
```

## Fixtures for Reusable Setup

```python
import pytest
from unittest.mock import Mock

@pytest.fixture
def mock_repository():
    return Mock(spec=UserRepository)

@pytest.fixture
def user_service(mock_repository):
    return UserService(mock_repository)

def test_create_user(user_service, mock_repository):
    mock_repository.save.return_value = User(id="123")
    result = user_service.create_user(name="Alice")
    assert result.id == "123"
```

## Integration Tests

Integration tests verify services work together - only mock infrastructure.

```python
def test_workflow_integration():
    # Mock infrastructure only
    mock_database = Mock()
    mock_email_client = Mock()

    # Use real services
    entity_service = EntityService(mock_database)
    notification_service = NotificationService(mock_email_client)
    workflow_service = WorkflowService(entity_service, notification_service)

    # Test real interaction
    result = workflow_service.execute_workflow(name="Test", value=100, user_id="u123")

    assert result.success is True
    mock_database.save.assert_called()
    mock_email_client.send.assert_called()
```

## One Concept Per Test

Each test should verify one specific behavior.

```python
# WRONG: Tests multiple things
def test_user_service():
    service = UserService(mock_repo)
    user = service.create_user("Alice")
    assert user.name == "Alice"
    found = service.find_user(user.id)
    service.delete_user(user.id)  # ❌

# RIGHT: Separate tests
def test_create_user_sets_name():
    service = UserService(mock_repo)
    user = service.create_user("Alice")
    assert user.name == "Alice"

def test_find_user_returns_existing():
    service = UserService(mock_repo)
    mock_repo.get_by_id.return_value = User(id="123")
    found = service.find_user("123")
    assert found.id == "123"
```

## Common Anti-Patterns

### ❌ Mocking Internal Logic

```python
# WRONG
def test_bad():
    service = UserService(mock_repo)
    with patch.object(UserService, 'validate_email', return_value=True):
        service.create_user(email="test@example.com")  # ❌ Not testing real behavior

# RIGHT
def test_good():
    service = UserService(mock_repo)
    service.create_user(email="test@example.com")  # ✓ Tests real validation
```

### ❌ Mocking Domain Models

```python
# WRONG
mock_user = Mock(spec=User)  # ❌

# RIGHT
user = User(id="123", name="Alice")  # ✓
```

### ❌ Testing Implementation Details

```python
# WRONG
def test_bad():
    service.create_user("Alice")
    assert service._validate_name.called  # ❌ Internal implementation

# RIGHT
def test_good():
    user = service.create_user("Alice")
    assert user.name == "Alice"  # ✓ Observable behavior
```

### ❌ Not Using spec= Parameter

```python
# WRONG
mock_repo = Mock()
mock_repo.typo_method.return_value = "data"  # No error caught

# RIGHT
mock_repo = Mock(spec=UserRepository)
# mock_repo.typo_method raises AttributeError ✓
```

## Coverage Guidance

**Skip testing**:
- Trivial getters/setters
- Simple `__repr__` or `__str__`
- Framework boilerplate
- One-line defensive code

**Focus coverage on**:
- Business logic in services
- Complex conditionals and loops
- Error handling for important cases
- Integration points between services

## Test Organization

```
tests/
├── unit/
│   ├── services/
│   │   ├── test_entity_service.py
│   │   └── test_workflow_service.py
│   └── domain/
│       └── test_models.py
├── integration/
│   └── test_workflows.py
└── conftest.py  # Shared fixtures
```

**Naming**:
- Files: `test_<module_name>.py`
- Classes: `Test<ClassName>`
- Functions: `test_<behavior_being_tested>`

## Further Reading

- [Martin Fowler on Testing](https://martinfowler.com/testing/)
- [pytest Documentation](https://docs.pytest.org/)
- [unittest.mock Documentation](https://docs.python.org/3/library/unittest.mock.html)
