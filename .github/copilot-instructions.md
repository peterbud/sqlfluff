# SQLFluff AI Coding Assistant Instructions

## Project Overview

SQLFluff is a dialect-flexible SQL linter and auto-fixer supporting 25+ SQL dialects. The architecture is built around:
- **Parser-first design**: SQL is lexed → parsed into segment trees → linted by rules → optionally fixed
- **Dialect inheritance**: Each dialect (in `src/sqlfluff/dialects/`) extends ANSI (`dialect_ansi.py`) using `.replace()` to override grammar segments
- **Segment-based AST**: Everything is a `BaseSegment` subclass with recursive tree structure
- **Rule crawlers**: Rules traverse segment trees to find violations and generate `LintFix` objects

## Repository Structure

The SQLFluff repository is organized as follows:

### Core Source Code
- **`src/sqlfluff/`**: Main package source code
  - `dialects/`: SQL dialect definitions (ANSI, T-SQL, BigQuery, PostgreSQL, MySQL, etc.)
  - `rules/`: Linting rules organized by category (layout, capitalisation, references, structure, etc.)
  - `core/`: Core parser, lexer, and configuration infrastructure
    - `parser/`: Segment classes, grammar primitives, parsing logic
    - `parser/grammar/`: Grammar composition primitives (`Sequence`, `OneOf`, `Ref`, etc.)
    - `parser/segments/`: Base segment classes and common segment types
  - `cli/`: Command-line interface implementation
  - `api/`: Public Python API
  - `utils/`: Utility functions and helpers

### Testing Infrastructure
- **`test/`**: Comprehensive test suite
  - `dialects/`: Dialect-specific parsing tests
  - `fixtures/dialects/<dialect>/`: SQL test files (`.sql`) and expected parse trees (`.yml`)
  - `rules/`: Rule testing infrastructure
  - `fixtures/rules/`: YAML test cases for linting rules
  - `api/`, `cli/`, `core/`: Unit tests for corresponding source modules
  - `generate_parse_fixture_yml.py`: Script to regenerate parser test fixtures

### Documentation & Configuration
- **`docs/`**: Sphinx-based documentation source
- **`.github/`**: GitHub-specific files (workflows, copilot instructions)
- **`pyproject.toml`**: Main project metadata and dependencies
- **`tox.ini`**: Test automation configuration
- **`requirements_dev.txt`**: Development dependencies

### Plugins & Extensions
- **`plugins/`**: Pluggable extensions
  - `sqlfluff-templater-dbt/`: dbt templater plugin
  - `sqlfluff-plugin-example/`: Example plugin template

### Rust Components (Optional)
- **`sqlfluffrs/`**: Experimental Rust reimplementation of performance-critical components
  - Used for accelerating lexing and parsing in production environments

### Utilities & Build Tools
- **`utils/`**: Build and development utilities
  - `build_dialect.py`, `build_dialects.py`: Dialect build automation
  - `build_lexers.py`: Lexer generation
- **`examples/`**: API usage examples
- **`docker/`**: Docker development environment

### Key Files
- **`.sqlfluff`**: Example SQLFluff configuration file
- **`CONTRIBUTING.md`**: Contribution guidelines
- **`CHANGELOG.md`**: Version history and release notes

## Core Architecture

### Dialects (`src/sqlfluff/dialects/`)
- Each dialect file (e.g., `dialect_tsql.py`, `dialect_bigquery.py`) defines a `Dialect` object and grammar segments
- Dialects inherit from ANSI via `inherits_from="ansi"` and use `.replace()` to override specific segments
- Grammar composition uses `Sequence()`, `OneOf()`, `Delimited()`, `AnyNumberOf()`, `Ref()`, `Bracketed()` from `src/sqlfluff/core/parser/grammar/`
- Segment classes inherit from `BaseSegment` and define `type`, `match_grammar`, and optional methods
- Keywords are separated into `*_keywords.py` files as `reserved` and `unreserved` keyword lists

#### T-SQL Syntax Notation (Microsoft Docs)
When implementing T-SQL dialect features from Microsoft documentation, translate their syntax conventions to SQLFluff grammar:

| Microsoft Notation | Meaning | SQLFluff Translation |
|-------------------|---------|---------------------|
| `UPPERCASE` | T-SQL keywords | Literal string `"UPPERCASE"` |
| *italic* | User-supplied parameter | `Ref("SegmentName")` |
| **bold** | Literal identifier/name | Literal string or `Ref()` |
| `|` (pipe) | Choice between items | `OneOf(...)` |
| `[ ]` (brackets) | Optional element | `optional=True` or `Ref(..., optional=True)` |
| `{ }` (braces) | Required choice | `OneOf(...)` without optional |
| `[, ...n]` | Comma-separated repetition | `Delimited(...)` |
| `[...n]` | Space-separated repetition | `AnyNumberOf(...)` or `Sequence(..., optional=True)` repeatedly |
| `;` | Statement terminator | `Ref("SemicolonSegment")` |
| `<label> ::=` | Named syntax block | Define as separate segment: `class LabelSegment(BaseSegment)` |

**Example translation:**
```
Microsoft: CREATE TABLE <table_name> ( <column_definition> [, ...n] )
SQLFluff: Sequence("CREATE", "TABLE", Ref("TableReferenceSegment"),
                   Bracketed(Delimited(Ref("ColumnDefinitionSegment"))))
```

### Rules (`src/sqlfluff/rules/`)
- Organized by category: `layout/`, `capitalisation/`, `references/`, `structure/`, `ambiguous/`, `convention/`, etc.
- Each rule inherits from `BaseRule` and implements `_eval()` or uses a `BaseCrawler` subclass
- Return `LintResult` objects with optional `LintFix` arrays for auto-fixing
- Rules have metadata: `code` (e.g., "AL01"), `name`, `description`, `groups`, `config` schema

### Parser (`src/sqlfluff/core/parser/`)
- `segments/base.py`: `BaseSegment` - the fundamental AST node
- `grammar/`: Composable grammar primitives (`Sequence`, `OneOf`, `Ref`, etc.)
- `lexer.py`: Tokenizes SQL via `RegexLexer` and `StringLexer` matchers
- Segments have `match_grammar` that defines parsing rules, evaluated recursively

## Development Workflows

### Environment Setup
- **Always activate the virtual environment** before running scripts:
  ```bash
  source .venv/bin/activate
  ```
- Create dev environment if it doesn't exist:
  ```bash
  tox -e py312 --devenv .venv
  source .venv/bin/activate
  ```

### Testing
```bash
# Run tests for specific Python version
tox -e py312

# Test specific rules (faster for rule development)
tox -e py312 -- test/rules/yaml_test_cases_test.py -k AL01

# Test dialect parsing (generate fixtures for specific dialect)
tox -e generate-fixture-yml -- -d mysql

# Test complete dialect (e.g., T-SQL)
python test/generate_parse_fixture_yml.py -d tsql

# Full test suite
tox -e cov-init,py312,cov-report,linting,mypy
```

### Coverage Testing
```bash
# Run coverage for specific rule tests
python -m pytest test/rules/ -k AL03 --cov=src/sqlfluff/rules --cov-report=term-missing

# Run coverage for all rule tests (shows which lines are not covered)
python -m pytest test/rules/ --cov=src/sqlfluff/rules --cov-report=term-missing:skip-covered

# Run coverage for dialect tests
python -m pytest test/dialects/ -k tsql --cov=src/sqlfluff/dialects --cov-report=term-missing

# Full coverage report (same as tox command but more direct)
python -m pytest test/ --cov=src/sqlfluff --cov-report=term-missing

# Coverage with HTML report (creates htmlcov/ directory)
python -m pytest test/ --cov=src/sqlfluff --cov-report=html
```

### Linting
```bash
# Run pre-commit hooks (linting and auto-fixes) before commits
.venv/bin/pre-commit run --all-files
```

### Adding Dialect Features
1. Add `.sql` test file to `test/fixtures/dialects/<dialect>/`
2. Run `python test/generate_parse_fixture_yml.py -d <dialect>` to generate `.yml` structure
3. Define/override segments in `src/sqlfluff/dialects/dialect_<name>.py`
4. Use `dialect.replace()` to override inherited segments
5. Verify with `tox -e generate-fixture-yml -- -d <dialect>`

### Dialect Test File Conventions
- **Organize by segment type**: Create separate test files for each major segment (e.g., `create_table.sql`, `select_statement.sql`, `merge_statement.sql`)
- **Comprehensive lexical coverage**: Each test file should include variations testing different:
  - Keyword combinations and optional clauses
  - Identifier formats (quoted, unquoted, schema-qualified)
  - Literal types (strings, numbers, dates)
  - Comments and whitespace edge cases
- **Test file naming**: Use descriptive names matching the segment being tested: `create_materialized_view.sql`, `alter_table_add_column.sql`
- **Multiple test cases per file**: Include several SQL statements in each `.sql` file to cover edge cases without creating excessive files
- **Example structure**:
  ```
  test/fixtures/dialects/bigquery/
    create_table.sql              # Various CREATE TABLE syntaxes
    create_function.sql           # UDF creation patterns
    select_struct.sql             # STRUCT literal variations
    select_array.sql              # ARRAY operations
  ```
- **Avoid monolithic files**: Don't put all dialect tests in one file - this makes debugging failures harder and slows iteration

### Adding Rules
1. Create rule in appropriate category under `src/sqlfluff/rules/`
2. Define rule metadata: `code`, `name`, `description`, `groups`, `crawl_behaviour`
3. Implement `_eval(context: RuleContext) -> Optional[LintResult]`
4. Add test cases to `test/fixtures/rules/std_rule_cases/<category>.yml`
5. Use YAML format: `test_name:`, `fail_str:`, optional `fix_str:`, optional `configs:`
6. Run `tox -e py312 -- test/rules/yaml_test_cases_test.py -k <rule_code>`

## Python Coding Guidelines

SQLFluff follows strict Python coding standards enforced through automated tooling:

### Code Style & Formatting
- **Black**: Auto-formatter for consistent code style (line length, quotes, spacing)
- **Ruff**: Fast Python linter with import sorting (isort) and pydocstyle checks
- **Flake8**: Additional linting with flake8-black plugin
- **Pre-commit hooks**: Run all formatters/linters automatically before commits
  - Use `.venv/bin/pre-commit run --all-files` to check/fix all files

### Type Annotations
- **Mypy**: Strict type checking enabled with `warn_unused_configs`, `strict_equality`, `no_implicit_reexport`
- **Type hints required**: Use `from typing import Optional, Union, cast, TYPE_CHECKING` etc.
- **Future imports**: Use `from __future__ import annotations` for forward references in complex files
- **Runtime checks**: Use `TYPE_CHECKING` to avoid circular imports in type annotations

### Documentation Standards
- **Google-style docstrings**: Required for all public modules, classes, and functions
- **Exceptions**: Magic methods (D105) and `__init__` (D107) don't require docstrings
- **Docstring format**:
  ```python
  """Short one-line summary.

  Longer description if needed. Explain purpose, behavior, and usage.

  Args:
      param_name: Description of parameter.

  Returns:
      Description of return value.
  """
  ```

### Import Organization
- **Import order** (enforced by Ruff isort):
  1. Standard library imports
  2. Third-party imports
  3. First-party imports (`sqlfluff`, `sqlfluff_plugin_example`, `sqlfluff_templater_dbt`, `test`)
- **Import grouping**: Group related imports, use explicit imports over `import *`
- **Import linter**: Tool in `pyproject.toml` enforces architectural boundaries (e.g., core can't import cli/api)

### Architectural Principles
- **Layer separation**: Enforced by `tool.importlinter` contracts in `pyproject.toml`
  - `core` cannot import `api`, `cli`, `dialects`, `rules`, `utils`
  - `api` cannot import `cli`
  - Dependency layers: `linter` → `rules` → `parser` → `errors`/`types` → `helpers`
- **Immutability**: Segments are immutable; use `.copy()` or `LintFix` for modifications
- **Lazy loading**: Dialects loaded via `dialect_selector()` or `load_raw_dialect()`, not direct imports

### Testing Conventions
- **Test file naming**: `*_test.py` (enforced by pytest config)
- **Test organization**: Mirror source structure in `test/` directory
- **Fixtures**: Use pytest fixtures in `conftest.py`, YAML fixtures for dialect/rule tests
- **Markers**: Use `@pytest.mark.dbt`, `@pytest.mark.integration` for test categorization

### Code Quality Tools Configuration
All tools configured in `pyproject.toml`:
- **Black**: Default settings
- **Ruff**: Extends with isort (I) and pydocstyle (D) rules
- **Mypy**: Strict mode with specific overrides for 3rd-party packages
- **Pytest**: Custom markers for test categorization
- **Codespell**: Spell checking with custom ignore lists

Run quality checks:
```bash
.venv/bin/pre-commit run --all-files  # All checks
black src/ test/                       # Format code
ruff check src/ test/                  # Lint code
mypy src/sqlfluff/                     # Type check
```

## Critical Conventions

### Grammar Organization and Reusability

When defining complex statement segments, follow these patterns for organizing grammar rules:

#### Internal Grammar Rules (Underscore Prefix)
Use **private class attributes** with underscore prefix (`_`) for grammar components that are:
- Specific to a single statement type
- Used only within that segment's `match_grammar`
- Not needed by other segments in the dialect

**Example from `CreateDatabaseStatementSegment`:**
```python
class CreateDatabaseStatementSegment(BaseSegment):
    """A `CREATE DATABASE` statement."""

    # Internal grammar - specific to CREATE DATABASE only
    _filestream_option = OneOf(
        Sequence("NON_TRANSACTED_ACCESS", Ref("EqualsSegment"), ...),
        Sequence("DIRECTORY_NAME", Ref("EqualsSegment"), ...),
    )

    _create_database_option = OneOf(
        Sequence("FILESTREAM", Bracketed(Delimited(_filestream_option))),
        Sequence("DEFAULT_LANGUAGE", Ref("EqualsSegment"), ...),
    )

    _create_database_normal = Sequence(
        Sequence("CONTAINMENT", Ref("EqualsSegment"), ..., optional=True),
        Sequence("ON", ..., optional=True),
        Sequence("WITH", Delimited(_create_database_option), optional=True),
    )

    type = "create_database_statement"
    match_grammar = Sequence(
        "CREATE", "DATABASE", Ref("DatabaseReferenceSegment"),
        OneOf(_create_database_normal, _create_database_attach, optional=True),
    )
```

#### Shared Grammar Segments (Named Classes)
Create **separate segment classes** for grammar components that:
- Are reusable across multiple statement types
- Represent logical SQL constructs (file specs, clauses, options)
- Need to be referenced by name in other segments

**Example of reusable segments:**
```python
class LogicalFileNameSegment(BaseSegment):
    """Logical file name - used in CREATE/ALTER DATABASE."""
    type = "logical_file_name"
    match_grammar = Sequence("NAME", Ref("EqualsSegment"), Ref("QuotedLiteralSegment"))

class FileSpecSegment(BaseSegment):
    """File specification - reusable in CREATE/ALTER statements."""
    type = "file_spec"
    match_grammar = Bracketed(
        Sequence(
            Ref("LogicalFileNameSegment", optional=True),
            Ref("FileSpecFileNameSegment"),
            Ref("FileSpecSizeSegment", optional=True),
        )
    )
```

#### Decision Criteria
**Use internal grammar (`_prefix`) when:**
- Grammar is specific to one statement type
- No other segments need to reference it
- Breaking down complex `match_grammar` for readability
- Creating logical groupings within a statement

**Use shared segments (classes) when:**
- Multiple statement types use the same construct (e.g., `FileSpecSegment` in both `CREATE DATABASE` and `ALTER DATABASE`)
- The construct represents a meaningful SQL element that should appear in the AST
- Other rules or segments need to `Ref()` it by name
- The grammar component has semantic meaning beyond one statement

#### Benefits of This Pattern
1. **Readability**: Complex statements remain manageable by breaking grammar into logical chunks
2. **Maintainability**: Changes to shared constructs propagate automatically via `Ref()`
3. **DRY principle**: Reusable components defined once, referenced many times
4. **AST structure**: Shared segments create meaningful parse tree nodes
5. **Testing**: Shared segments can be tested independently

### Segment Definition Pattern
```python
class MyStatementSegment(BaseSegment):
    """Description."""
    type = "my_statement"
    match_grammar = Sequence(
        "SELECT",
        Ref("SelectClauseSegment"),
        Ref("FromClauseSegment"),
    )
```

### Dialect Override Pattern
```python
tsql_dialect.replace(
    SelectStatementSegment=Sequence(
        "SELECT",
        Ref("TopClauseSegment", optional=True),  # T-SQL specific
        Ref("SelectClauseSegment"),
    ),
)
```

### Rule Pattern
```python
class Rule_AL01(BaseRule):
    """Implicit aliasing of table not allowed."""
    groups = ("all", "aliasing")
    crawl_behaviour = SegmentSeekerCrawler({"table_reference"})

    def _eval(self, context: RuleContext) -> Optional[LintResult]:
        # Return LintResult with anchor and fixes
        return LintResult(anchor=segment, fixes=[LintFix(...)])
```

### Testing Pattern (YAML)
```yaml
test_descriptive_name:
  fail_str: SELECT * FROM x
  fix_str: SELECT * FROM x AS x_alias
  configs:
    rules:
      aliasing.table:
        aliasing: explicit
```

## Configuration

- Config files: `.sqlfluff` (INI format) in project root or any parent directory
- Programmatic config: `FluffConfig.from_root(overrides={...})`
- Key settings: `dialect`, `rules`, `exclude_rules`, `templater`, layout config
- Rule-specific config in `[sqlfluff:rules:<rule_code>]` sections

## Plugin System

- Plugins in `plugins/` directory (e.g., `sqlfluff-templater-dbt/`)
- Installed via `pip install -e plugins/<plugin-name>`
- Entry points defined in `pyproject.toml` under `[project.entry-points]`

## Templating

- Default: Jinja2 templating support
- dbt templater: Separate plugin, requires dbt-core + dbt-postgres
- SQL placeholders and Python format strings also supported

## Common Pitfalls

- **Don't modify segments directly**: Parser creates immutable trees, use `.copy()` or fix mechanisms
- **Dialect loading is lazy**: Use `dialect_selector()` not direct imports
- **Grammar references**: Use `Ref("SegmentName")` not direct class references in grammars
- **Test isolation**: Dialect tests should be in highest applicable dialect (prefer ANSI over vendor-specific)
- **Parser test regeneration**: Always regenerate YAML fixtures after grammar changes
