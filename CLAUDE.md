# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a **notes and learning materials repository** focused on documenting software engineering concepts and best practices. It's not a production application—it's a collection of markdown guides for learning and reference.

Current focus:
- Nest.js core concepts and practical patterns
- Will expand to other frameworks/languages as notes are added

## File Organization

- **Subject-specific markdown files** - Each major topic gets its own file (e.g., `nestjs-core-concepts.md`)
- **CLAUDE.md** - This guidance file
- **Git repository** - Version control for tracking updates to notes

## Writing Guidelines for New Notes

When adding new documentation files:

1. **Follow the 80/20 principle** - Focus on the 20% of concepts that cover 80% of real-world usage
2. **Structure by topic, not chronologically** - Use clear headings and sections
3. **Include practical examples** - Show code examples for each concept
4. **Add decision-making tips** - Include "80/20 Tips" or "Key Takeaways" sections
5. **Keep it scannable** - Use bullet points, bold text, and consistent formatting
6. **Link to detailed guides** - Reference external docs where deeper learning is available

## Naming Conventions

- Markdown files: `<topic>-<subtopic>.md` (kebab-case)
  - Examples: `nestjs-core-concepts.md`, `react-hooks-patterns.md`
- Keep filenames descriptive but concise

## Syncing with Memory

Important concept collections should also be saved to the memory system at `/Users/ricoputrapradana/.claude/projects/-Users-ricoputrapradana-LEARN-notes/memory/` for persistence across conversations:

- Each topic gets a dedicated memory file (e.g., `nestjs.md`)
- Memory files should be concise summaries linked to the full guides in this repo
- Update memory files when notes are significantly expanded or corrected

## Git Workflow

**This is critical:** To prevent losing work, commit and push to GitHub regularly.

### Commit Frequency
- After adding or significantly updating any markdown file
- After creating new memory files
- After expanding existing notes with meaningful content
- Don't wait until the end of a session—commit as you go

### Commit Message Format
Use **conventional commits** for consistency:
```
<type>(<scope>): <subject>

<body>

Co-Authored-By: Claude Haiku 4.5 <noreply@anthropic.com>
```

**Types:**
- `add` - New learning material or note
- `update` - Significant expansion or revision of existing notes
- `fix` - Correction or improvement to existing content

**Examples:**
- `add(react): hooks patterns guide`
- `update(nestjs): expand database integration section`
- `fix(nestjs): clarify middleware execution order`

### Push to GitHub
Always push after committing:
```bash
git push origin main
```

This ensures all work is backed up and accessible from anywhere. Never accumulate unpushed commits.

### Quick Checklist
- [ ] Made changes to markdown or memory files
- [ ] Staged changes: `git add <file>`
- [ ] Committed with conventional commit message
- [ ] Pushed to GitHub: `git push origin main`
- [ ] Verified on https://github.com/ricoputrap/notes
