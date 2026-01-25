## Background

This plan integrates four comprehensive Python architecture documentation files (python-style.md, domain-model-design.md, command-dispatcher.md, testing-philosophy.md) from the claude-chain project into the python-architecture plugin by creating focused skills.

**Current state:**
- Plugin has 3 existing skills: creating-services, testing-services, identifying-layer-placement
- 4 architecture docs copied to repository root with detailed Python patterns

**User requirements:**
- Create multiple new skills (4-5 new) with narrow focus
- Extract distinct topics into separate skills for maximum discoverability
- Maintain consistency with existing skill structure (YAML frontmatter, code examples with ✅/❌ markers)
- Add cross-references between related skills

**Outcome:**
7 total skills (3 existing + 4 new) covering: service creation, testing, layer placement, domain modeling, CLI architecture, dependency injection, and Python code style.

## Phases

- [x] Phase 1: Create domain-modeling skill ✓ **COMPLETED**

**Technical notes:**
- Created `plugin/skills/domain-modeling/SKILL.md` with all required content
- Extracted and synthesized content from `domain-model-design.md` and `python-style.md` (lines 856-1182)
- Included YAML frontmatter with user-invocable: true
- Added comprehensive sections: Quick Reference, Anti-Pattern, Recommended Pattern, Benefits, Architecture Pattern, Key Principles, Related Patterns, Related Skills, Further Reading
- All code examples include ✅/❌ markers consistent with existing skills
- Added cross-references to creating-services, testing-services, and identifying-layer-placement skills
- File structure verified: 4 total skills now available

Create `plugin/skills/domain-modeling/SKILL.md` extracting content from domain-model-design.md and python-style.md (lines 856-1182):

**YAML frontmatter:**
```yaml
---
name: domain-modeling
description: Design domain models following parse-once principle with type-safe APIs, factory methods, and Repository pattern. Use when creating domain models, parsing data structures, or organizing business logic in models.
user-invocable: true
argument-hint: "[model-name]"
---
```

**Content sections:**
- Quick reference with parse-once pattern example
- Principle: Parse Once Into Well-Formed Models
- Anti-pattern: Services parsing strings directly (lines 8-42 from domain-model-design.md)
- Recommended pattern: Domain models with factory methods (lines 44-210)
- Architecture pattern diagram showing External System → Infrastructure → Domain → Service layers
- Benefits: type safety, centralized parsing, testability, validation at boundaries, reusability
- Key principles: parse at boundaries, parse once, domain owns parsing, type-safe APIs
- Related patterns: Repository, Factory, Domain-Driven Design, Dependency Inversion
- Code examples: Project, ProjectConfiguration, SpecContent models with @classmethod factories

**Expected outcome:** Comprehensive skill teaching developers to create rich domain models that encapsulate parsing logic and provide type-safe interfaces.

- [x] Phase 2: Create cli-architecture skill ✓ **COMPLETED**

**Technical notes:**
- Created `plugin/skills/cli-architecture/SKILL.md` with comprehensive CLI architecture guidance
- Extracted and synthesized content from `command-dispatcher.md` (full file) and CLI patterns from `python-style.md`
- Included YAML frontmatter with user-invocable: true and argument-hint: "[command-name]"
- Added comprehensive sections: When to Use, Quick Reference, Architecture Pattern, Command Structure, Module Organization, CLI Command Pattern (Explicit Parameters), Service Instantiation Pattern, Class-Based Services, Design Principles, Related Patterns, Related Skills, Further Reading
- All code examples include ✅/❌ markers consistent with existing skills
- Covered: single dispatcher pattern, command routing, layered directory structure, explicit parameter flow (anti-pattern: commands reading env vars), three-section service instantiation pattern, static vs instance methods
- Added cross-references to creating-services, dependency-injection, identifying-layer-placement, and domain-modeling skills
- File structure verified: 5 total skills now available (3 existing + 2 new)

Create `plugin/skills/cli-architecture/SKILL.md` extracting content from command-dispatcher.md and python-style.md CLI sections:

**YAML frontmatter:**
```yaml
---
name: cli-architecture
description: Structure CLI applications using command dispatcher pattern with layered architecture. Covers entry points, command routing, service instantiation, and explicit parameter flow. Use when building CLI tools or refactoring command structure.
user-invocable: true
argument-hint: "[command-name]"
---
```

**Content sections:**
- Quick reference showing dispatcher pattern (`python -m package <command>`)
- Entry point pattern using `__main__.py` as single dispatcher
- Available commands table (command, description, used by)
- Command structure: Dispatcher routes to command functions
- Command implementation pattern: get config → initialize infrastructure → initialize services → use services → format output → return exit code
- Module organization showing layered directory structure (domain/, infrastructure/, services/, cli/)
- CLI command pattern: commands use explicit parameters, NOT environment variables
- Anti-patterns: commands reading env vars, commands containing business logic, mixing args and env
- Recommended pattern: `__main__.py` adapter layer reads env vars, passes explicit params to commands
- Service instantiation pattern: three-section pattern (get deps, init infrastructure, init services)
- Benefits: single entry point, consistent interface, shared utilities, easy extension, local testing

**Expected outcome:** Clear guidance on structuring CLI applications with proper separation between orchestration (commands) and business logic (services).

- [x] Phase 3: Create dependency-injection skill ✓ **COMPLETED**

**Technical notes:**
- Created `plugin/skills/dependency-injection/SKILL.md` with comprehensive DI guidance
- Extracted and synthesized content from `python-style.md` (lines 95-723)
- Included YAML frontmatter with user-invocable: true and argument-hint: ""
- Added comprehensive sections: When to Use, Quick Reference, Principle sections for avoiding optional dependencies, fail-fast patterns, configuration flow, and environment variables
- All code examples include ✅/❌ markers consistent with existing skills
- Covered: required vs optional dependencies, fail-fast exception handling, custom exception classes, configuration flowing from entry points, services not reading environment variables, constructor vs method parameters
- Added cross-references to creating-services, cli-architecture, testing-services, and identifying-layer-placement skills
- File structure verified: 6 total skills now available (3 existing + 3 new)

Create `plugin/skills/dependency-injection/SKILL.md` extracting DI patterns from python-style.md (lines 95-723):

**YAML frontmatter:**
```yaml
---
name: dependency-injection
description: Apply constructor-based dependency injection with explicit configuration flow. Covers required dependencies, fail-fast principles, and avoiding optional parameters with default factories. Use when designing service constructors or managing dependencies.
user-invocable: true
argument-hint: ""
---
```

**Content sections:**
- Quick reference with constructor pattern showing explicit dependencies
- Anti-pattern: Optional dependencies with default factories (lines 95-125)
  - Shows `param: Optional[Type] = None` with `self.param = param or DefaultType()` as code smell
  - Problems: hidden dependencies, configuration ambiguity, testing confusion, dangerous defaults
- Recommended pattern: Required dependencies via constructor (lines 127-178)
  - All dependencies as required parameters, no defaults, explicit at call site
  - Benefits: explicit dependencies, no hidden behavior, clear ownership, better testing, fail fast
- When optional parameters ARE acceptable: truly optional behavior, optional filters, backward compatibility
- Fail-fast principle: Avoid silent failures (lines 221-411)
  - Treat abnormal cases as errors, not empty data
  - Anti-pattern: returning empty data structures when something is wrong
  - Recommended: raise exceptions with clear error messages
  - Custom exception classes for domain errors
- Configuration defaults and flow (lines 413-574)
  - Configuration flows from entry point (CLI, web handler) down through layers
  - Defaults only at entry point, never in middle of call stack
  - Anti-pattern: default configuration deep in call stack
  - Recommended: constructor parameters or explicit method parameters
- Services should NOT read environment variables (lines 576-723)
  - Service classes never use `os.environ.get()`
  - All env var access at entry point layer
  - Exception: Infrastructure layer for connecting to external systems (with preference for explicit params)

**Expected outcome:** Developers understand how to design services with explicit, testable dependencies and configuration that flows from the top down.

- [x] Phase 4: Create python-code-style skill ✓ **COMPLETED**

**Technical notes:**
- Created `plugin/skills/python-code-style/SKILL.md` with comprehensive Python code style guidance
- Extracted and synthesized content from `python-style.md` (lines 1-94 for method organization, 1183-1393 for datetime handling, 1395-1431 for circular imports, 1432-1477 for type annotations)
- Included YAML frontmatter with user-invocable: true and argument-hint: "[topic]"
- Added comprehensive sections: When to Use, Quick Reference, Method Organization Principles (Public Before Private, Standard Method Order), Section Headers for Complex Services, Module-Level Code Organization, Timezone-Aware Datetimes, Circular Import Avoidance, Modern Type Annotations
- All code examples include ✅/❌ markers consistent with existing skills
- Covered: public before private, high-level before low-level, standard method order (special → class → static → instance → private), section headers, module organization, timezone-aware datetimes with UTC, parse_iso_timestamp helper, domain model validation, circular import fixes, Self type for factory methods, from __future__ import annotations
- Added cross-references to creating-services, testing-services, domain-modeling skills
- File structure verified: 7 total skills now available (3 existing + 4 new)

Create `plugin/skills/python-code-style/SKILL.md` extracting Python style guidance from python-style.md:

**YAML frontmatter:**
```yaml
---
name: python-code-style
description: Follow Python code organization conventions including method ordering, datetime handling, circular import avoidance, and type annotations. Use when organizing service classes, handling dates, or structuring modules.
user-invocable: true
argument-hint: "[topic]"
---
```

**Content sections:**
- Quick reference with method organization hierarchy
- Method organization principles (lines 1-94):
  - Public before private
  - High-level before low-level
  - Logical grouping
  - Standard order: special methods → class methods → static methods → instance methods → private
- Section headers with examples (simple separators vs descriptive headers with ===)
- Module-level code organization for function-based modules:
  - Dataclasses/models → Public API functions → Module utilities → Private helpers
- Datetime and timezone handling (lines 1183-1393):
  - Always use timezone-aware datetimes (never naive)
  - Use UTC for all internal operations
  - Anti-pattern: `datetime.now()` without timezone, deprecated `datetime.utcnow()`
  - Recommended: `datetime.now(timezone.utc)`, `parse_iso_timestamp()` helper
  - Domain model validation with `__post_init__` checks
  - ISO 8601 serialization format
- Avoiding circular imports (lines 1395-1431):
  - Design one-way dependency graphs, don't use TYPE_CHECKING workarounds
  - Anti-pattern: TYPE_CHECKING guards, using Any to avoid circular imports
  - Fix: Identify foundational (lower-level) modules, ensure they never import from higher-level
- Type annotations (lines 1432-1477):
  - Use `Self` for factory methods returning instance of same class
  - Use `from __future__ import annotations` for self-referential types
  - Anti-pattern: Quoted string annotations for forward references

**Expected outcome:** Consistent code organization across Python projects with proper datetime handling, clean imports, and modern type annotations.

- [x] Phase 5: Enhance testing-services skill ✓ **COMPLETED**

**Technical notes:**
- Enhanced `plugin/skills/testing-services/SKILL.md` with comprehensive testing guidance from testing-philosophy.md
- All existing content preserved and enhanced with new sections
- Added Test Isolation and Independence section after Quick Reference with self-contained test examples
- Enhanced One Concept Per Test section with E2E exception explaining pragmatic trade-offs for slow tests
- Added comprehensive Layer-Based Testing Strategy section with architecture diagram showing CLI → Service → Infrastructure → Domain layers
- Included testing approach and coverage targets for each layer (Domain 99%, Infrastructure 97%, Service 95%, CLI 98%)
- Added Coverage Guidelines section with priority-based testing strategy and coverage report commands
- Added Quick Start section near beginning with local test execution examples and coverage viewing
- Enhanced Test Organization section with complete directory structure mirroring src/, comprehensive naming conventions
- Added CI/CD Integration section with GitHub Actions workflow example and PR requirements
- Added cross-references to python-code-style skill (test organization), domain-modeling skill (what not to mock)
- Added Related Skills section referencing creating-services, domain-modeling, python-code-style, identifying-layer-placement
- File structure verified: testing-services skill now contains comprehensive testing philosophy integrated with existing patterns

Update `plugin/skills/testing-services/SKILL.md` by adding content from testing-philosophy.md while preserving existing content:

**New sections to add:**

1. **Test Isolation and Independence** (after Quick Reference, before Mock at Boundaries):
   - Every test completely independent
   - Must not rely on execution order, share mutable state, depend on previous results, affect other tests
   - Example showing good (self-contained with tmp_path) vs bad (shared state)

2. **One Concept Per Test** (after AAA Pattern section):
   - Each test verifies ONE specific behavior
   - Exception for E2E tests: pragmatic trade-off to verify multiple aspects in slow tests
   - Examples of wrong (multiple unrelated assertions) vs right (separate focused tests)

3. **Layer-Based Testing Strategy** (new major section before "Key Testing Patterns"):
   - Architecture diagram showing CLI → Service → Infrastructure → Domain layers
   - Key insight: lower the layer, less you mock
   - Testing by layer:
     - Domain Layer (99% coverage): minimal mocking, test business rules directly
     - Infrastructure Layer (97% coverage): mock external systems, test success/error paths
     - Service Layer (95% coverage): mock infrastructure, test business logic
     - CLI Integration Tests (98% coverage): mock services and infrastructure, test orchestration

4. **Coverage Guidelines** (new section before "Common Anti-Patterns"):
   - Current status: 85%+ overall coverage, 493 tests
   - High priority (90%+): business logic, commands, configuration
   - Medium priority (70%+): utilities, formatters, API integrations
   - Low priority: simple getters, third-party wrappers, CLI parsing (E2E tested)
   - Viewing coverage reports with pytest commands

5. **Quick Start** (near beginning, after When to Use):
   - Running tests locally with PYTHONPATH commands
   - Understanding test results and coverage reports
   - Examples: run all, run specific file, run with coverage

6. **Test Organization** (replace existing brief mention):
   - Directory structure showing tests/unit/, tests/integration/, tests/e2e/, tests/builders/
   - Naming conventions with examples

7. **CI/CD Integration** (new section at end):
   - Automated testing on push/PR
   - Test workflow YAML example
   - PR requirements: all tests pass, 70% minimum coverage

**Integration approach:**
- Keep all existing content (AAA pattern, mock rules, templates, fixtures, parametrized tests)
- Insert new sections logically between existing sections
- Add cross-references to new skills (python-code-style for test organization, domain-modeling for what to mock)
- Maintain consistent structure and formatting with ✅/❌ examples

**Expected outcome:** Enhanced testing skill with comprehensive coverage of test architecture, layer-based strategies, and practical coverage guidance.

- [ ] Phase 6: Add cross-references between skills

Update existing skills to reference new skills:

**creating-services skill updates:**
- In "Constructor Pattern" section: Add note referencing dependency-injection skill for detailed DI patterns
- In "Service Templates" section: Add note referencing domain-modeling skill for creating models used by services
- In "Service Instantiation" section: Add note referencing cli-architecture skill for command-level instantiation patterns

**testing-services skill updates:**
- In test organization section: Reference python-code-style skill for code organization conventions
- In "Mock at Boundaries Rule" section: Reference domain-modeling skill for understanding what domain models are and why not to mock them

**identifying-layer-placement skill updates:**
- In "Entry Point Layer" section: Reference cli-architecture skill for detailed command dispatcher patterns
- In "Domain Layer" section: Reference domain-modeling skill for comprehensive domain modeling guidance
- In "Service Layer" section: Reference dependency-injection skill for service constructor patterns

**New skills - add "Related Skills" section:**

Each new skill gets a Related Skills section at the end (before Further Reading):

- **domain-modeling**: References creating-services (for using models in services), testing-services (for testing models), identifying-layer-placement (for understanding domain layer)
- **cli-architecture**: References creating-services (for service instantiation), identifying-layer-placement (for entry point layer details)
- **dependency-injection**: References creating-services (for constructor patterns in services), cli-architecture (for configuration flow)
- **python-code-style**: References creating-services (for method organization in services), testing-services (for test code organization)

**Expected outcome:** Interconnected skill network where users can easily discover related guidance.

- [ ] Phase 7: Update README.md

Update README.md to document all 7 skills:

**Changes to make:**

1. **Overview section:**
   - Change "3 specialized skills" to "7 specialized skills"
   - Update description to mention the breadth of coverage (service creation, testing, architecture, domain modeling, CLI structure, dependency injection, code style)

2. **Skills list:**
   - Keep existing 3 skills: creating-services, testing-services, identifying-layer-placement
   - Add 4 new skills with descriptions:
     - **domain-modeling**: Design rich domain models with parse-once principle, factory methods, and type-safe APIs following Repository pattern
     - **cli-architecture**: Structure CLI applications using command dispatcher pattern with proper service instantiation and explicit parameter flow
     - **dependency-injection**: Apply constructor-based dependency injection with fail-fast principles and configuration flowing from entry points
     - **python-code-style**: Follow Python code organization conventions including method ordering, timezone-aware datetimes, and type annotations

3. **When to Use / Activation Triggers:**
   - Add triggers for new skills:
     - domain-modeling: "when creating domain models", "parsing YAML/JSON data", "designing data structures"
     - cli-architecture: "building CLI tools", "structuring commands", "command dispatcher pattern"
     - dependency-injection: "designing service constructors", "managing dependencies", "configuration flow"
     - python-code-style: "organizing service methods", "handling dates and times", "avoiding circular imports"

4. **Example Scenarios:**
   - Add scenarios showing when to use the new skills:
     - "Creating a User model that parses from YAML" → Use domain-modeling skill
     - "Building a multi-command CLI tool" → Use cli-architecture skill
     - "Service constructor keeps reading env vars" → Use dependency-injection skill
     - "Tests failing due to naive datetimes" → Use python-code-style skill

5. **Installation/Usage:**
   - Verify instructions still accurate (they should be, as skills are auto-discovered)

**Expected outcome:** README accurately documents all 7 skills with clear descriptions and usage guidance.

- [ ] Phase 8: Validation

Verify the integration is complete and consistent:

**1. Skill structure validation:**
- Run: `find plugin/skills -name "SKILL.md" | wc -l` → Should show 7
- Check each new skill has:
  - Valid YAML frontmatter (name, description, user-invocable: true, argument-hint)
  - Quick Reference section at top
  - Detailed sections with code examples
  - Consistent use of ✅ and ❌ markers
  - Properly formatted Python code blocks with triple backticks
  - Related Skills section (for new skills)
  - Further Reading section at end

**2. Content validation:**
- Read through each skill checking:
  - No significant duplication between skills (each has unique focus)
  - Cross-references are accurate (skill names match, sections exist)
  - Examples are practical and demonstrate principles clearly
  - Anti-patterns clearly marked and explained
  - Code examples are complete and runnable

**3. File structure validation:**
- Verify directory structure:
  ```
  plugin/skills/
  ├── creating-services/SKILL.md
  ├── testing-services/SKILL.md
  ├── identifying-layer-placement/SKILL.md
  ├── domain-modeling/SKILL.md
  ├── cli-architecture/SKILL.md
  ├── dependency-injection/SKILL.md
  └── python-code-style/SKILL.md
  ```
- Check README.md mentions all 7 skills
- Verify no broken internal references

**4. Documentation validation:**
- README.md lists all 7 skills with descriptions
- Activation triggers make sense and cover main use cases
- Installation instructions unchanged
- Example scenarios reference appropriate skills

**5. Cross-reference validation:**
- Check that all mentioned skills exist
- Verify cross-references are bidirectional where appropriate
- Test that references point to actual sections

**6. Manual review:**
- Read through each new skill end-to-end
- Verify flow is logical and easy to follow
- Check that the "When to Use This Skill" section clearly identifies use cases
- Ensure Related Skills section helps users discover connected topics

**Success criteria:**
- 7 complete skills (3 existing enhanced + 4 new)
- Clear separation of concerns with minimal duplication
- Comprehensive coverage of Python architecture documentation
- Consistent formatting throughout (YAML, code examples, markers)
- Accurate cross-references enabling skill discovery
- README.md accurately documents all skills with usage guidance
- All validation checks pass
