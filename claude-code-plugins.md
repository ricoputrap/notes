# Claude Code Plugins

A comprehensive guide to understanding, building, and using plugins to extend Claude Code with custom capabilities.

## 1. What Are Plugins?

Plugins are packaged extensions that bundle multiple Claude Code capabilities into reusable, distributable components.

**Key Characteristics:**
- Combine skills, hooks, MCP servers, and LSP servers
- Installed once, work across all projects (or specific projects)
- Versioned and distributed through marketplaces
- Can be enabled/disabled per project or globally
- Encapsulate complex workflows and integrations

**Think of plugins as:** Complete feature packages that extend Claude Code across multiple dimensions.

**Plugins vs. Skills:**
- **Skills** - Single instruction bundle; simple to create; for one specific task
- **Plugins** - Full packages; can include skills, hooks, integrations; distributed/versioned

**Plugins vs. CLAUDE.md:**
- **CLAUDE.md** - Project context; specific to one repository
- **Plugins** - Reusable across projects; shared capabilities

---

## 2. Plugin Architecture

A plugin packages together multiple components:

```
my-plugin/
├── plugin.yaml           # Plugin metadata & configuration
├── skills/               # Custom skills
│   ├── skill-one/
│   │   └── SKILL.md
│   └── skill-two/
│       └── SKILL.md
├── hooks/                # Execution hooks
│   ├── pre-tool.sh
│   └── post-commit.sh
├── mcp-servers/          # Model Context Protocol servers
│   └── custom-server.py
├── lsp-servers/          # Language Server Protocol configs
│   └── language-config.json
├── package.json          # Node.js dependencies
├── pyproject.toml        # Python dependencies
└── README.md             # Plugin documentation
```

### Core Components

**1. Skills**
- Custom instructions and capabilities
- Reusable across projects
- Can be automatic or manual

**2. Hooks**
- Triggers that run at specific events
- Examples: before/after tool execution, on permission denied
- Enable automation and custom workflows

**3. MCP Servers**
- Provide new tools and data sources to Claude
- Examples: database access, file systems, APIs
- Require implementation in Node.js, Python, or Go

**4. LSP Servers**
- Language-specific code intelligence
- Examples: linting, formatting, diagnostics
- Enable language support beyond built-in

---

## 3. Plugin Metadata (plugin.yaml)

Every plugin defines itself in `plugin.yaml`:

```yaml
---
name: my-awesome-plugin
version: 1.0.0
description: A brief description of what this plugin does
author: Your Name
license: MIT

# Where the plugin is active
activation:
  # Global: available in all projects
  scope: global
  # Or: Per project, requires explicit enable
  # scope: project

# Dependencies
dependencies:
  # Plugin dependencies (other plugins this requires)
  - plugin-name@^1.0.0

requirements:
  # Node.js version required
  node: ">=18.0.0"
  # Python version required
  python: ">=3.10"

# What this plugin provides
provides:
  # Custom skills (relative paths)
  skills:
    - skills/format-commit/SKILL.md
    - skills/api-design/SKILL.md

  # Hooks to register
  hooks:
    - hooks/pre-commit.sh
    - hooks/post-tool.sh

  # MCP servers to start
  mcp-servers:
    - name: database-server
      command: python mcp-servers/database.py
      description: Database access and querying

  # LSP servers for languages
  lsp-servers:
    python:
      command: pylsp
      args: ["--plugins", "pylsp_mypy"]

# Configuration options users can customize
config:
  - name: api_style
    description: API design style to enforce
    type: choice
    values: ["rest", "graphql", "grpc"]
    default: rest

  - name: commit_format
    description: Commit message format
    type: text
    default: "conventional"

# What permissions this plugin needs
permissions:
  tools: [Bash, Read, Write]
  commands: ["git", "python"]
  environment: ["PLUGIN_API_KEY"]
```

### Key Fields Explained

**`scope`:**
- `global` - Available in all projects automatically
- `project` - Must be explicitly enabled per project

**`activation`:**
- Controls when/how plugin loads
- Can specify conditions for loading

**`dependencies`:**
- Other plugins this plugin requires
- Version constraints (e.g., `^1.0.0`)

**`requires`:**
- System requirements (Node, Python versions)
- Ensures compatibility

**`provides`:**
- What the plugin offers (skills, hooks, servers)
- Paths to components

**`config`:**
- Customization options for users
- Types: choice, text, boolean, number
- Can be set per-project or globally

**`permissions`:**
- What tools and commands the plugin needs
- Required for security

---

## 4. Building a Simple Plugin

### Step 1: Create Plugin Structure

```bash
mkdir -p my-plugin/skills/commit-formatter
mkdir -p my-plugin/hooks
touch my-plugin/plugin.yaml
```

### Step 2: Create plugin.yaml

```yaml
---
name: commit-formatter
version: 1.0.0
description: Tools for creating well-formatted Git commits
author: Your Name
license: MIT

activation:
  scope: project

provides:
  skills:
    - skills/commit-formatter/SKILL.md

config:
  - name: commit_style
    description: Commit message style
    type: choice
    values: ["conventional", "semantic"]
    default: conventional

permissions:
  tools: [Bash, Read]
  commands: [git]
```

### Step 3: Create a Skill

`skills/commit-formatter/SKILL.md`:

```yaml
---
name: format-commit
description: Create a well-formatted Git commit with your configured style
disable-model-invocation: true
---

Create a Git commit following the configured style.

The current style is: $PLUGIN_CONFIG[commit_style]

For conventional commits:
- Format: type(scope): subject
- Types: add, update, fix, docs
- Example: add(auth): JWT token validation

For semantic commits:
- Format: [component] action
- Example: [auth] add JWT token validation

Follow these guidelines:
1. Use imperative tense
2. Keep subject under 50 characters
3. Wrap body at 72 characters
4. Reference issues: fixes #123
```

### Step 4: Test Locally

```bash
# Install plugin locally
claude-code plugin install ./my-plugin

# List installed plugins
claude-code plugin list

# Enable for a project
cd my-project
claude-code plugin enable commit-formatter
```

### Step 5: Publish

```bash
# Package the plugin
claude-code plugin pack my-plugin

# Publish to marketplace (requires account)
claude-code plugin publish my-plugin-1.0.0.plugin
```

---

## 5. Hooks - Automation Triggers

Hooks are scripts that execute at specific events without user interaction.

### Hook Types

**Tool Execution Hooks:**
```bash
hooks/
├── pre-tool.sh          # Before any tool executes
├── post-tool.sh         # After any tool executes
├── pre-bash.sh          # Before Bash tool
└── post-bash.sh         # After Bash tool
```

**Git Hooks:**
```bash
hooks/
├── pre-commit.sh        # Before commit (can modify staging)
├── post-commit.sh       # After commit succeeds
└── commit-msg.sh        # Validate commit message
```

**Permission Hooks:**
```bash
hooks/
├── on-permission-denied.sh    # When Claude is denied permission
└── on-permission-granted.sh   # When Claude gets permission
```

### Hook Example: Auto-Format on Commit

`hooks/pre-commit.sh`:

```bash
#!/bin/bash
# Auto-format code before committing

echo "📝 Running pre-commit checks..."

# Check for unstaged changes
if git diff --quiet; then
    echo "✅ No changes to stage"
else
    echo "⚠️  Formatting staged files..."

    # Format TypeScript files
    npx prettier --write $(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(ts|tsx)$')

    # Lint
    npx eslint --fix $(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(ts|tsx)$')

    # Re-stage formatted files
    git add $(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(ts|tsx)$')

    echo "✅ Code formatted and re-staged"
fi
```

### Hook Example: Validation

`hooks/commit-msg.sh`:

```bash
#!/bin/bash
# Validate commit message format

COMMIT_MSG=$(cat "$1")

# Check for conventional commit format
if ! echo "$COMMIT_MSG" | grep -qE '^(add|update|fix|docs)\(.*\): .{1,50}'; then
    echo "❌ Invalid commit message format"
    echo "Expected: type(scope): subject (max 50 chars)"
    echo "Types: add, update, fix, docs"
    echo "Got: $COMMIT_MSG"
    exit 1
fi

echo "✅ Commit message format valid"
exit 0
```

### Hook Context

Hooks receive context via environment variables:

```bash
$CLAUDE_SESSION_ID       # Current session ID
$CLAUDE_PROJECT_ROOT     # Project root directory
$CLAUDE_TOOL_NAME        # Name of tool being run
$CLAUDE_TOOL_RESULT      # Result of tool execution
$CLAUDE_PERMISSION_DENIED # true if permission was denied
```

### Hook Best Practices

1. **Keep hooks fast** - They block tool execution
2. **Provide clear output** - Tell user what happened
3. **Exit with proper codes** - 0 for success, non-zero for failure
4. **Handle errors gracefully** - Don't crash Claude
5. **Log important events** - Help debug issues
6. **Make them optional** - Let users disable if needed

---

## 6. MCP Servers - Extending Tool Capabilities

MCP (Model Context Protocol) servers provide new tools and data sources to Claude.

### When to Use MCP Servers

- Access specialized databases
- Connect to proprietary systems
- Provide custom data sources
- Implement language-specific tools
- Enable integrations with external services

### Building an MCP Server

**Node.js Example:**

```typescript
// mcp-servers/database.ts
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { CallToolRequestSchema, ListToolsRequestSchema } from "@modelcontextprotocol/sdk/types.js";

class DatabaseServer {
  private server: Server;

  constructor() {
    this.server = new Server({
      name: "database-server",
      version: "1.0.0",
    });

    // Register handlers
    this.server.setRequestHandler(
      ListToolsRequestSchema,
      this.handleListTools.bind(this)
    );

    this.server.setRequestHandler(
      CallToolRequestSchema,
      this.handleCallTool.bind(this)
    );
  }

  private handleListTools() {
    return {
      tools: [
        {
          name: "query-database",
          description: "Execute SQL query against database",
          inputSchema: {
            type: "object",
            properties: {
              sql: {
                type: "string",
                description: "SQL query to execute",
              },
            },
            required: ["sql"],
          },
        },
        {
          name: "get-schema",
          description: "Get database schema information",
          inputSchema: {
            type: "object",
            properties: {
              table: {
                type: "string",
                description: "Table name (optional)",
              },
            },
          },
        },
      ],
    };
  }

  private async handleCallTool(request: any) {
    const { name, arguments: args } = request;

    if (name === "query-database") {
      // Execute query
      return {
        content: [
          {
            type: "text",
            text: "Query results...",
          },
        ],
      };
    }

    if (name === "get-schema") {
      // Return schema info
      return {
        content: [
          {
            type: "text",
            text: "Schema information...",
          },
        ],
      };
    }

    throw new Error(`Unknown tool: ${name}`);
  }

  async run() {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
  }
}

// Start server
const server = new DatabaseServer();
server.run();
```

**Python Example:**

```python
# mcp-servers/database.py
from mcp.server import Server, Tool
from mcp.types import TextContent
import sqlite3

class DatabaseServer:
    def __init__(self):
        self.server = Server("database-server")

        # Register tools
        self.server.register_tool(
            Tool(
                name="query",
                description="Execute SQL query",
                inputSchema={
                    "type": "object",
                    "properties": {
                        "sql": {
                            "type": "string",
                            "description": "SQL query"
                        }
                    },
                    "required": ["sql"]
                },
                handler=self.handle_query
            )
        )

    async def handle_query(self, sql: str):
        try:
            conn = sqlite3.connect("app.db")
            cursor = conn.cursor()
            cursor.execute(sql)
            results = cursor.fetchall()
            conn.close()

            return TextContent(text=str(results))
        except Exception as e:
            return TextContent(text=f"Error: {str(e)}")

    async def run(self):
        await self.server.run()

if __name__ == "__main__":
    import asyncio
    server = DatabaseServer()
    asyncio.run(server.run())
```

### MCP Server Best Practices

1. **Secure by default** - Validate all inputs
2. **Limit operations** - Only expose necessary tools
3. **Handle errors gracefully** - Return clear error messages
4. **Document tools** - Clear descriptions and parameters
5. **Test thoroughly** - Verify all edge cases
6. **Monitor performance** - Keep response times fast

---

## 7. LSP Servers - Language Intelligence

LSP (Language Server Protocol) servers provide language-specific features like linting, formatting, and diagnostics.

### Configuring LSP Servers

In `plugin.yaml`:

```yaml
provides:
  lsp-servers:
    python:
      command: pylsp
      args: ["--plugins", "pylsp_mypy,pylsp_black"]

    typescript:
      command: typescript-language-server
      args: ["--stdio"]

    go:
      command: gopls
      settings:
        formatOnSave: true
        gofumpt: true
```

### Using Existing LSP Servers

**Python (pylsp):**
```yaml
python:
  command: pylsp
  args: [
    "--plugins",
    "pylsp_mypy",      # Type checking
    "pylsp_black",     # Code formatting
    "pylsp_isort",     # Import sorting
    "pylsp_flake8"     # Linting
  ]
```

**TypeScript (tsserver):**
```yaml
typescript:
  command: typescript-language-server
  args: ["--stdio"]
```

**Go (gopls):**
```yaml
go:
  command: gopls
  args: []
```

---

## 8. Plugin Distribution

### Marketplace

Plugins can be published to the Claude Code Plugin Marketplace:

1. **Create account** at https://plugins.claude.dev
2. **Package plugin** - `claude-code plugin pack my-plugin`
3. **Submit for review** - Review process takes 24-48 hours
4. **Publish** - Once approved, available to all users

### Plugin Versioning

Use semantic versioning:
- **MAJOR.MINOR.PATCH** (e.g., 1.2.3)
- MAJOR: Breaking changes
- MINOR: New features, backward compatible
- PATCH: Bug fixes

```yaml
version: 1.0.0  # Initial release
version: 1.1.0  # New skill added
version: 2.0.0  # Breaking changes
```

### Plugin Discovery

Users find plugins via:
- Claude Code command: `claude-code plugin search`
- Web marketplace: https://plugins.claude.dev
- GitHub (community plugins)

---

## 9. Practical Plugin Example: Node.js Project Helper

Complete plugin that helps with Node.js projects:

`plugin.yaml`:
```yaml
---
name: nodejs-helper
version: 1.0.0
description: Tools for Node.js/TypeScript development
author: Your Name
license: MIT

activation:
  scope: project

provides:
  skills:
    - skills/npm-manage/SKILL.md
    - skills/typescript-debug/SKILL.md

  hooks:
    - hooks/pre-commit.sh

  mcp-servers:
    - name: npm-server
      command: node mcp-servers/npm.js

config:
  - name: typescript_strict
    description: Use TypeScript strict mode
    type: boolean
    default: true

  - name: formatter
    description: Code formatter
    type: choice
    values: ["prettier", "eslint"]
    default: prettier

permissions:
  tools: [Bash, Read, Write]
  commands: ["npm", "npm-check", "typescript"]
```

`skills/npm-manage/SKILL.md`:
```yaml
---
name: npm-manage
description: Manage Node.js dependencies and scripts
---

Commands to manage this Node.js project:

1. **List outdated packages**
   npm outdated

2. **Update packages safely**
   npm update
   npm audit fix

3. **Check for vulnerabilities**
   npm audit

4. **Add new dependency**
   npm install <package-name>

5. **Add dev dependency**
   npm install --save-dev <package-name>

6. **Update package to latest**
   npm install <package-name>@latest

7. **Remove dependency**
   npm uninstall <package-name>
```

---

## 10. Plugin Marketplace Best Practices

### Plugin Naming
- **Good:** `nodejs-helper`, `commit-formatter`, `database-tools`
- **Bad:** `my-plugin`, `tool`, `stuff`

### Documentation
```markdown
# NodeJS Helper Plugin

Helper plugin for Node.js and TypeScript development.

## Features
- Dependency management commands
- TypeScript debugging helpers
- Pre-commit hooks for code quality

## Installation
1. Search for "nodejs-helper" in Claude Code
2. Click Install
3. Enable for your project

## Configuration
- `typescript_strict`: Enable TypeScript strict mode
- `formatter`: Choose code formatter

## Usage
- Use `/npm-manage` for dependency commands
- Use `/typescript-debug` for debugging help
```

### Icon and Branding
- Include `icon.png` (256x256 or 512x512)
- Use consistent branding
- Include screenshots of key features

### Testing
- Test plugin installation
- Verify all skills work
- Check hooks execute correctly
- Test MCP server connections
- Validate LSP server functionality

---

## 11. Plugin Lifecycle

### Installation
```bash
# Install from marketplace
claude-code plugin install nodejs-helper

# Or from GitHub
claude-code plugin install github:username/plugin-name

# Or from local directory
claude-code plugin install ./my-plugin
```

### Enabling/Disabling
```bash
# Enable for current project
claude-code plugin enable nodejs-helper

# Disable for current project
claude-code plugin disable nodejs-helper

# List enabled plugins
claude-code plugin list
```

### Configuration
```bash
# Set plugin configuration
claude-code plugin config nodejs-helper typescript_strict=true
claude-code plugin config nodejs-helper formatter=prettier
```

### Updates
```bash
# Check for updates
claude-code plugin updates

# Update specific plugin
claude-code plugin update nodejs-helper

# Update all plugins
claude-code plugin update --all
```

### Uninstallation
```bash
# Disable first (optional)
claude-code plugin disable nodejs-helper

# Uninstall
claude-code plugin uninstall nodejs-helper
```

---

## 12. Plugin vs. Other Approaches

| Approach | Best For | Complexity | Scope |
|----------|----------|-----------|-------|
| **Skill** | Single task | Low | One thing |
| **CLAUDE.md** | Project context | Low | One project |
| **Plugin** | Complete feature | High | Multiple projects |
| **MCP Server** | New tools | High | All projects |
| **Hook** | Automation | Medium | All projects |

**Decision Tree:**
- Need to teach Claude one task? → **Skill**
- Need project-specific guidance? → **CLAUDE.md**
- Building reusable package for many projects? → **Plugin**
- Need new capabilities across all projects? → **Plugin with MCP**
- Need to automate specific events? → **Hook**

---

## 13. Troubleshooting Plugins

### Plugin Not Loading
```bash
# Check plugin is enabled
claude-code plugin list

# Enable it
claude-code plugin enable plugin-name

# Check for errors
claude-code plugin logs plugin-name
```

### MCP Server Connection Failed
```bash
# Verify server executable exists
which python  # or node

# Test server directly
python mcp-servers/server.py

# Check permissions
ls -la mcp-servers/
chmod +x mcp-servers/server.py
```

### Hooks Not Running
```bash
# Verify hook files exist
ls -la hooks/

# Make executable
chmod +x hooks/*.sh

# Test hook directly
bash hooks/pre-commit.sh

# Check logs
claude-code plugin logs --hook=pre-commit
```

### Permission Issues
```bash
# Check plugin permissions
claude-code plugin permissions plugin-name

# Grant permissions
claude-code plugin grant plugin-name Bash
```

---

## 14. Key Takeaways

1. **Plugins package complete features** - Skills + hooks + servers in one
2. **Reusable across projects** - Install once, use everywhere
3. **Multiple components** - Combine skills, hooks, MCP, LSP servers
4. **Configuration support** - Users customize per project
5. **Marketplace distribution** - Publish and share with community
6. **Hooks enable automation** - Run code at specific events
7. **MCP servers add tools** - Extend Claude's capabilities
8. **LSP servers add intelligence** - Language-specific features
9. **Security by design** - Declare permissions upfront
10. **Versioning matters** - Semantic versioning for compatibility

---

## 15. Quick Reference

### Create Plugin
```bash
mkdir -p my-plugin/skills
touch my-plugin/plugin.yaml
```

### Minimal plugin.yaml
```yaml
---
name: my-plugin
version: 1.0.0
description: What it does
author: Your Name

provides:
  skills:
    - skills/skill/SKILL.md

permissions:
  tools: [Read]
```

### Install & Test
```bash
claude-code plugin install ./my-plugin
claude-code plugin enable my-plugin
claude-code plugin logs my-plugin
```

### Publish
```bash
claude-code plugin pack my-plugin
claude-code plugin publish my-plugin-1.0.0.plugin
```

### Common Plugin Types

1. **Task Helper** - Skills + hooks
2. **Language Support** - LSP + MCP server
3. **System Integration** - MCP servers for APIs
4. **Automation Pack** - Multiple skills + hooks
5. **Validation Tools** - Hooks + error checking

