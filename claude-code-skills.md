# Claude Code Skills

A practical guide to understanding, creating, and managing custom skills in Claude Code.

## 1. What Are Skills?

Skills are bundles of instructions that extend Claude Code's capabilities. They teach Claude how to perform specific tasks or provide domain-specific knowledge.

**Key Characteristics:**
- Stored as `SKILL.md` files with configuration and instructions
- Follow the open [Agent Skills](https://agentskills.io) standard
- Can be invoked manually or automatically
- Support custom arguments and dynamic context
- Work across multiple AI tools

**Think of skills as:** Custom commands you teach Claude to follow when needed.

---

## 2. Anatomy of a Skill

Every skill has two parts:

### YAML Frontmatter (Configuration)
```yaml
---
name: skill-name
description: When and why to use this skill
optional-field: value
---
```

**Frontmatter defines:**
- When the skill applies
- What tools it can access
- How it can be invoked
- Execution context

### Markdown Content (Instructions)
```markdown
Detailed instructions Claude follows when using the skill.
Can reference files, include examples, and use variables.
```

**Instructions are:**
- Natural language directions
- Code examples and templates
- Links to supporting documentation
- Step-by-step procedures

---

## 3. How to Create a Skill

### Step 1: Create the Directory

```bash
# Project-specific skill (this repository only)
mkdir -p .claude/skills/skill-name

# Personal skill (all your projects)
mkdir -p ~/.claude/skills/skill-name
```

### Step 2: Write SKILL.md

```yaml
---
name: my-skill
description: Clear description of when to use this skill
---

Instructions for Claude to follow...
```

### Step 3: Test the Skill

Manual invocation:
```
/my-skill
```

Or wait for automatic invocation if Claude detects it's relevant.

### Step 4: Support Files (Optional)

Add supporting documentation:
```
.claude/skills/my-skill/
├── SKILL.md           # Main skill definition
├── reference.md       # Detailed documentation
├── examples.md        # Usage examples
└── scripts/
    └── helper.sh      # Executable scripts
```

---

## 4. Skill Configuration Fields

Essential and optional frontmatter fields:

| Field | Required | Purpose |
|-------|----------|---------|
| `name` | Yes | Slash command name (lowercase, hyphens, max 64 chars) |
| `description` | Yes | When to use; Claude reads this for auto-loading |
| `argument-hint` | No | Hint in autocomplete (e.g., `[filename]`) |
| `disable-model-invocation` | No | `true` = only users invoke (default: false) |
| `user-invocable` | No | `false` = only Claude uses internally (default: true) |
| `allowed-tools` | No | Restrict which tools Claude can use |
| `model` | No | Override default model for this skill |
| `context` | No | Set to `fork` to run in isolated subagent |
| `agent` | No | Subagent type: `Explore`, `Plan`, `general-purpose` |

### Important Fields Explained

**`description`** (Most Important)
- Claude reads this to decide whether to auto-load the skill
- Be specific about when the skill applies
- Good: "Database migration patterns for PostgreSQL"
- Bad: "Database stuff"

**`disable-model-invocation: true`**
- Use for side-effect operations you control
- Examples: deploy, commit, push, release
- Prevents Claude from unexpectedly running risky commands
- Users can still invoke with `/skill-name`

**`user-invocable: false`**
- Skill is background knowledge for Claude only
- Users cannot invoke with slash command
- Useful for internal reference material
- Example: project architecture patterns

**`allowed-tools`**
```yaml
allowed-tools: Read, Grep, Glob
```
- Restricts what Claude can do with this skill
- Improves security for sensitive operations
- Examples: deploy skill might only allow Bash; research skill might only allow file reads

---

## 5. How Skills Are Invoked

### Manual Invocation

User types a slash command:
```
/my-skill
/format-commit
/research authentication patterns
```

Claude executes the skill immediately.

### Automatic Invocation

Claude detects when a skill is relevant based on its description:

```
User: How do I create a migration for this database?
Claude: [Automatically loads database-migration skill based on description]
```

Claude decides based on:
- Skill description matching user request
- Skill's `user-invocable` and `disable-model-invocation` flags
- Conversation context

### With Arguments

Pass arguments to skills that accept them:

```
/migrate-component SearchBar React Vue
/summarize-pr 456
/generate-test utils/helpers.ts
```

Arguments are available in the skill as:
- `$ARGUMENTS` - All arguments as a string
- `$0`, `$1`, `$2`, etc. - Individual arguments by position
- `$ARGUMENTS[N]` - Argument by index

---

## 6. Skill Best Practices

### 1. Descriptions Drive Auto-Loading

Make descriptions specific and actionable:

✅ Good:
```yaml
description: Create well-formatted Git commits following conventional commit format
```

❌ Bad:
```yaml
description: Commit helper
```

### 2. Use `disable-model-invocation` for Safety

For operations with side effects, require user invocation:

```yaml
---
name: deploy-production
description: Deploy to production environment
disable-model-invocation: true
---
```

### 3. Keep SKILL.md Focused

Keep instructions under 500 lines:

```yaml
---
name: design-api
description: Design REST API endpoints following project conventions
---

See [API Design Patterns](./api-patterns.md) for detailed guidelines.
See [Examples](./examples.md) for real endpoint designs.
```

Reference supporting files instead of inline content.

### 4. Use Dynamic Context for Live Data

Inject shell command output into the skill:

```yaml
---
name: review-pr
description: Review current pull request
---

Current PR diff:
!`gh pr diff`

PR description and comments:
!`gh pr view --json body,comments`

Recent commits:
!`git log --oneline -n 10`
```

### 5. Be Clear About Tool Restrictions

When restricting tools, explain why:

```yaml
---
name: audit-security
description: Security audit of codebase
allowed-tools: Read, Grep
---

For security reasons, this skill has read-only access.
It analyzes code patterns without modifying files.
```

### 6. Use Subagents for Complex Tasks

For tasks needing deep exploration or planning:

```yaml
---
name: research-architecture
description: Deep research into $ARGUMENTS with full codebase exploration
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly:

1. Find all relevant files
2. Understand relationships
3. Document patterns and decisions
4. Provide detailed findings
```

---

## 7. Practical Skill Examples

### Example 1: Commit Formatting (Manual + Safety)

```yaml
---
name: format-commit
description: Create a well-formatted Git commit with conventional format
disable-model-invocation: true
argument-hint: [commit-message]
---

Create a commit following conventions:

Format: `type(scope): subject`

Types: add, update, fix
- add: New content or feature
- update: Expansion or revision
- fix: Correction or improvement

Examples:
- add(nestjs): authentication patterns guide
- update(claude): expand skills section
- fix(nestjs): clarify middleware ordering
```

### Example 2: API Design Reference (Auto-Load)

```yaml
---
name: api-conventions
description: REST API design conventions and patterns for this project
user-invocable: true
---

When designing REST API endpoints, follow these patterns:

1. **URL Structure**
   - Use kebab-case: `/user-profiles` not `/userProfiles`
   - Resource nouns: `/posts` not `/getPosts`
   - Hierarchical: `/users/123/posts/456`

2. **HTTP Methods**
   - GET: Retrieve resource
   - POST: Create resource
   - PUT: Replace entire resource
   - PATCH: Partial update
   - DELETE: Remove resource

3. **Response Format**
   ```json
   {
     "status": "success|error",
     "data": {},
     "error": null
   }
   ```

4. **Error Responses**
   - Always include status code
   - Provide error message
   - Include error code for programmatic handling

See [API Examples](./examples.md) for real endpoint designs.
```

### Example 3: Codebase Research (Subagent)

```yaml
---
name: codebase-research
description: Deep research into $ARGUMENTS throughout the codebase
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly using available tools:

1. **Find all relevant files**
   - Use Glob to find files by pattern
   - Use Grep to find references

2. **Understand the code**
   - Read key files
   - Trace through implementations
   - Document relationships

3. **Analyze patterns**
   - Identify architectural patterns
   - Find similar implementations
   - Note inconsistencies

4. **Provide findings**
   - List relevant files with line numbers
   - Explain key patterns
   - Suggest improvements if applicable
```

### Example 4: Database Migrations (Tool-Restricted)

```yaml
---
name: db-migrate
description: Create database migrations for PostgreSQL
allowed-tools: Read, Write, Bash(psql migrate)
---

Create a new migration:

1. Name the migration: `YYYY-MM-DD-HH-MM-migration-name.sql`
2. Use standard SQL commands
3. Include rollback migration in the same file
4. Test locally before committing

Example migration:
```sql
-- Up
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Down
DROP TABLE users;
```
```

---

## 8. Skill Locations and Priority

Where skills can be stored:

| Location | Path | Scope | Priority |
|----------|------|-------|----------|
| Enterprise | Admin-configured | Organization | Highest |
| Personal | `~/.claude/skills/<name>/` | All projects | Medium |
| Project | `.claude/skills/<name>/` | This project | Lowest |
| Plugin | Inside plugin | Where enabled | Varies |

**When skills share names:** Enterprise > Personal > Project

---

## 9. String Substitutions

Skills support dynamic variable substitution:

| Variable | Meaning |
|----------|---------|
| `$ARGUMENTS` | All arguments passed to skill |
| `$0`, `$1`, `$2` | First, second, third argument |
| `$ARGUMENTS[N]` | Argument by index (0-based) |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `!`command`` | Shell command output injection |

**Example:**
```yaml
---
name: git-info
---

File status:
!`git status --short`

Recent changes:
!`git log --oneline -n 5`

Current branch: $0
```

---

## 10. Controlling Skill Access

### Restrict Invocation

Only users can invoke (prevents auto-run):
```yaml
disable-model-invocation: true
```

Only Claude can use (not a user command):
```yaml
user-invocable: false
```

### Restrict Tools

Limit what the skill can do:
```yaml
allowed-tools: Read, Grep, Glob
```

### Restrict by Permission

In your Claude Code settings:
```json
{
  "permissions": {
    "deny": ["Skill(deploy)", "Skill(delete)"]
  }
}
```

---

## 11. Common Skill Patterns

### Pattern 1: Task Automation

```yaml
---
name: task-name
description: What it does
disable-model-invocation: true
---

Step-by-step instructions...
```

### Pattern 2: Domain Knowledge

```yaml
---
name: domain-ref
description: Reference for domain concepts
---

Detailed information and patterns...
```

### Pattern 3: Deep Exploration

```yaml
---
name: explore-topic
description: Research $ARGUMENTS
context: fork
agent: Explore
---

Research instructions...
```

### Pattern 4: Secure Operations

```yaml
---
name: secure-task
description: Safe operation with limited tools
allowed-tools: Read, Grep
---

Instructions with restricted capability...
```

---

## 12. Key Takeaways

1. **Skills extend Claude** - They teach new capabilities and domain knowledge
2. **Descriptions matter** - Claude auto-loads based on descriptions; be specific
3. **Manual vs. automatic** - Use `disable-model-invocation` to control when skills run
4. **Safety first** - Restrict tools for sensitive operations
5. **Keep it simple** - One skill per concern; reference external files
6. **Use subagents** - For complex research or planning tasks
7. **Dynamic context** - Inject live data with shell commands
8. **Test thoroughly** - Verify skills work as expected before relying on them
9. **Document well** - Include examples and clear instructions
10. **Update regularly** - Keep skills current as your codebase evolves

---

## Quick Reference

**Creating a skill:**
```bash
mkdir -p .claude/skills/my-skill
# Write .claude/skills/my-skill/SKILL.md
```

**Basic SKILL.md:**
```yaml
---
name: my-skill
description: When to use this skill
---

Instructions here...
```

**Testing:**
```
/my-skill arguments here
```

**Best practices:**
- Specific descriptions for auto-loading
- `disable-model-invocation: true` for side effects
- Keep under 500 lines, reference supporting files
- Use `allowed-tools` to restrict capabilities
- Test before relying on automation

