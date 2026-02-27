# Claude Code Fundamentals

A comprehensive guide to understanding and working with Claude Code, Anthropic's CLI for Claude.

## 1. What is Claude Code?

Claude Code is a command-line interface (CLI) that brings Claude AI capabilities directly to your terminal and code editor. It allows you to work with AI assistance on software engineering tasks without leaving your development environment.

**Key Benefits:**
- Work directly in your codebase with full file access
- Use powerful AI assistance for coding, debugging, and learning
- Keep context about your project without manual explanation
- Use tools like Git, file reading/editing, and shell commands
- Create custom skills to extend capabilities

---

## 2. Core Concepts

### CLAUDE.md - Project Context

A special file that provides guidance to Claude Code instances when working in your repository.

**Purpose:**
- Document project structure and architecture
- Share development conventions and best practices
- Define how to build, test, and deploy
- Provide context that applies to all Claude Code sessions

**Location:** Root of repository (`./CLAUDE.md`)

**What to Include:**
- Repository purpose and high-level architecture
- Build and test commands
- Development workflows and conventions
- Important architectural decisions
- Custom development practices

**What to Skip:**
- Generic best practices (write tests, use proper error handling)
- Obvious instructions
- Component-by-component file listings
- Information easily discovered in code

### Memory System

Persistent storage for learning and patterns across conversations.

**Location:** `~/.claude/projects/<project-path>/memory/`

**Files:**
- `MEMORY.md` - Quick reference (truncated at 200 lines)
- Topic-specific files (e.g., `nestjs.md`, `debugging.md`)

**What to Save:**
- Stable patterns and conventions you've confirmed
- Key architectural decisions and important file paths
- User preferences for workflow and tools
- Solutions to recurring problems
- Important context for future sessions

**What NOT to Save:**
- Session-specific context or temporary state
- Incomplete or unverified information
- Information that duplicates CLAUDE.md
- Speculative conclusions

---

## 3. Tools Available to Claude Code

Claude Code has access to specialized tools for different tasks:

### File Operations
- **Read** - Read file contents (preferred over `cat`)
- **Write** - Create new files
- **Edit** - Make targeted replacements in existing files
- **Glob** - Find files by pattern (preferred over `find`)
- **Grep** - Search file contents (preferred over `grep`/`rg`)

### Version Control
- **Bash** - Execute shell commands including Git operations
- Commit with conventional commit format
- Push to remote repositories
- Manage branches and merges

### Code Exploration & Development
- **Task** - Launch specialized agents for complex work (Explore, Plan, general-purpose)
- **EnterPlanMode** - Get approval for implementation plans before starting
- **ExitPlanMode** - Signal plan completion and request user approval

### Web & Research
- **WebFetch** - Retrieve and analyze web content
- **WebSearch** - Search the web for current information

### Interaction
- **AskUserQuestion** - Ask for clarification or choices during work
- **Skill** - Invoke custom skills

---

## 4. Skills - Extending Claude Code Capabilities

Skills are bundles of instructions that teach Claude new capabilities or domain knowledge.

### Anatomy of a Skill

A skill is a `SKILL.md` file with two parts:

**YAML Frontmatter** (Configuration):
```yaml
---
name: skill-name
description: When and how to use this skill
disable-model-invocation: true  # Only user invokes
allowed-tools: Read, Grep, Bash(*)  # Restricted tools
---
```

**Markdown Content** (Instructions):
```markdown
Detailed instructions Claude follows when using the skill...
```

### How Skills Work

**Three invocation methods:**
1. **Manual** - User types `/skill-name`
2. **Automatic** - Claude detects relevance from description and loads it
3. **With arguments** - `/skill-name arg1 arg2`

### Skill Configuration Options

| Option | Purpose |
|--------|---------|
| `name` | Slash command (lowercase, hyphens, max 64 chars) |
| `description` | When Claude should auto-load; be specific |
| `disable-model-invocation: true` | Only users invoke (good for side effects) |
| `user-invocable: false` | Only Claude uses internally |
| `allowed-tools` | Tools Claude can access without permission |
| `argument-hint` | Hint in autocomplete (e.g., `[issue-number]`) |
| `context: fork` | Run in isolated subagent |
| `agent` | Subagent type: `Explore`, `Plan`, `general-purpose` |

### Creating a Custom Skill

**Step 1:** Create directory
```bash
# Project-specific
mkdir -p .claude/skills/my-skill

# Personal (all projects)
mkdir -p ~/.claude/skills/my-skill
```

**Step 2:** Write `SKILL.md`
```yaml
---
name: my-skill
description: What this skill does and when to use it
---

Instructions for Claude to follow...
```

**Step 3:** Test with `/my-skill`

### Skill Best Practices

1. **Be specific in descriptions** - Claude auto-loads based on descriptions
2. **Use `disable-model-invocation: true`** for operations you control (deploys, commits, etc.)
3. **Keep instructions focused** - Move detailed docs to supporting files
4. **Reference supporting files** - Use markdown links instead of inline content
5. **Use dynamic context** - Inject live data with shell commands: `!`command``
6. **Keep SKILL.md under 500 lines** - Split into multiple files if needed

### Practical Examples

**Task-based skill (manual invocation):**
```yaml
---
name: format-commit
description: Create a well-formatted Git commit with conventional format
disable-model-invocation: true
---

Create commits following conventions:
- Use format: type(scope): description
- Types: add, update, fix
- Keep subject under 50 chars
```

**Reference skill (auto-loads):**
```yaml
---
name: api-design
description: API design patterns and conventions for this project
---

When designing API endpoints:
- Use kebab-case in URLs
- Return consistent error format
- Include proper validation
```

**Research skill with subagent:**
```yaml
---
name: research
description: Deep research into $ARGUMENTS
context: fork
agent: Explore
---

Research $ARGUMENTS thoroughly using Glob, Grep, and Read tools...
```

---

## 5. Request/Response Lifecycle

Understanding how Claude Code processes requests helps debug issues:

```
1. Request received
2. CLAUDE.md loaded (project context)
3. Memory loaded (persistent context)
4. Skills evaluated for auto-loading (based on descriptions)
5. Tools executed in order requested
6. Response generated
7. Optional: Persist findings to memory
```

---

## 6. Tool Usage Guidelines

### General Principles

- **Use dedicated tools when available** - They provide better UX than Bash equivalents
  - Read files with `Read` (not `cat`)
  - Search with `Glob`/`Grep` (not `find`/`grep`)
  - Edit with `Edit` (not `sed`)

- **Prefer independent parallel calls** - When multiple tool calls don't depend on each other, call them together
- **Chain dependent calls sequentially** - If one call must complete before another, run them in order

### When to Use Bash

Reserve Bash for:
- Git commands (commits, pushes, branches)
- System operations (installing packages, checking processes)
- Commands that can't be done with dedicated tools
- Running tests and builds

### Making Changes Safely

1. **Read before editing** - Always use `Read` before using `Edit`
2. **Make targeted changes** - Use `Edit` for replacements, not complete rewrites
3. **Avoid destructive operations** - Don't use `git reset --hard` or `rm -rf` without confirmation
4. **Confirm risky actions** - Ask before force-pushing, deleting branches, or making breaking changes

---

## 7. Git Workflow

Claude Code integrates closely with Git for version control.

### Conventional Commits

Use structured commit messages for clarity:

```
<type>(<scope>): <subject>

<body>

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>
```

**Commit Types** (varies by project):
- For a notes repo: `add`, `update`, `fix`
- For code repos: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`

### Best Practices

- **Commit frequently** - Don't accumulate unpushed commits
- **Push regularly** - Back up work to remote after each commit
- **Write meaningful messages** - Focus on the "why" not the "what"
- **Stage specific files** - Avoid `git add .` which can include sensitive files
- **Never skip hooks** - Let pre-commit hooks run unless you have explicit permission

---

## 8. Working with Models

Claude Code works with different Claude models:

**Latest Models (as of Feb 2026):**
- `claude-opus-4-6` - Most capable (for complex tasks)
- `claude-sonnet-4-6` - Balanced (general purpose)
- `claude-haiku-4-5-20251001` - Fast (quick tasks)

**Usage:**
- Default: Uses the most appropriate model for the task
- Can specify model in skills or tool parameters
- Use faster models for simple research, larger models for complex code

---

## 9. Permission Model

Claude Code respects user permissions for sensitive operations:

### Automatic Operations
- Reading files
- Searching code
- Planning and analysis

### Permission-Required Operations
- Writing/editing files (except auto-approved)
- Running Bash commands
- Using deprecated/risky options
- Accessing authenticated services

### When Denied
- If you deny a tool call, Claude adjusts approach
- Clarify why if the denial wasn't expected
- Check hooks configuration if frequently blocked

---

## 10. Common Workflows

### Adding a New Feature

1. **Plan** - Use `EnterPlanMode` to design approach
2. **Explore** - Use Task tool with Explore agent if needed
3. **Implement** - Write code with frequent commits
4. **Test** - Verify behavior and add tests
5. **Commit** - Final commit with complete feature
6. **Push** - Back up to GitHub

### Debugging an Issue

1. **Read** - Understand the error and context
2. **Search** - Use Grep to find related code
3. **Analyze** - Trace through the logic
4. **Identify** - Pinpoint the root cause
5. **Fix** - Make targeted change
6. **Test** - Verify the fix works
7. **Commit** - Document the fix

### Refactoring Code

1. **Understand** - Read and understand current code thoroughly
2. **Plan** - Use `EnterPlanMode` to design refactoring
3. **Implement** - Make changes incrementally
4. **Test** - Run tests after each significant change
5. **Commit** - Commit refactoring separately from logic changes

---

## 11. Key Takeaways

1. **CLAUDE.md is essential** - It's how you communicate project context to Claude Code
2. **Memory is persistent** - Use it to build on previous work across sessions
3. **Skills extend capabilities** - Create custom skills for repeated workflows
4. **Use dedicated tools** - They're better than Bash equivalents
5. **Commit frequently** - Never lose work; commit and push regularly
6. **Read before editing** - Always understand code before making changes
7. **Ask for help** - Use `AskUserQuestion` when clarification is needed
8. **Plan complex work** - Use `EnterPlanMode` before implementing big changes
9. **Understand the lifecycle** - Know how Claude Code processes requests
10. **Keep context updated** - Update CLAUDE.md and memory as you learn

---

## Resources

- **Official Docs:** https://code.claude.com
- **Agent Skills Standard:** https://agentskills.io
- **This Repository:** Check CLAUDE.md for project-specific guidance
- **Memory System:** Located at `~/.claude/projects/<project-path>/memory/`

