# Service Layer Skills Plugin

A Claude Code plugin providing general-purpose skills for developing Python applications following Martin Fowler's Service Layer pattern. This plugin helps Claude understand and apply architectural best practices for service-oriented codebases.

## Overview

This plugin provides three complementary skills that guide Claude when working with Python projects using the Service Layer pattern:

- **creating-services** - Guides Claude in creating well-designed service classes with proper dependency injection and layering
- **testing-services** - Helps Claude write effective tests following the Arrange-Act-Assert pattern with proper mocking at boundaries
- **identifying-layer-placement** - Assists Claude in determining the correct architectural layer for new code

These skills work together to ensure your codebase maintains clean architecture, testability, and separation of concerns.

## Installation

### Via Marketplace (Recommended)

1. Add the marketplace to Claude Code:
   ```
   /plugin marketplace add gestrich/service-layer-skills
   ```

2. Install the plugin:
   ```
   /plugin install service-layer-skills@gestrich-service-layer-skills
   ```

3. Restart Claude Code if necessary

### Local Testing

For local development or testing:
```bash
claude --plugin-dir ~/path/to/service-layer-skills
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

## Usage

These skills are **automatically invoked** by Claude based on the context of your requests. You don't need to explicitly call them - Claude will recognize when architectural guidance is needed and apply the relevant skill.

The skills work best when:
- You provide clear descriptions of what you're trying to accomplish
- Your project follows (or aims to follow) the Service Layer pattern
- You're asking for guidance on design decisions, not just quick fixes

## Requirements

- Python project (any version)
- Codebase following or transitioning to Martin Fowler's Service Layer pattern
- Claude Code installed

The skills are framework-agnostic and work with any Python application using service-oriented architecture.

## Use Cases

This plugin is especially helpful when:

- **Starting a new project** - Establish good architectural patterns from the beginning
- **Refactoring legacy code** - Migrate toward cleaner layered architecture
- **Onboarding new developers** - Ensure consistent patterns across the team
- **Code reviews** - Claude can apply these patterns when suggesting improvements
- **Adding features** - Maintain architectural consistency as the codebase grows

## Further Reading

To learn more about the Service Layer pattern that these skills are based on:

- [Martin Fowler's Service Layer Pattern](https://martinfowler.com/eaaCatalog/serviceLayer.html)
- [Patterns of Enterprise Application Architecture](https://martinfowler.com/books/eaa.html)

## Contributing

Found an issue or have a suggestion? Please open an issue on the [GitHub repository](https://github.com/gestrich/service-layer-skills).

Contributions are welcome! Feel free to submit pull requests for:
- Improvements to skill content
- Additional examples
- Documentation enhancements
- Bug fixes

## License

This plugin is licensed under the MIT License. See [LICENSE](LICENSE) file for details.

## Acknowledgments

These skills are based on architectural patterns from Martin Fowler's work on enterprise application architecture, adapted for modern Python development.
