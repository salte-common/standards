# Python-Specific Development Standards

## Security Focus Areas

### Injection Vulnerabilities
- **SQL Injection**: Always use parameterized queries, ORM safe practices
- **Code Injection**: Never use `eval()`, `exec()` with user input
- **Command Injection**: Use `subprocess` with shell=False, validate inputs

### Serialization Security
- **Pickle Safety**: Avoid `pickle` with untrusted data, use JSON when possible
- **YAML Loading**: Use `yaml.safe_load()` instead of `yaml.load()`

### Framework-Specific Patterns
- **Django**: Use CSRF protection, validate forms, secure settings
- **Flask**: Implement proper session management, input validation
- **FastAPI**: Use Pydantic models, implement proper authentication

## Code Quality Standards

### Python-Specific Metrics
- Cyclomatic complexity: ≤ 10
- Function length: ≤ 50 lines
- Class length: ≤ 300 lines
- Import organization: Follow PEP 8

### Pythonic Patterns
- Use list/dict comprehensions appropriately
- Prefer context managers for resource handling
- Follow PEP 8 naming conventions
- Use type hints for public APIs