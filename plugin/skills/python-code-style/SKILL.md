---
name: python-code-style
description: Follow Python code organization conventions including method ordering, datetime handling, circular import avoidance, and type annotations. Use when organizing service classes, handling dates, or structuring modules.
user-invocable: true
argument-hint: "[topic]"
---

# Python Code Style

Python code organization conventions for clean, maintainable codebases covering method ordering, datetime handling, circular import avoidance, and modern type annotations.

## When to Use This Skill

Use this skill when:
- Organizing methods in service classes or modules
- Working with datetime objects and timestamps
- Resolving circular import issues
- Adding type annotations to code
- Structuring function-based modules

## Quick Reference

### Method Organization Hierarchy

```python
class MyService:
    # 1. Special methods (__init__, __str__, etc.)
    def __init__(self):
        pass

    # 2. Class methods
    @classmethod
    def from_config(cls, config: dict):
        pass

    # 3. Static methods
    @staticmethod
    def parse_id(value: str):
        pass

    # 4. Instance methods (public → high-level → low-level)
    def process_request(self):  # Public, high-level
        pass

    def validate_input(self):  # Public, mid-level
        pass

    # 5. Private helper methods
    def _internal_helper(self):
        pass
```

### Datetime Usage

```python
from datetime import datetime, timezone

# ✅ Always timezone-aware
now = datetime.now(timezone.utc)
timestamp = datetime(2025, 1, 15, 10, 30, 0, tzinfo=timezone.utc)

# ❌ Never naive
now = datetime.now()  # Missing timezone
```

## Principle: Method Organization

### Public Before Private

Public methods (part of the API) appear before private/internal methods (prefixed with `_`). This allows developers to understand the public API of each service at a glance.

### Anti-Pattern (❌ Avoid)

```python
class UserService:
    def __init__(self):
        pass

    # ❌ Private method before public methods
    def _validate_email(self, email: str):
        pass

    # ❌ Public methods scattered after private ones
    def create_user(self, email: str, name: str):
        return self._validate_email(email)

    def _hash_password(self, password: str):
        pass

    def authenticate(self, email: str, password: str):
        pass
```

**Problems:**
- Readers must scan through implementation details to find the public API
- Harder to understand what the service does at a glance
- Inconsistent organization makes navigation difficult

### Recommended Pattern (✅ Use This)

```python
class UserService:
    def __init__(self, db: Database):
        self.db = db

    # Public API methods first (high-level to low-level)
    def create_user(self, email: str, name: str) -> User:
        """Create a new user account."""
        self._validate_email(email)
        return User(email=email, name=name)

    def authenticate(self, email: str, password: str) -> bool:
        """Authenticate user credentials."""
        user = self.db.find_user(email)
        return self._verify_password(user, password)

    # Private helper methods last
    def _validate_email(self, email: str):
        """Validate email format."""
        if "@" not in email:
            raise ValueError("Invalid email")

    def _verify_password(self, user: User, password: str) -> bool:
        """Verify password hash."""
        return self._hash_password(password) == user.password_hash

    def _hash_password(self, password: str) -> str:
        """Hash password for storage."""
        return hashlib.sha256(password.encode()).hexdigest()
```

### Standard Method Order

Methods follow this ordering:

1. **Special methods** (`__init__`, `__str__`, `__repr__`, etc.)
2. **Class methods** (`@classmethod`)
3. **Static methods** (`@staticmethod`)
4. **Instance methods** (public, ordered by abstraction level)
5. **Private methods** (prefixed with `_`)

```python
class ReportService:
    def __init__(self, repo: str):
        self.repo = repo

    def __str__(self):
        return f"ReportService({self.repo})"

    # Class methods
    @classmethod
    def from_config(cls, config: dict):
        return cls(config["repo"])

    # Static methods
    @staticmethod
    def validate_format(fmt: str):
        return fmt in ["json", "csv", "html"]

    # Public instance methods (high-level first)
    def generate_report(self, data: list) -> str:
        validated = self._validate_data(data)
        return self._format_output(validated)

    def export_report(self, report: str, path: str):
        self._write_file(path, report)

    # Private helpers
    def _validate_data(self, data: list):
        return [item for item in data if item]

    def _format_output(self, data: list) -> str:
        return "\n".join(str(item) for item in data)

    def _write_file(self, path: str, content: str):
        with open(path, "w") as f:
            f.write(content)
```

### Benefits

✅ **Easier onboarding**: New developers quickly understand what a service does

✅ **Better maintainability**: Clear separation between public contracts and implementation

✅ **Intuitive navigation**: Consistent structure helps developers find methods quickly

✅ **Clearer API boundaries**: Public vs. private methods are visually distinct

## Principle: Section Headers for Complex Services

### When to Use Section Headers

For services with many methods (10+), use descriptive section headers with separators to group related functionality.

### Simple Section Comments

```python
class SimpleService:
    def __init__(self):
        pass

    # Public API methods
    def operation_one(self):
        pass

    def operation_two(self):
        pass

    # Static utility methods
    @staticmethod
    def utility_function():
        pass

    # Private helper methods
    def _internal_helper(self):
        pass
```

### Descriptive Headers for Complex Services

```python
class ComplexService:
    def __init__(self):
        pass

    # ============================================================
    # Core CRUD Operations
    # ============================================================

    def create_resource(self, data: dict):
        pass

    def get_resource(self, id: str):
        pass

    def update_resource(self, id: str, data: dict):
        pass

    def delete_resource(self, id: str):
        pass

    # ============================================================
    # Query Operations
    # ============================================================

    def find_resources(self, filter: dict):
        pass

    def count_resources(self, filter: dict):
        pass

    # ============================================================
    # Utility Operations
    # ============================================================

    @staticmethod
    def parse_identifier(text: str):
        pass

    @staticmethod
    def validate_data(data: dict):
        pass
```

### Benefits

✅ **Visual organization**: Separators make different sections obvious

✅ **Faster navigation**: Developers quickly jump to relevant sections

✅ **Logical grouping**: Related methods are clearly grouped together

## Principle: Module-Level Code Organization

### Function-Based Modules

For modules with functions rather than classes, use this order:

1. **Dataclasses and models** (public before private)
2. **Public API functions** (high-level to low-level)
3. **Module utilities** (helper functions used by the public API)
4. **Private helper functions** (prefixed with `_`)

### Recommended Pattern (✅ Use This)

```python
# artifact_operations.py

from dataclasses import dataclass
from typing import List

# ============================================================
# Public Models
# ============================================================

@dataclass
class ProjectArtifact:
    """Public dataclass for artifact data."""
    name: str
    size: int
    created_at: str

@dataclass
class ArtifactMetadata:
    """Metadata for artifact collections."""
    total_count: int
    total_size: int

# ============================================================
# Public API Functions
# ============================================================

def find_artifacts(project: str) -> List[ProjectArtifact]:
    """Highest-level public function - main entry point."""
    raw_data = _fetch_from_api(project)
    return [_parse_artifact(item) for item in raw_data]

def get_artifact_details(artifact: ProjectArtifact) -> dict:
    """Mid-level public function - supporting operation."""
    return {
        "name": artifact.name,
        "size_mb": artifact.size / 1024 / 1024,
        "age_days": calculate_age(artifact.created_at)
    }

# ============================================================
# Module Utilities
# ============================================================

def parse_artifact_name(name: str) -> tuple[str, str]:
    """Utility function used by multiple operations."""
    parts = name.split("-")
    return parts[0], "-".join(parts[1:])

def calculate_age(timestamp: str) -> int:
    """Calculate age in days from timestamp."""
    # Implementation
    pass

# ============================================================
# Private Helper Functions
# ============================================================

def _fetch_from_api(project: str) -> list:
    """Private implementation detail - API interaction."""
    # Implementation
    pass

def _parse_artifact(data: dict) -> ProjectArtifact:
    """Private implementation detail - data parsing."""
    return ProjectArtifact(
        name=data["name"],
        size=data["size"],
        created_at=data["created_at"]
    )
```

### Benefits

✅ **Clear structure**: Module organization is consistent and predictable

✅ **Easy to find**: Related code is grouped logically

✅ **Public API first**: Readers see the module's purpose immediately

## Principle: Always Use Timezone-Aware Datetimes

**All datetime objects must be timezone-aware.** Naive datetimes (without timezone information) are not allowed and will cause validation errors.

### Anti-Pattern (❌ Avoid)

```python
from datetime import datetime

# ❌ Naive datetime - no timezone information
now = datetime.now()
timestamp = datetime(2025, 1, 15, 10, 30, 0)
utc_now = datetime.utcnow()  # ❌ Deprecated and naive

# Problems with naive datetimes:
# - Ambiguous: Which timezone does this represent?
# - Cannot compare naive and timezone-aware datetimes
# - Serialized without timezone, leading to misinterpretation
```

### Recommended Pattern (✅ Use This)

```python
from datetime import datetime, timezone

# ✅ Always use timezone-aware datetimes
now = datetime.now(timezone.utc)
timestamp = datetime(2025, 1, 15, 10, 30, 0, tzinfo=timezone.utc)

# ✅ Use UTC for all internal operations
created_at = datetime.now(timezone.utc)
last_updated = datetime.now(timezone.utc)

# ✅ Serialize to ISO 8601 with timezone
timestamp_str = now.isoformat()  # "2025-01-15T10:30:00+00:00"
```

### Parsing ISO 8601 Timestamps

Use a helper function for parsing ISO 8601 timestamps:

```python
from datetime import datetime, timezone

def parse_iso_timestamp(timestamp_str: str) -> datetime:
    """Parse ISO 8601 timestamp to timezone-aware datetime.

    Handles both "Z" and "+00:00" timezone formats.
    """
    if timestamp_str.endswith("Z"):
        timestamp_str = timestamp_str[:-1] + "+00:00"

    dt = datetime.fromisoformat(timestamp_str)

    # Ensure timezone-aware
    if dt.tzinfo is None:
        dt = dt.replace(tzinfo=timezone.utc)

    return dt

# Usage
parsed_dt = parse_iso_timestamp("2025-01-15T10:30:00Z")
parsed_dt2 = parse_iso_timestamp("2025-01-15T10:30:00+00:00")
```

### Domain Model Validation

Validate timezone-aware datetimes in domain models:

```python
from dataclasses import dataclass
from datetime import datetime, timezone

@dataclass
class Event:
    name: str
    created_at: datetime

    def __post_init__(self):
        """Validate that datetime fields are timezone-aware."""
        if self.created_at.tzinfo is None:
            raise ValueError(
                f"created_at must be timezone-aware. "
                f"Use datetime.now(timezone.utc) or parse_iso_timestamp()"
            )

# ✅ Valid - timezone-aware
event = Event("deployment", datetime.now(timezone.utc))

# ❌ Raises ValueError - naive datetime
event = Event("deployment", datetime.now())
```

### Why UTC?

✅ **Unambiguous**: UTC has no daylight saving time

✅ **Standard**: Industry best practice for system timestamps

✅ **Interoperable**: Works across all timezones

✅ **Comparable**: All timestamps in same timezone can be directly compared

### Common Pitfalls

| ❌ Avoid | ✅ Use Instead |
|---------|---------------|
| `datetime.now()` | `datetime.now(timezone.utc)` |
| `datetime.utcnow()` (deprecated) | `datetime.now(timezone.utc)` |
| `datetime(2025, 1, 15, 10, 30)` | `datetime(2025, 1, 15, 10, 30, tzinfo=timezone.utc)` |
| `datetime.fromisoformat(iso_str)` | `parse_iso_timestamp(iso_str)` |

### Benefits

✅ **Prevents comparison errors**: No "can't compare offset-naive and offset-aware datetimes"

✅ **Unambiguous data**: Timestamps clearly indicate their timezone

✅ **ISO 8601 compliant**: Standard format works everywhere

✅ **Validated**: Domain models prevent naive datetimes at construction time

## Principle: Avoid Circular Imports

Circular imports indicate architectural problems. Fix the dependency structure rather than working around it.

### Anti-Pattern (❌ Avoid)

```python
# ❌ BAD: TYPE_CHECKING guard
from typing import TYPE_CHECKING, List

if TYPE_CHECKING:
    from module_b import SomeClass  # Only imported during type checking

def process_items(items: List["SomeClass"]):  # String annotation workaround
    pass

# ❌ BAD: Using Any to avoid circular import
from typing import Any, List

def process_items(items: List[Any]):  # Lost type safety
    pass
```

**Problems:**
- `TYPE_CHECKING` means your code works at runtime only because the import is skipped
- `Any` throws away type safety entirely
- Both hide the architectural issue rather than solving it

### Recommended Pattern (✅ Use This)

Fix the dependency structure by establishing one-way dependencies:

1. **Identify which module is more foundational** (lower-level)
2. **Ensure the lower-level module never imports from higher-level ones**
3. **Verify with grep**: `grep "from higher_module" lower_module.py` should return nothing

```python
# lower_level.py - No imports from higher_level.py
from dataclasses import dataclass

@dataclass
class GitHubPullRequest:
    number: int
    title: str
    state: str

# higher_level.py - Can import from lower_level.py
from typing import List
from lower_level import GitHubPullRequest  # ✅ One-way dependency

@dataclass
class ProjectStats:
    project_name: str
    open_prs: List[GitHubPullRequest]  # ✅ Proper typing, no strings
```

### When You Have a Genuine Cycle

If you have a genuine cycle, either:

1. **Move shared code to a third module** that both can depend on
2. **Refactor** so the lower-level module doesn't need higher-level types

```python
# Before: Circular dependency
# module_a.py imports from module_b.py
# module_b.py imports from module_a.py

# After: Shared module breaks the cycle
# shared.py - Common types both modules need
@dataclass
class SharedType:
    pass

# module_a.py - Imports from shared
from shared import SharedType

# module_b.py - Imports from shared
from shared import SharedType
```

### Benefits

✅ **Clean architecture**: One-way dependency graphs are easier to understand

✅ **No runtime surprises**: All imports work at runtime, not just type checking

✅ **Better modularity**: Lower-level modules are more reusable

✅ **Full type safety**: No need for `Any` or string annotations

## Principle: Modern Type Annotations

### Use `Self` for Factory Methods

When a classmethod or factory returns an instance of its own class, use `Self` instead of quoted string annotations:

```python
# ❌ Avoid: Quoted string annotation
class User:
    @classmethod
    def from_string(cls, value: str) -> "User":
        return cls()

# ✅ Use: Self type (Python 3.11+)
from typing import Self

class User:
    @classmethod
    def from_string(cls, value: str) -> Self:
        return cls()

    @classmethod
    def from_dict(cls, data: dict) -> Self:
        return cls()
```

**Benefits:**
- Cleaner syntax
- Works correctly with subclasses
- Avoids forward reference strings

### Use `from __future__ import annotations` for Self-Referential Types

For classes that reference themselves in type hints (tree structures, containers with nested elements):

```python
# ✅ Use: Deferred annotations for self-referential types
from __future__ import annotations

from dataclasses import dataclass
from typing import List, Optional

@dataclass
class Section:
    name: str
    elements: List[Section]  # ✅ No quotes needed - deferred evaluation
    parent: Optional[Section] = None

    def add(self, element: Section) -> Section:  # ✅ Clean type hints
        self.elements.append(element)
        element.parent = self
        return self

# Without __future__ import, you'd need:
# elements: List["Section"]  # ❌ Less readable
```

### Benefits

✅ **Cleaner code**: No quoted string annotations

✅ **Works with subclasses**: `Self` adapts to the actual class

✅ **Better readability**: Self-referential types look natural

✅ **Forward compatible**: Prepares for future Python versions

## Related Patterns

This skill relates to several software design patterns:

- **Template Method Pattern**: Method organization makes it easier to see which methods to override
- **Strategy Pattern**: Static methods are good candidates for strategy implementations
- **Factory Pattern**: Class methods serve as alternative constructors
- **Value Object Pattern**: Timezone-aware datetimes make better value objects

## Related Skills

- **creating-services**: Apply method organization principles when creating service classes
- **testing-services**: Use consistent organization for test code readability
- **domain-modeling**: Organize domain model methods following these conventions
- **dependency-injection**: Constructor organization relates to dependency management

## Further Reading

- [PEP 8 - Style Guide for Python Code](https://peps.python.org/pep-0008/)
- [Python datetime documentation](https://docs.python.org/3/library/datetime.html)
- [PEP 563 - Postponed Evaluation of Annotations](https://peps.python.org/pep-0563/)
- [PEP 673 - Self Type](https://peps.python.org/pep-0673/)
- [ISO 8601 DateTime Standard](https://en.wikipedia.org/wiki/ISO_8601)
