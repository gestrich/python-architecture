# Python Architecture Plugin

A Claude Code plugin providing architectural best practices for Python applications. This plugin helps Claude understand and apply clean architecture patterns, including the Service Layer pattern, proper testing practices, and correct layer placement.

## Overview

This plugin provides seven specialized skills that guide Claude when working with Python projects using the Service Layer pattern. These skills cover service creation, testing, architectural layer placement, domain modeling, CLI structure, dependency injection, and code style conventions.

- **creating-services** - Guides Claude in creating well-designed service classes with proper dependency injection and layering
- **testing-services** - Helps Claude write effective tests following the Arrange-Act-Assert pattern with proper mocking at boundaries
- **identifying-layer-placement** - Assists Claude in determining the correct architectural layer for new code
- **domain-modeling** - Design rich domain models with parse-once principle, factory methods, and type-safe APIs following Repository pattern
- **cli-architecture** - Structure CLI applications using command dispatcher pattern with proper service instantiation and explicit parameter flow
- **dependency-injection** - Apply constructor-based dependency injection with fail-fast principles and configuration flowing from entry points
- **python-code-style** - Follow Python code organization conventions including method ordering, timezone-aware datetimes, and type annotations

These skills work together to ensure your codebase maintains clean architecture, testability, and separation of concerns.

## Installation

### Via Marketplace (Recommended)

1. Add the marketplace to Claude Code:
   ```
   /plugin marketplace add gestrich/python-architecture
   ```

2. Install the plugin:
   ```
   claude plugin install python-architecture@gestrich-python-architecture --scope project
   ```
   > **Note:** Use `--scope user` instead of `--scope project` to install system-wide for all projects.

3. Restart Claude Code if necessary

### Local Testing

For local development or testing:
```bash
claude --plugin-dir ~/path/to/python-architecture
```

## Skills Included

### 1. creating-services

**When activated:** Claude automatically uses this skill when you ask it to create new service classes or refactor code into services.

**What it provides:**
- Constructor-based dependency injection patterns
- Proper separation between Core and Composite services
- Static vs instance method guidelines
- Service initialization patterns
- Best practices for fail-fast design

**Example triggers:**
- "Create a new EmailService that sends notifications"
- "Refactor this code into a service class"
- "Add a new service for handling user authentication"

### 2. testing-services

**When activated:** Claude automatically uses this skill when you ask it to write tests for service classes or test business logic.

**What it provides:**
- Arrange-Act-Assert (AAA) test structure
- Mock-at-boundaries philosophy (mock external systems, not internal logic)
- Test organization and naming conventions
- Common anti-patterns to avoid
- Integration vs unit testing guidance

**Example triggers:**
- "Write tests for UserService.authenticate_user()"
- "Add test coverage for the PaymentService"
- "How should I test this service method?"

### 3. identifying-layer-placement

**When activated:** Claude automatically uses this skill when you ask where to place new functionality or need architectural guidance.

**What it provides:**
- Clear decision tree for layer placement
- Definitions of CLI, Service, Infrastructure, and Domain layers
- Guidelines for cross-layer dependencies
- Common patterns for each layer

**Example triggers:**
- "Where should I put JSON formatting code?"
- "Which layer should handle database queries?"
- "I need to add validation logic - where does it go?"

### 4. domain-modeling

**When activated:** Claude automatically uses this skill when you're creating domain models, parsing data structures, or designing data classes.

**What it provides:**
- Parse-once principle with type-safe APIs
- Factory method patterns for creating domain models
- Repository pattern for data access
- Type-safe domain model design
- Validation at model boundaries

**Example triggers:**
- "Create a User model that parses from YAML"
- "I need to parse this JSON into domain objects"
- "Design a domain model for project configuration"
- "Add validation to this data class"

### 5. cli-architecture

**When activated:** Claude automatically uses this skill when you're building CLI tools or structuring command-line applications.

**What it provides:**
- Command dispatcher pattern with single entry point
- Service instantiation in CLI commands
- Explicit parameter flow (commands don't read env vars)
- Layered directory structure for CLI apps
- Three-section service instantiation pattern

**Example triggers:**
- "Build a multi-command CLI tool"
- "Structure my CLI application"
- "Add a new command to the dispatcher"
- "How should commands instantiate services?"

### 6. dependency-injection

**When activated:** Claude automatically uses this skill when you're designing service constructors, managing dependencies, or dealing with configuration.

**What it provides:**
- Constructor-based dependency injection
- Required vs optional dependency patterns
- Fail-fast exception handling
- Configuration flow from entry points
- Avoiding services that read environment variables

**Example triggers:**
- "My service constructor has too many optional parameters"
- "Services are reading environment variables directly"
- "How should I handle missing dependencies?"
- "Where should configuration defaults go?"

### 7. python-code-style

**When activated:** Claude automatically uses this skill when you're organizing code, handling dates and times, or dealing with circular imports.

**What it provides:**
- Method organization (public before private, standard ordering)
- Section headers for complex services
- Timezone-aware datetime handling with UTC
- Circular import avoidance strategies
- Modern type annotations (Self, from __future__ import annotations)

**Example triggers:**
- "Organize the methods in this service class"
- "Tests failing due to naive datetimes"
- "How do I fix this circular import?"
- "Add type hints to this factory method"

## Usage

These skills are **automatically invoked** by Claude based on the context of your requests. You don't need to explicitly call them - Claude will recognize when architectural guidance is needed and apply the relevant skill.

The skills work best when:
- You provide clear descriptions of what you're trying to accomplish
- Your project follows (or aims to follow) the Service Layer pattern
- You're asking for guidance on design decisions, not just quick fixes

## Troubleshooting

### Slash commands not showing in terminal

The slash commands (e.g., `/creating-services`) may not appear in the terminal autocomplete but will show up in the Claude Code VSCode extension. Use the VSCode extension for the best experience with slash command discovery.

### Updating the plugin

If you need to update the plugin to a newer version, you may need to remove and re-add it:

```
/plugin uninstall python-architecture@gestrich-python-architecture
/plugin install python-architecture@gestrich-python-architecture
```

### Plugin not available after adding marketplace

Sometimes the plugin won't appear as an installation option even after adding the marketplace. You can work around this by manually enabling the plugin in your Claude Code configuration.

Create or edit `~/.claude/config.json` and add:

```json
{
  "enabledPlugins": {
    "python-architecture@gestrich-python-architecture": true
  }
}
```

Then restart Claude Code.

## Requirements

- Python project (any version)
- Codebase following or transitioning to Martin Fowler's Service Layer pattern
- Claude Code installed

The skills are framework-agnostic and work with any Python application using service-oriented architecture.

## Use Cases

This plugin is especially helpful when:

- **Starting a new project** - Establish good architectural patterns from the beginning with proper service design, domain modeling, and CLI structure
- **Refactoring legacy code** - Migrate toward cleaner layered architecture with explicit dependency injection and proper layer separation
- **Onboarding new developers** - Ensure consistent patterns across the team for code organization, testing, and architectural decisions
- **Code reviews** - Claude can apply these patterns when suggesting improvements to service constructors, domain models, or test structure
- **Adding features** - Maintain architectural consistency as the codebase grows by following established patterns for each layer
- **Building CLI tools** - Structure command-line applications with proper dispatcher patterns and service instantiation
- **Parsing external data** - Create robust domain models with type-safe parsing and validation at boundaries
- **Fixing datetime bugs** - Apply timezone-aware datetime handling to prevent naive datetime issues
- **Resolving circular imports** - Restructure module dependencies following one-way dependency graphs

## Further Reading

To learn more about the Service Layer pattern that these skills are based on:

- [Martin Fowler's Service Layer Pattern](https://martinfowler.com/eaaCatalog/serviceLayer.html)
- [Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html)

## Contributing

Found an issue or have a suggestion? Please open an issue on the [GitHub repository](https://github.com/gestrich/python-architecture).

Contributions are welcome! Feel free to submit pull requests for:
- Improvements to skill content
- Additional examples
- Documentation enhancements
- Bug fixes

## License

This plugin is licensed under the MIT License. See [LICENSE](LICENSE) file for details.

## Acknowledgments

These skills are based on architectural patterns from Martin Fowler's work on enterprise application architecture, adapted for modern Python development.
