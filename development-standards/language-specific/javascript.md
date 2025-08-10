# JavaScript-Specific Development Standards

## Security Focus Areas

### Client-Side Vulnerabilities
- **XSS Prevention**: Always sanitize user input, use textContent over innerHTML
- **DOM Manipulation**: Validate all dynamic DOM insertions
- **Prototype Pollution**: Avoid unsafe object merging patterns

### Node.js Security
- **Dependency Management**: Regular `npm audit` and dependency updates
- **Command Injection**: Never use `eval()`, `child_process.exec()` with user input
- **Path Traversal**: Validate file paths, use `path.resolve()` safely

### Framework-Specific Patterns
- **React**: Avoid `dangerouslySetInnerHTML`, validate props
- **Express**: Use security middleware (helmet, cors), validate inputs
- **JWT**: Proper secret management, token expiration

## Code Quality Standards

### Complexity Thresholds
- Cyclomatic complexity: ≤ 10
- Function length: ≤ 50 lines
- Nested callbacks: ≤ 3 levels (prefer async/await)

### Common Anti-Patterns to Avoid
- Callback hell (use Promises/async-await)
- Global variable pollution
- Missing error handling in Promises
- Blocking synchronous operations in Node.js