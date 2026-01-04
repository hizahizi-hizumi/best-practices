---
description: 'Best Practices for Creating GitHub Copilot Custom Instructions Files'
applyTo: '**/*.instructions.md, .github/copilot-instructions.md'
---

# GitHub Copilot Custom Instructions Best Practices

Guidelines summarizing key points for effectively utilizing VS Code Custom Instructions.

## Core Principles

### 1. Conciseness and Clarity
- Express each instruction as a single, simple sentence
- Split multiple concepts into separate instructions
- Write specific, actionable instructions
- Avoid abstract expressions

### 2. Imperative and Specificity
- Start with verbs like "Use", "Avoid", "Validate"
- Do not use vague terms like "appropriately", "somehow", "if possible"
- Explicitly specify output format (bullet points, tables, JSON, etc.)

**Recommended**: `Use camelCase for variable names`
**Not Recommended**: `Use appropriate naming conventions`

### 3. Minimize Changes
- Do not refactor, format, or rename unless requested
- Prefer "integration with existing" over replacement
- Achieve objectives with minimal diffs

### 4. Explicit Priorities
In environments where multiple instructions are combined, specify conflict resolution criteria:

1. User's explicit instructions take highest priority
2. Security requirements always take priority
3. Project-specific rules
4. General best practices
5. Verify uncertain external information with tools

### 5. File Size
- Target 300-500 lines
- Include only high-priority key points
- Maximize reproducibility

## Choosing File Types

### `.github/copilot-instructions.md`
- **Characteristics**: Single file automatically applied workspace-wide
- **Use Cases**: Basic rules for entire project, tech stack information
- **Configuration**: `"github.copilot.chat.codeGeneration.useInstructionFiles": true`

### `.instructions.md`
- **Characteristics**: Multiple files possible, conditional application via glob patterns
- **Use Cases**: Language-specific conventions, framework-specific patterns
- **Scope**: Workspace (`.github/instructions/`) or user (profile folder)

Recommended directory structure:
```
.github/
├── copilot-instructions.md
└── instructions/
    ├── python-standards.instructions.md
    ├── react-standards.instructions.md
    └── security-guidelines.instructions.md
```

## YAML Frontmatter

### Basic Structure
```yaml
---
description: 'Purpose and scope of the file (concise description for humans)'
applyTo: '**/*.py'  # glob pattern
---
```

### applyTo Pattern Examples
```yaml
applyTo: '**/*.py'                    # All Python files
applyTo: '**/*.{jsx,tsx}'             # React components
applyTo: 'src/api/**/*'               # Specific directory
applyTo: '**/*.{test,spec}.{ts,js}'   # Test files
```

## Writing Instructions

### Recommended Sections
1. **Purpose and Scope**: Explain what is controlled in one sentence
2. **Applicable Scope**: Specify which files/directories are targeted
3. **Project Overview**: Tech stack, main features, target users
4. **Tools and Versions**: Precise commands and version information
5. **Coding Conventions**: Naming rules, style, formatting
6. **API Contracts** (Optional): References to type definitions and API specifications
7. **Security and Privacy** (Optional): Secure by Default, excluding confidential information, vulnerability countermeasures
8. **Performance Optimization** (Optional): Measure First, avoiding typical anti-patterns

### Presenting Good/Bad Examples
```typescript
// Recommended
interface User {
  id: string;
  name: string;
}

// Not Recommended
function getUser(id: any): any { }
```

### Attaching WHY (Rationale)
Supplement important rules with "why" in 1-2 sentences:

```markdown
- Use parameterized queries for SQL queries
  **Rationale**: String concatenation causes SQL injection attacks
```

### Utilizing References
- **Within Project**: `See types/api.ts for details`
- **External**: `[Google Style Guide](https://...)`
- **Code Files**: Specify `{ "file": "database/schema.sql" }` in settings.json

## Scope Management

### Conditional Application
Split files by technical domain and activate only when needed with `applyTo`:

```yaml
applyTo: '**/*.prompt.md'  # Target specific extensions
```

**Effect**: Reduce intrusion of irrelevant instructions and conserve context

### Hierarchical Structure
```markdown
# .github/copilot-instructions.md
- Follow DRY principle

# .github/instructions/python-standards.instructions.md
---
applyTo: '**/*.py'
---
In addition to project-wide rules:
- Conform to PEP 8
- Use type hints
```

## Considerations When Creating Files

### Trust Boundary Configuration

**Workspace Trust**: Recommend functionality restrictions in untrusted workspaces
**Tool Auto-Approval**: Require confirmation steps for destructive operations

```json
{
  "github.copilot.chat.tool.approval": "user",  // Recommended
  "github.copilot.chat.terminal.autoApprove": {
    "commands": ["git status", "npm test"]  // Least privilege
  }
}
```

## Team Development Operations

### Version Control
```gitignore
# Exclude user-specific
.vscode/settings.json

# Include team-shared
!.github/copilot-instructions.md
!.github/instructions/
!.github/prompts/
```

### Regular Maintenance
Every 3-6 months:
- Remove unnecessary rules
- Add new best practices
- Reflect team feedback
- Verify consistency with project changes

### Onboarding
For new members:
- Location and purpose of instructions files
- Required settings for activation
- Available prompts

### PR Template Integration
```markdown
## Copilot Checklist
- [ ] Generated code reviewed
- [ ] Security guidelines compliant
- [ ] No unintended changes (minimal change principle)
```

## Troubleshooting

### When Instructions Don't Apply
1. **Check Settings**: Is `useInstructionFiles: true`?
2. **Check Placement**: Location of `.github/copilot-instructions.md`
3. **Check Pattern**: Is `applyTo: '**/*.py'` correct?

### Different from Expected Generation
1. Check clarity of instructions (eliminate vague terms)
2. Add Good/Bad examples
3. Reference related files
4. Specify priorities

### Common Anti-Patterns

#### Verbose Explanations
```markdown
# Not Recommended: Multiple elements in one sentence
All functions should have proper documentation, types, error handling...

# Recommended: Split
- Write JSDoc for functions
- Specify types
- Document error cases
```

#### Vague Instructions
```markdown
# Not Recommended
Write good code

# Recommended
Functions follow single responsibility principle
```

#### Conflicting Instructions
```markdown
# Problem: Conflict in same scope
File 1: Detailed comments required
File 2: Minimize comments

# Solution: Specify priorities
1. Self-explanatory code first
2. Comment only "why"
```

## Reference Resources

- [VS Code - Custom Instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions)
- [awesome-copilot](https://github.com/github/awesome-copilot)
