---
description: 'Best practices for creating Agent Skills directories and SKILL.md files with discoverability, portability, security, and verifiability'
applyTo: '**/SKILL.md, **/.github/skills/**, **/.claude/skills/**'
---

# Agent Skills Creation Best Practices

Guidelines for creating discoverable, readable, secure, and verifiable Agent Skills directories and SKILL.md files.

## Core Principles

### Priorities
1. Security and safety always take highest priority
2. Discoverability (metadata optimization)
3. Portability (cross-platform compatibility)
4. Minimal context (token efficiency)

### Design Rules
- Limit each skill to one clear capability or workflow
- Design for progressive disclosure: agents match on `name` and `description` before loading `SKILL.md`
  **Rationale**: Pre-activation metadata consumes token budget
- Write metadata concisely: specify "what it does" + "when to use"
- Align with standard specifications for cross-environment compatibility (VS Code Copilot, GitHub Copilot, Claude Code)

## Directory Placement

- Place repository-scoped skills under `.github/skills/<skill-name>/`
- Use `.claude/skills/<skill-name>/` for Claude-specific clients if needed
- Contain each skill within a single directory
  **Rationale**: Prevents cross-skill dependencies that reduce portability and auditability

**Recommended**:
```text
.github/
  skills/
    api-contract-review/
      SKILL.md
      references/
        checklist.md
      scripts/
        validate-openapi.sh
      assets/
        examples/
          sample-request.json
```

**Not Recommended**:
```text
.github/skills/api-contract-review/
  SKILL.md
../shared-skill-assets/   # External dependency reduces portability
```

## Directory Structure

### Required
- Place `SKILL.md` directly under skill directory

### Optional (only when needed)
- `references/`: Detailed explanations, checklists, procedures, conventions, FAQs
- `scripts/`: Executable scripts or helper code (minimal and secure)
- `assets/`: Templates, examples, sample data (exclude confidential information)
  **Rationale**: Security first - no secrets, credentials, or personal data

### Reference Rules
- Limit references to files within the same skill directory
- Avoid deep nesting of reference paths
  **Rationale**: Reduces read misses and reference errors
- Keep references shallow (e.g., `references/<file>.md`)

## SKILL.md Frontmatter

### Required Structure
- Start YAML frontmatter on line 1 (no blank lines or comments before)
- Use spaces only for indentation (no tabs)
  **Rationale**: Tab characters may cause parsing errors

### Required Fields
- `name`: Skill identifier (use lowercase slug format)
- `description`: Single-line purpose statement (key to discoverability)
  **Rationale**: Multi-line blocks (`|`, `>`) may not be supported across all clients

### Compatibility Notes
- Keep `description` as single line
- Prevent auto-formatting tools (e.g., Prettier) from wrapping `description`

### Optional Fields (use minimally)
- `allowed-tools`: List with least privilege when supported
- `license`, `metadata`, `compatibility`: Add only when operationally necessary

**Recommended**:
```markdown
---
name: api-contract-review
description: Reviews OpenAPI/Swagger diffs for breaking changes. Use when reviewing API specification PRs.
metadata:
  owner: platform-team
  version: '1.0'
---

# API Contract Review

## When to Use
- Reviewing OpenAPI/Swagger change PRs
- Suspected breaking changes affecting client compatibility
```

**Not Recommended**:
```markdown

---
name: API Contract Review  # Spaces/mixed case makes identifier unstable
description: |
  Reviews OpenAPI.
  Use whenever needed.
---

# API Contract Review
```

## Name Field Rules

- Use lowercase slug format with hyphens (e.g., `webapp-testing`, `api-contract-review`)
- Match skill directory name to `name` field
- Avoid overly generic names (e.g., `helper`, `general`, `dev`)
  **Rationale**: Specific names improve discoverability and prevent confusion

## Description Field Rules

- Specify "what it does" + "when to use" in 1-2 sentences
- Include trigger keywords for better agent matching
  **Examples**: PR review, OpenAPI, breaking changes, migration, incident
- Avoid overly broad descriptions (e.g., "helps with everything")
  **Rationale**: Broad descriptions reduce matching precision
- Keep as single line for compatibility

**Recommended**:
```yaml
description: Audits React component accessibility (a11y) and suggests fixes. Use when creating/reviewing UI change PRs.
```

**Not Recommended**:
```yaml
description: Solves all frontend problems. Use anytime.
```

## SKILL.md Body Content

- Write procedures concisely using numbered lists or checklists
- Constrain degrees of freedom appropriately
  **Rationale**: Explicit inputs/outputs/criteria prevent ambiguous hand-offs
- Limit choices to 2-3 options with explicit priorities
  **Rationale**: Too many options (A/B/C/D/E) reduce clarity
- Avoid deep reference chains
  **Not Recommended**: `SKILL.md` → `references/guide.md` → `references/details.md` → ...
- Move long content to reference files with table of contents (keep main body concise with pointers)

## Reference Files Design

- Keep `SKILL.md` body focused on policies, procedures, and deliverable definitions
- Move detailed conventions and long examples to `references/`

### Recommended Reference Content
- Checklists (review criteria, completion conditions)
- Common failures and mitigation strategies
- Deliverable templates (issue/PR comment examples)
- Table of contents (for long documents)

### Referencing Rules
- Specify references explicitly ("Read next" not "Read if needed")
  **Rationale**: Explicit pointers ensure agents access required context

**Recommended**:
```markdown
## Procedure
1. Read `references/checklist.md` to confirm review criteria
2. Determine presence of breaking changes from diff
3. If breaking changes exist, use `assets/examples/migration-note.md` as template for migration steps
```

**Not Recommended**:
```markdown
## Procedure
- Investigate as needed
- Follow references arbitrarily
```

## Scripts and Assets

### Scripts Directory
- Place scripts to support reproducible tasks
- Design skills to function without scripts when tools are unavailable
  **Rationale**: Not all environments support tool execution

### Assets Directory
- Limit to reference-only content (templates, samples)
- Exclude executable or sensitive content

### Referencing Scripts/Assets
- Specify both filename and purpose when referencing
  **Rationale**: Clear context improves usability

**Recommended**:
```markdown
## Execution (if available)
- OpenAPI lint: `scripts/validate-openapi.sh path/to/openapi.yaml`

## Reference
- Breaking change note template: `assets/examples/migration-note.md`
```

**Not Recommended**:
```markdown
- Run the script (Which? Where? Arguments? How to read results?)
```

## Security

### Critical Priority: Security Always First

### Prompt Injection Prevention
- Exclude instruction-overriding phrases from reference files and assets
- Exclude secret information collection attempts
  **Rationale**: Skill content can become prompt injection vectors

### Metadata Security
- Exclude confidential information, PII, tokens, and internal URLs from `description` and `metadata`
  **Rationale**: Metadata may be injected into system prompts
- Keep metadata minimal and safe

### Script Execution Safety
Design with these assumptions:
- Use sandboxing/isolation when available
- Require explicit execution permission (user confirmation)
- Maintain allowlist of trusted skills only
- Log all execution content and results for audit
  **Rationale**: Script execution carries high security risk

### Secrets Management
- Never place credentials, private keys, tokens, or real PII in `assets/` or `references/`
- Use placeholder values for examples (e.g., `YOUR_TOKEN_HERE`)
- Reference environment variables or secret stores instead

**Recommended**:
```markdown
## Security Notes
- Never hardcode credentials in files. Use environment variables or secret stores.
- Require user confirmation for destructive commands (delete/publish/billing).
```

**Not Recommended**:
```markdown
## Tips
- Use token from `assets/prod-token.txt` to call API.
```

## Validation (Always Required)

### Post-Creation Checks
- Verify skill is discoverable:
  - Confirm `name` and `description` are correctly parsed
  - Verify `SKILL.md` starts with frontmatter
  - Verify `description` is single line
- Validate directory with skills specification tools when available (e.g., `skills-ref validate <path>`)
- Confirm reference file paths exist and are not deeply nested

## Final Checklist

### Structure
- [ ] `SKILL.md` line 1 starts with `---`
- [ ] `name` uses lowercase slug format and matches directory name
- [ ] `description` is single line with "what + when" format

### Content
- [ ] Body focuses on procedures/checklists with appropriate constraint
- [ ] Long content moved to `references/` with shallow, explicit references
- [ ] Scripts are minimal and secure (destructive operations require confirmation)

### Security
- [ ] No credentials, PII, or confidential internal data included
- [ ] No prompt injection vectors in content
- [ ] Metadata excludes sensitive information
