# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`target-mysql` is a Singer target for MySQL databases, built with the Meltano Singer SDK. It receives data from Singer taps in the standard Singer format (SCHEMA, RECORD, and STATE messages) and writes them to MySQL tables.

## Development Commands

### Setup
```bash
pipx install poetry
poetry install
```

### Testing
```bash
# Run all tests
poetry run pytest

# Run specific test
poetry run pytest tests/test_core.py::test_name

# Start MySQL container for testing
docker compose -f docker-compose.yml up -d
```

### Linting and Formatting
```bash
# Check code formatting and style
poetry run tox -e lint

# Auto-format code
poetry run tox -e format

# Individual linting tools
poetry run black target_mysql
poetry run isort target_mysql
poetry run flake8 target_mysql
```

### Running the Target
```bash
# Get help
poetry run target-mysql --help

# Show configuration options
poetry run target-mysql --about

# Run with a tap (example)
tap-carbon-intensity | poetry run target-mysql --config config.json
```

## Architecture

### Core Components

**target_mysql/target.py (TargetMySQL)**
- Entry point for the Singer target
- Validates configuration (either `sqlalchemy_url` OR individual connection params required)
- Enforces single-threaded operation (`max_parallelism = 1`) to avoid MySQL concurrency issues
- Handles SCHEMA and RECORD message validation

**target_mysql/connector.py (MySQLConnector)**
- Manages SQLAlchemy database connections and DDL operations
- Handles JSON Schema to MySQL type conversion with specific precedence rules
- Key type mapping decisions:
  - `integer` → `BIGINT`
  - `number` → `DECIMAL(65,30)`
  - `object`/`array` → `JSON`
  - `format: date-time` → `TIMESTAMP`
  - VARCHAR primary keys limited to 191 chars (utf8mb4 constraint)
  - Non-PK VARCHAR fields use `max_varchar_size` config (default 255)

**target_mysql/sinks.py (MySQLSink)**
- Implements the core write strategy using temp tables and upsert logic
- Process flow per batch:
  1. Create/verify main table exists
  2. Create temporary table with same schema
  3. Bulk insert batch records to temp table (deduplicating by PK if not append-only)
  4. Upsert from temp to main table
  5. Drop temp table
- Append-only mode: streams with no `key_properties` append all records without deduplication
- Upsert mode: uses LEFT OUTER JOIN to insert new records, then UPDATE existing ones
- Handles `activate_version` messages for hard/soft deletes based on `hard_delete` config

### Type System Precedence

When multiple JSON Schema types are present (e.g., `anyOf`, union types), the connector selects types using this precedence order (highest to lowest):
1. JSON
2. VARCHAR
3. TIMESTAMP
4. DATETIME
5. DATE
6. TIME
7. DECIMAL
8. BIGINT
9. INTEGER
10. BOOLEAN

### Configuration Notes

- Default MySQL port is 3306
- Default schema is "melty"
- The `dialect+driver` defaults to "mysql+mysqldb" and should generally not be changed
- `add_record_metadata` must be enabled (default: true) for `activate_version` to work
- MySQL row size limit is 65535 bytes - consider VARCHAR sizes carefully

### Testing

Tests use a MySQL container (root/password@127.0.0.1:3306/melty) defined in docker-compose.yml. The test suite includes:
- Standard SDK target tests
- Custom tests for MySQL-specific behavior (type mapping, upserts, soft/hard deletes)
- Singer message format validation tests

Test data files are in `tests/data_files/*.singer` and sample taps are in `tests/samples/`.

### Python Compatibility

Supports Python 3.8 through 3.11 (but not 3.12+). SQLAlchemy is pinned to version 1.x.

### System Dependencies

Requires `mysqlclient` library which needs MySQL development headers:
```bash
# Ubuntu/Debian
sudo apt-get install libmysqlclient-dev
```
