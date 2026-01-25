# Testing Philosophy

This document describes the testing architecture and philosophy for the ClaudeChain project. It explains **why** we test the way we do and provides guidance on how to approach testing new features.

## Table of Contents

- [Testing Philosophy](#testing-philosophy-1)
- [Testing Principles](#testing-principles)
- [Test Architecture Overview](#test-architecture-overview)
- [Testing by Layer](#testing-by-layer)
- [Quick Start](#quick-start)
- [Test Style Guide](#test-style-guide)
- [Common Patterns](#common-patterns)
- [What to Test vs What Not to Test](#what-to-test-vs-what-not-to-test)
- [Coverage Guidelines](#coverage-guidelines)

## Testing Philosophy

ClaudeChain's testing strategy is built on three core beliefs:

1. **Test behavior, not implementation** - Tests should verify what the code does, not how it does it
2. **Mock at boundaries, not internals** - Mock external systems (APIs, subprocess, filesystem), not internal logic
3. **Value over coverage** - We prioritize meaningful tests over arbitrary coverage metrics

Our goal is **85% code coverage** with **493 tests** that provide confidence in refactoring and catch real regressions, not tests that break when internal implementations change.

### Why This Matters

ClaudeChain is a GitHub Action that orchestrates complex workflows involving:
- Git operations (branching, commits, merges)
- GitHub API interactions (PRs, comments, labels)
- File I/O (reading specs, writing artifacts)
- Subprocess execution (gh CLI, git commands)

Poor test architecture would lead to:
- ❌ Tests that break on refactoring (testing implementation details)
- ❌ Over-mocked tests that don't catch real bugs (mocking too much)
- ❌ Slow, flaky tests (relying on timing or external services)
- ❌ Unmaintainable test code (duplicating production logic in tests)

Good test architecture gives us:
- ✅ Confidence in refactoring (tests verify behavior, not structure)
- ✅ Fast, reliable test suite (493 tests run in ~5 seconds)
- ✅ Clear test failures (when a test breaks, you know what's wrong)
- ✅ Easy to write new tests (clear patterns and fixtures)

## Testing Principles

### 1. Test Isolation and Independence

**Every test should be completely independent.** Tests must not:
- Rely on execution order
- Share mutable state
- Depend on previous test results
- Affect other tests when run in parallel

**Example:**
```python
# ✅ GOOD - Each test is self-contained
class TestTaskManagement:
    def test_find_first_task(self, tmp_path):
        """Should find the first unchecked task"""
        # Arrange - Create fresh test data for this test only
        spec_file = tmp_path / "spec.md"
        spec_file.write_text("""
        - [ ] Task 1
        - [ ] Task 2
        """)

        # Act
        result = find_next_available_task(str(spec_file))

        # Assert
        assert result == (1, "Task 1")

# ❌ BAD - Tests depend on shared state
shared_state = {"count": 0}

def test_increment():
    shared_state["count"] += 1
    assert shared_state["count"] == 1  # Breaks if tests run out of order
```

### 2. Mock at System Boundaries, Not Internal Logic

**Mock external systems, not your own code.** The boundary is where your code interacts with the outside world.

**System boundaries to mock:**
- Subprocess execution (`subprocess.run`)
- HTTP requests (GitHub API)
- File I/O (reading/writing files in production paths)
- Time (`datetime.now()`)
- Environment variables

**Internal logic NOT to mock:**
- Helper functions within your module
- Domain models
- Business logic
- Data transformations

### 3. Arrange-Act-Assert Pattern

**Every test should follow AAA structure:**

1. **Arrange** - Set up test data and mocks
2. **Act** - Execute the code under test
3. **Assert** - Verify the outcome

This makes tests easy to read and understand.

### 4. One Concept Per Test

**Each test should verify ONE specific behavior.** If a test has multiple unrelated assertions, split it into separate tests.

**Why?** When a test fails, you should immediately know what broke. Multi-concept tests make debugging harder.

**Exception for E2E Tests:** End-to-end tests are expensive and slow to run (often taking minutes). For E2E tests, it's acceptable and encouraged to verify multiple related aspects of a workflow in a single test to minimize execution time. For example, an E2E test that triggers a workflow can verify:
- The workflow completes successfully
- A PR was created
- The PR has correct labels
- The PR has an AI summary
- The PR has cost information

This is a pragmatic trade-off: E2E tests prioritize execution speed over granular isolation, while unit/integration tests maintain strict one-concept-per-test discipline.

## Test Architecture Overview

### Directory Structure

Tests mirror the `src/` directory structure:

```
tests/
├── conftest.py                           # Shared fixtures
├── unit/
│   ├── domain/                           # Domain layer tests
│   │   ├── test_config.py
│   │   ├── test_models.py
│   │   └── test_exceptions.py
│   ├── infrastructure/                   # Infrastructure layer tests
│   │   ├── git/
│   │   │   └── test_operations.py
│   │   ├── github/
│   │   │   ├── test_operations.py
│   │   │   └── test_actions.py
│   │   └── filesystem/
│   │       └── test_operations.py
│   └── services/                         # Service Layer tests
│       ├── formatters/
│       │   └── test_table_formatter.py
│       ├── test_pr_operations.py
│       ├── test_task_management.py
│       ├── test_reviewer_management.py
│       └── test_statistics_service.py
├── integration/                          # Integration tests
│   └── cli/                              # CLI command integration tests
│       └── commands/
│           ├── test_prepare.py
│           ├── test_finalize.py
│           └── test_statistics.py
├── e2e/                                  # End-to-end tests
└── builders/                             # Test helpers/factories
```

### Test Layers

1. **Domain Tests** (`tests/unit/domain/`) - Test models, configuration, exceptions
2. **Infrastructure Tests** (`tests/unit/infrastructure/`) - Test external integrations (git, GitHub, filesystem)
3. **Service Layer Tests** (`tests/unit/services/`) - Test business logic in service classes
4. **CLI Integration Tests** (`tests/integration/cli/`) - Test command orchestration of service classes
5. **E2E Tests** (`tests/e2e/`) - Test complete workflows with real GitHub API

### Layer-Based Testing Strategy

ClaudeChain follows a **layered architecture**, and we test each layer differently:

```
┌─────────────────────────────────────────┐
│          CLI Layer                      │  ← Test command orchestration
│  (commands: prepare, finalize, etc.)    │     Mock everything below
├─────────────────────────────────────────┤
│       Service Layer                     │  ← Test business logic
│  (services: task mgmt, reviewer mgmt)   │     Mock infrastructure
├─────────────────────────────────────────┤
│     Infrastructure Layer                │  ← Test external integrations
│  (git, github, filesystem)              │     Mock external systems
├─────────────────────────────────────────┤
│         Domain Layer                    │  ← Test directly
│  (models, config, exceptions)           │     Minimal/no mocking
└─────────────────────────────────────────┘
```

**Key insight:** The **lower** the layer, the **less** you mock. Domain models are pure logic and need minimal mocking. CLI commands orchestrate everything and need maximum mocking.

## Testing by Layer

### Domain Layer (99% average coverage)

**Testing approach:**
- ✅ **Direct testing** - Minimal mocking
- ✅ **Test business rules** - Validation logic, data transformations
- ✅ **Test edge cases** - Invalid inputs, boundary conditions

**What to mock:** Almost nothing. Domain models are pure logic.

**Example:**
```python
class TestMarkdownFormatter:
    """Test suite for MarkdownFormatter functionality"""

    def test_bold_formatting_for_github(self):
        """Should format text as bold using GitHub markdown syntax"""
        # Arrange - No mocking needed
        formatter = MarkdownFormatter(for_slack=False)

        # Act
        result = formatter.bold("important")

        # Assert
        assert result == "**important**"
```

### Infrastructure Layer (97% average coverage)

**Testing approach:**
- ✅ **Mock external systems** - subprocess, HTTP, file I/O
- ✅ **Test success and error paths** - What happens on success? On failure?
- ✅ **Verify system calls** - Check correct commands are executed

**What to mock:** Everything external to Python (subprocess, network, filesystem)

### Service Layer (95% average coverage)

**Testing approach:**
- ✅ **Mock infrastructure** - Mock git, GitHub API, filesystem
- ✅ **Test business logic** - Reviewer selection, task finding algorithms
- ✅ **Test service instantiation** - Verify services are created with correct dependencies
- ✅ **Test edge cases** - No reviewers available, all tasks complete, etc.

**What to mock:** Infrastructure layer (subprocess, GitHub API, file I/O) and service dependencies

### CLI Integration Tests (98% average coverage)

**Testing approach:**
- ✅ **Mock service layer** - Mock service classes and their methods
- ✅ **Mock infrastructure** - Mock git, GitHub API, filesystem operations
- ✅ **Test command orchestration** - Verify correct sequence of service calls
- ✅ **Test service instantiation** - Verify services are created with correct dependencies
- ✅ **Test output** - Verify GitHub Actions outputs are written correctly

**What to mock:** Service Layer classes and Infrastructure dependencies

## Quick Start

### Running Tests Locally

```bash
# Run all unit tests
PYTHONPATH=src:scripts pytest tests/unit/ -v

# Run all integration tests
PYTHONPATH=src:scripts pytest tests/integration/ -v

# Run both unit and integration tests
PYTHONPATH=src:scripts pytest tests/unit/ tests/integration/ -v

# Run with coverage report
PYTHONPATH=src:scripts pytest tests/unit/ tests/integration/ --cov=src/claudechain --cov-report=term-missing --cov-report=html

# Run tests for a specific module
PYTHONPATH=src:scripts pytest tests/integration/cli/commands/test_prepare.py -v

# Run a specific test
PYTHONPATH=src:scripts pytest tests/integration/cli/commands/test_prepare.py::TestCmdPrepare::test_successful_preparation -v
```

### Understanding Test Results

```bash
# View coverage in terminal
PYTHONPATH=src:scripts pytest tests/unit/ tests/integration/ --cov=src/claudechain --cov-report=term-missing

# View detailed HTML coverage report (opens in browser)
PYTHONPATH=src:scripts pytest tests/unit/ tests/integration/ --cov=src/claudechain --cov-report=html
open htmlcov/index.html  # macOS
xdg-open htmlcov/index.html  # Linux
```

## Test Style Guide

**IMPORTANT**: All tests in this project MUST follow these conventions. Consistency is critical for maintainability.

### Required Test Structure

```python
class TestFeatureName:
    """Test suite for FeatureName functionality"""

    @pytest.fixture
    def fixture_name(self):
        """Fixture providing test data - clear docstring explaining purpose"""
        return setup_test_data()

    def test_feature_does_something_when_condition(self, fixture_name):
        """Test description explaining what this verifies

        Should read like: "Test that feature does X when Y condition exists"
        """
        # Arrange - Set up test data and mocks
        expected_value = "expected"
        mock_dependency = Mock()

        # Act - Execute the code under test
        result = function_under_test(fixture_name, mock_dependency)

        # Assert - Verify the outcome
        assert result == expected_value
        mock_dependency.method.assert_called_once()
```

### Naming Conventions

**Class Names**: `TestModuleName` or `TestFunctionName`
- ✅ `TestReviewerManagement`
- ✅ `TestCheckReviewerCapacity`
- ❌ `ReviewerTests` (wrong suffix)
- ❌ `TestReviewers` (too vague)

**Test Method Names**: `test_<what>_<when>_<condition>`
- ✅ `test_find_reviewer_returns_none_when_all_at_capacity`
- ✅ `test_create_branch_raises_error_when_branch_exists`
- ✅ `test_parse_branch_extracts_project_and_index`
- ❌ `test_reviewer` (not descriptive)
- ❌ `test_find_reviewer_works` (vague)

**Fixture Names**: `<resource>_<state>` or `mock_<service>`
- ✅ `reviewer_config`, `sample_spec_file`, `mock_github_api`
- ❌ `data`, `setup`, `fixture1`

## Common Patterns

### Using Fixtures from conftest.py

Fixtures in `conftest.py` are automatically available to all tests:

```python
# tests/conftest.py defines:
@pytest.fixture
def sample_spec_file(tmp_path):
    """Fixture providing a sample spec.md file"""
    spec_content = """# Project

    - [x] Task 1
    - [ ] Task 2
    """
    spec_file = tmp_path / "spec.md"
    spec_file.write_text(spec_content)
    return spec_file

# Your test can use it:
def test_find_task(sample_spec_file):
    """No import needed - pytest finds it automatically"""
    result = find_next_available_task(str(sample_spec_file))
    assert result == (2, "Task 2")
```

**Common fixtures:**
- `tmp_path` - Built-in pytest fixture for temporary directories
- `sample_spec_file` - Pre-built spec.md with tasks
- `sample_config_dict` - Pre-built configuration
- `mock_subprocess` - Mocked subprocess module
- `mock_github_api` - Mocked GitHub API client

### Parametrized Tests for Boundary Conditions

Use `@pytest.mark.parametrize` to test multiple cases with the same logic:

```python
@pytest.mark.parametrize("open_prs,max_prs,expected", [
    (0, 2, True),   # Well under capacity
    (1, 2, True),   # Just under capacity
    (2, 2, False),  # Exactly at capacity
    (3, 2, False),  # Over capacity
    (0, 0, False),  # Edge case: zero max
])
def test_check_capacity_boundaries(open_prs, max_prs, expected, mock_github_api):
    """Should correctly handle all capacity boundary conditions"""
    # Arrange
    reviewer = ReviewerConfig(username="alice", maxOpenPRs=max_prs)
    mock_github_api.get_open_prs.return_value = [
        {"number": i} for i in range(open_prs)
    ]

    # Act
    result = check_reviewer_capacity(reviewer, mock_github_api)

    # Assert
    assert result == expected
```

### Error Handling and Edge Cases

Always test both success and failure paths:

```python
class TestLoadConfig:
    def test_load_valid_config(self, sample_config_file):
        """Should load and parse valid configuration"""
        # Test success path
        config = load_config(str(sample_config_file))
        assert len(config.reviewers) == 3

    def test_load_missing_file(self):
        """Should raise FileNotFoundError when config missing"""
        # Test error path
        with pytest.raises(FileNotFoundError):
            load_config("/nonexistent/path/config.yml")

    def test_load_invalid_yaml(self, tmp_path):
        """Should raise ValidationError for invalid YAML"""
        # Test validation error path
        bad_config = tmp_path / "bad.yml"
        bad_config.write_text("invalid: yaml: content:")

        with pytest.raises(ValidationError):
            load_config(str(bad_config))
```

## What to Test vs What Not to Test

### What to Test ✅

1. **Business logic**
   - Task finding algorithms
   - Reviewer selection logic
   - Task completion marking
   - Project detection from branch names

2. **Edge cases and boundaries**
   - Empty inputs (no tasks, no reviewers)
   - Boundary conditions (exactly at capacity)
   - Invalid inputs (malformed configs, missing files)

3. **Error handling**
   - File not found
   - API failures
   - Invalid configuration
   - Git operation failures

4. **Integration points**
   - Correct subprocess commands
   - Correct API calls
   - Correct file operations

5. **Data transformations**
   - Parsing spec files
   - Formatting output
   - Generating task IDs

### What NOT to Test ❌

1. **Language/framework features**
   ```python
   # ❌ Don't test Python itself
   def test_list_append():
       my_list = []
       my_list.append(1)
       assert len(my_list) == 1  # Python works, don't test it
   ```

2. **Third-party libraries**
   ```python
   # ❌ Don't test pytest or PyYAML
   def test_yaml_loads_correctly():
       result = yaml.safe_load("key: value")
       assert result == {"key": "value"}  # PyYAML works, don't test it
   ```

3. **Trivial getters/setters**
   ```python
   # ❌ Don't test simple properties
   def test_reviewer_username_getter():
       reviewer = ReviewerConfig(username="alice", maxOpenPRs=2)
       assert reviewer.username == "alice"  # No logic to test
   ```

4. **Implementation details**
   ```python
   # ❌ Don't test internal helper calls
   def test_prepare_calls_internal_helper():
       with patch('module._internal_helper') as mock:
           cmd_prepare()
           assert mock.called  # Breaks if refactored
   ```

## Coverage Guidelines

### Current Coverage Status

As of December 2025:
- **Overall coverage: 85.03%** (exceeding 70% minimum threshold)
- **493 tests passing** (0 failures)
- **CI enforces minimum 70% coverage**

### What to Test

**High Priority (90%+ coverage):**
- Business logic (task management, reviewer assignment)
- Command orchestration (prepare, finalize, discover)
- Configuration validation
- Error handling paths

**Medium Priority (70%+ coverage):**
- Utilities and formatters
- GitHub API integrations
- Artifact operations

**Low Priority (may skip):**
- Simple getters/setters
- Third-party library wrappers
- CLI argument parsing (tested via E2E)
- Entry points (`__main__.py` - tested via E2E)

### Viewing Coverage Reports

```bash
# Generate HTML coverage report
PYTHONPATH=src:scripts pytest tests/unit/ tests/integration/ --cov=src/claudechain --cov-report=html

# Open in browser
open htmlcov/index.html  # macOS
xdg-open htmlcov/index.html  # Linux

# View in terminal with missing lines
PYTHONPATH=src:scripts pytest tests/unit/ tests/integration/ --cov=src/claudechain --cov-report=term-missing
```

Coverage reports are also available in GitHub Actions artifacts for every CI run.

## CI/CD Integration

### Automated Testing

Tests run automatically on:
- Every push to main branch
- Every pull request
- Via `.github/workflows/test.yml`

### Test Workflow

```yaml
# .github/workflows/test.yml
- name: Run unit and integration tests with coverage
  run: |
    PYTHONPATH=src:scripts pytest tests/unit/ tests/integration/ \
      --cov=src/claudechain \
      --cov-report=term-missing \
      --cov-report=html \
      --cov-fail-under=70
```

### PR Requirements

- All tests must pass
- Coverage must meet 70% minimum threshold
- New features require tests
- Bug fixes should include regression tests

## Best Practices Summary

### Quick Checklist

Before committing a test, verify:

- [ ] Test name clearly describes what's being tested
- [ ] Test has a docstring explaining its purpose
- [ ] Test follows Arrange-Act-Assert structure
- [ ] Test has meaningful assertions (not just "doesn't crash")
- [ ] Test mocks external dependencies, not internal logic
- [ ] Test is focused (one concept/behavior)
- [ ] Test will fail if the code is broken
- [ ] Test won't break if internal implementation changes
- [ ] Test doesn't duplicate existing test coverage
- [ ] Test adds value (not testing language/framework/library)

### When to Mock

| Component | Mock? | Why |
|-----------|-------|-----|
| subprocess.run | ✅ Yes | External system |
| GitHub API calls | ✅ Yes | External system |
| File I/O (production paths) | ✅ Yes | External system |
| Your business logic | ❌ No | Internal logic |
| Domain models | ❌ No | Pure logic |
| Helper functions | ❌ No | Internal logic |

### Coverage Guidelines

| Priority | Target Coverage | Examples |
|----------|----------------|----------|
| High | 90%+ | Business logic, commands |
| Medium | 70%+ | Utilities, formatters |
| Low | Skip | Getters, entry points (E2E tested) |

**Current overall coverage: 85.03%** (exceeding 70% minimum)

## Local Testing

### Unit Testing

Unit tests are located in `tests/unit/` and provide comprehensive test coverage:

```bash
# Set Python path
export PYTHONPATH=src:scripts

# Run all unit tests
pytest tests/unit/ -v

# Run with coverage
pytest tests/unit/ --cov=src --cov-report=html

# View coverage report
coverage report --show-missing
open htmlcov/index.html  # macOS
```

Current coverage: 85%+ with 506 tests

### End-to-End Testing

End-to-end integration tests are now located in this repository at `tests/e2e/` and use a **recursive workflow pattern** where ClaudeChain tests itself.

**Purpose:**
The E2E tests validate the complete ClaudeChain workflow by:
- Creating test projects in the same repository (`claude-chain/test-*`)
- Triggering the `claudechain-test.yml` workflow which runs the action on itself
- Verifying PRs are created correctly with AI-generated summaries
- Testing reviewer capacity limits
- Testing merge trigger functionality
- Cleaning up all test resources automatically

**Running E2E Tests Locally:**
```bash
# From the claude-chain repository root
cd /path/to/claude-chain
./tests/e2e/run_test.sh
```

**Prerequisites:**
- GitHub CLI (`gh`) installed and authenticated
- Python 3.11+ with pytest (`pip install pytest pyyaml`)
- Git configured with user.name and user.email
- Repository write access
- `ANTHROPIC_API_KEY` (optional, tests will prompt)

For comprehensive E2E testing documentation, see the [E2E Testing Guide](../../tests/e2e/README.md).

## Resources

- [pytest documentation](https://docs.pytest.org/)
- [pytest-cov documentation](https://pytest-cov.readthedocs.io/)
- [Python unittest.mock guide](https://docs.python.org/3/library/unittest.mock.html)
- [Testing Best Practices (Python Guide)](https://docs.python-guide.org/writing/tests/)

## Getting Help

If you have questions about testing:
1. Review existing tests in `tests/unit/` and `tests/integration/` for examples
2. Check this document for detailed patterns
3. Ask in code review for guidance on testing approach
4. Run tests locally before pushing to catch issues early
