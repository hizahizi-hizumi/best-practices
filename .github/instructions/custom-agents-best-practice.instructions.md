---
description: 'Best practices for creating VS Code Copilot custom agents'
applyTo: '**/.github/agents/**/*.agent.md, **/.github/chatmodes/**/*.chatmode.md'
---

# Custom Agent Creation Guidelines

Best practices for creating custom agents specialized for specific development tasks in VS Code.

## Core Principles

### 1. Principle of Least Privilege
- Grant only the minimum necessary tools
- Exercise particular caution when using the `execute` tool
- Grant only `["read", "search"]` for read-only tasks

### 2. Clear Role Definition
- Describe the agent's role in a single sentence
- Clearly define the scope of responsibility
- Explicitly state what should and should not be done

### 3. Concrete Instructions
- Provide actionable and specific steps
- Avoid abstract expressions
- Clearly specify the output format

## YAML Front Matter

### description (Required)

Describe the agent's purpose and functionality in 50-100 characters.

```yaml
# Recommended
description: Test specialist that improves test coverage without modifying production code

# Not Recommended
description: Testing agent
```

**Reason**: Displayed as a placeholder in the chat input field

### name

Specify a clear, purpose-driven name in kebab-case (e.g., `test-specialist`).

### tools - Minimum Privilege Settings by Task

**Grant only the minimum necessary tools.**

| Task Type | Recommended Tools | Reason |
|----------|----------|------|
| Planning/Analysis | `["read", "search", "fetch"]` | Read-only, prevents accidental modifications |
| Implementation | `["read", "edit", "search", "execute"]` | Code changes and execution |
| Review | `["read", "search"]` | Verification only, no changes required |

```yaml
# Recommended - Read-only
tools: ["read", "search", "fetch"]

# Not Recommended - Full privileges
tools: ["*"]
```

**Reason**: Minimizes security risks and clarifies the agent's role

**Available Tool Aliases**:
- `read` - Read file contents
- `edit` - Edit files (str_replace, etc.)
- `search` - Search files and text (grep, glob)
- `execute` - Execute shell commands
- `agent` - Invoke other custom agents
- `web` - Fetch URL content, web search
- `todo` - Structured task list management

**Specifying MCP Server Tools**:
```yaml
tools: ["read", "edit", "github/list-repos"]  # Specific tool
tools: ["playwright/*"]  # All tools from a server
```

### model

Select a model based on task complexity.

```yaml
model: Claude Sonnet 4  # Complex analysis and implementation
model: Claude Haiku     # Simple formatting changes
```

**Reason**: Cost optimization and improved response speed

### target

Restrict the execution environment.

```yaml
target: vscode           # VS Code only
target: github-copilot   # GitHub Copilot only
```

### infer

Control automatic agent selection.

```yaml
infer: false  # When manual selection is required
# Default is true
```

### handoffs - Designing Inter-Agent Transitions

Define sequential workflows.

```yaml
handoffs:
  - label: Review the plan
    agent: reviewer
    prompt: Please review this implementation plan and provide improvement suggestions.
    send: false
```

**Using the send Property**:
- `send: false` - User confirms/edits the prompt (recommended)
- `send: true` - Auto-send prompt (only for certain next steps)

### argument-hint

Provide hint text displayed in the chat input field.

```yaml
argument-hint: "Enter a feature name or description"
```

## Agent Body (Prompt) Design

### Clarifying Role and Responsibilities

Define the agent's role in a single sentence and clearly describe the scope of responsibility.

```markdown
# Planning Specialist

As a technical planning specialist, I focus on creating comprehensive implementation plans.

## Scope of Responsibility

- Analyze requirements and break them down into actionable tasks
- Document detailed technical specifications and architecture
- Generate plans with clear steps, dependencies, and timelines

## Constraints

- Do not implement code; focus on thorough documentation
- Structure plans with clear headings, task breakdowns, and acceptance criteria
```

### Task-Specific Instructions

Provide concrete and actionable steps in list format.

```markdown
## Code Review Process

1. **Security**: Check for SQL injection, XSS, authentication issues
2. **Performance**: Identify inefficient algorithms and unnecessary loops
3. **Code Quality**: Verify coding standards, naming conventions, and comments
4. **Test Coverage**: Evaluate whether changes are covered by tests
```

### References to Other Files

Reference other files using Markdown links to avoid duplication.

```markdown
For detailed coding standards, refer to the project's coding conventions.
```

### Tool Reference Notation

Use the `#tool:<tool-name>` syntax to reference tools.

```markdown
Use #tool:search to explore the codebase.
```

## Security Best Practices

### Secret Management

Do not include secrets in plaintext in files.

```yaml
# Recommended
env:
  API_KEY: ${{ secrets.API_KEY }}

# Not Recommended
env:
  API_KEY: "my-secret-key"
```

**Reason**: Prevents risk of secret leakage

For organization-level agents, set them in the repository's "copilot" environment and use the `${{ secrets.SECRET_NAME }}` syntax.

### Tool Access Restrictions

Grant only the minimum necessary tools, and exercise particular caution with the `execute` tool.

```yaml
# Security review agent
tools: ["read", "search"]  # execute not required
```

### Environment Isolation

Restrict the execution environment using the `target` property.

```yaml
target: vscode  # Use only in VS Code
```

## Common Anti-Patterns and Solutions

### ❌ Tool Privileges Too Broad

```yaml
# Not Recommended
tools: ["*"]
```

**Solution**: Grant only the minimum tools required for the task

```yaml
# Recommended
tools: ["read", "search"]  # For analysis tasks
```

### ❌ Unclear Instructions

```markdown
# Not Recommended
Improve the code.
```

**Solution**: Concrete and actionable instructions

```markdown
# Recommended
1. Remove redundant code
2. Break down complex functions into smaller ones
3. Add appropriate error handling
```

### ❌ Using the Same Model for All Tasks

**Solution**: Select models based on task complexity

```yaml
model: Claude Sonnet 4  # Complex analysis
model: Claude Haiku     # Simple formatting changes
```

## Performance Optimization

### Efficient Tool Selection

Do not include unnecessary tools (causes processing slowdown).

### Prompt Size Management

Prompts have a maximum of 30,000 characters; use reference links for large files.

### Model Selection Optimization

Use lightweight models (Haiku) for lightweight tasks, and high-performance models (Sonnet) only for complex tasks.

## Maintainability and Scalability

### Modularization

Separate reusable instructions into separate files and reference them with Markdown links.

### Version Control

Manage agent files with Git and track change history.

### Unified Naming Conventions

Agent names should clearly reflect their purpose, and file names should use kebab-case (e.g., `test-specialist.agent.md`).

## Organization-Level Best Practices

### Organization/Enterprise-Level Agents

Place them in the `agents/` directory of the `.github-private` repository; they can include MCP server configuration.

```yaml
---
name: org-security-reviewer
description: Review agent based on organizational security standards
tools: ['read', 'search', 'security-scanner/*']
mcp-servers:
  security-scanner:
    type: 'local'
    command: 'security-tool'
    args: ['--config', 'org-config.json']
    tools: ["*"]
    env:
      API_KEY: ${{ secrets.SECURITY_API_KEY }}
---
```

### MCP Server Configuration

**Syntax for Referencing Environment Variables and Secrets**:
- `${{ secrets.SECRET_NAME }}` - Secret reference
- `${{ var.VARIABLE_NAME }}` - Environment variable reference

## Practical Examples

### Example 1: Test Specialist Agent

```markdown
---
name: test-specialist
description: Test specialist that improves test coverage without modifying production code
tools: ["read", "edit", "search", "execute"]
handoffs:
  - label: Request code review
    agent: code-reviewer
    prompt: Please review the test code I created.
    send: false
---

# Test Specialist

Improve code quality through comprehensive testing.

## Scope of Responsibility

- Analyze existing tests and identify coverage gaps
- Create tests following best practices
- Ensure tests are independent, deterministic, and well-documented
- Do not modify production code unless specifically requested

## Execution Guidelines

- Provide clear test descriptions
- Use test patterns appropriate for the language and framework
- Prioritize readability and maintainability
```

### Example 2: Implementation Planning Agent

```markdown
---
name: implementation-planner
description: Create detailed implementation plans and technical specifications in Markdown format
tools: ["read", "search", "fetch"]
model: Claude Sonnet 4
handoffs:
  - label: Start implementation
    agent: agent
    prompt: Please implement the above plan.
    send: false
---

# Implementation Planning Specialist

Focus on creating comprehensive implementation plans.

## Scope of Responsibility

- Analyze requirements and break them down into actionable tasks
- Document detailed technical specifications and architecture
- Generate plans with clear steps, dependencies, and timelines

## Execution Guidelines

- Structure plans with clear headings, task breakdowns, and acceptance criteria
- Include considerations for testing, deployment, and potential risks
- Focus on thorough documentation rather than code implementation
```

### Example 3: Code Review Agent

```markdown
---
name: code-reviewer
description: Review code from security, performance, and quality perspectives
tools: ["read", "search"]
---

# Code Review Specialist

Conduct thorough code reviews from security, performance, and code quality perspectives.

## Review Process

1. **Security**: Check for SQL injection, XSS, CSRF, authentication mechanisms
2. **Performance**: Identify inefficient algorithms and unnecessary loops
3. **Code Quality**: Verify coding standards, naming conventions, and comments
4. **Test Coverage**: Confirm that changes are covered by tests

## Output Format

- **Critical Issues**: Security risks and bugs
- **Improvement Suggestions**: Performance and code quality enhancements
- **Minor Comments**: Style and naming improvements
- **Positive Aspects**: Commendable implementation details
```

## Summary: Key Principles

1. **Clear Role Definition** - Describe the agent's purpose and responsibilities in a single sentence
2. **Principle of Least Privilege** - Grant only the minimum necessary tools
3. **Concrete Instructions** - Provide actionable and clear instructions
4. **Leverage Handoffs** - Design smooth transitions between agents
5. **Security** - Enforce secret management and tool access restrictions
6. **Modularization** - Separate reusable instructions to improve maintainability
7. **Appropriate Model Selection** - Choose models based on task complexity
