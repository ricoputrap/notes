---
name: sdd-init-spec
description: Generate an Initial Spec for spec-driven development workflow - define problem statement, goals, scope, and constraints
argument-hint: [project-name]
user-invocable: true
---

# Initial Spec Generation Skill

Generate a comprehensive Initial Spec document for the Spec-Driven Development (SDD) workflow.

## How to Use This Skill

**Invoke manually:**
```
/sdd-init-spec "Task Management API"
/sdd-init-spec "User Authentication System"
```

**What AI will do:**
1. Ask clarifying questions about your project
2. Generate a complete Initial Spec with problem statement, goals, scope, and constraints
3. Provide output as a structured markdown document ready for Phase 1

## What Constitutes a Good Initial Spec

Your spec should answer these questions:

### 1. **Problem Statement**
- What pain point or problem exists?
- Who has this problem?
- Why is it worth solving?
- What happens if you don't solve it?

### 2. **Project Goals**
- What will this software achieve?
- List 3-5 main goals
- Define success criteria (measurable, not vague)
- Example: "3+ team members using daily" not "successful deployment"

### 3. **Scope & Constraints**
- **In-Scope:** What features are explicitly included?
- **Out-of-Scope:** What features are explicitly NOT included?
- **Technical Constraints:** Performance, scale, tech stack, third-party services
- **Business Constraints:** Timeline, budget, team size
- **Non-Functional Requirements:** Uptime target, data retention, compliance needs

### 4. **Key Stakeholders**
- End users: Who will use this system?
- Team members: Who builds it?
- Decision makers: Who approves it?
- Operators: Who maintains it in production?

## Output Format

The generated spec will follow this structure:

```markdown
# [Project Name] - Initial Spec

## Problem Statement
[Clear description of the problem and why it matters]

## Project Goals
- Goal 1: ...
- Goal 2: ...
- Goal 3: ...

Success Criteria: [Measurable outcomes]

## Scope

### In-Scope
- Feature 1
- Feature 2
- Feature 3

### Out-of-Scope
- Non-feature 1
- Non-feature 2

## Constraints

### Technical Constraints
- Scale: ...
- Performance: ...
- Tech Stack: ...
- Integrations: ...

### Business Constraints
- Timeline: ...
- Budget: ...
- Team Size: ...

## Key Stakeholders
- End Users: ...
- Development Team: ...
- Decision Makers: ...
- Operations: ...

## Next Steps
→ Phase 2: Generate detailed Requirements from this spec
```

## Instructions for Claude

When the user invokes this skill:

1. **Extract project name** from `$ARGUMENTS` (or ask if not provided)

2. **Ask clarifying questions** to understand:
   - What problem does the project solve?
   - Who are the end users?
   - What are the main features/goals?
   - Timeline and scale expectations?
   - Any known constraints (budget, tech stack, performance)?

3. **Generate the Initial Spec** based on answers:
   - Write clear, concise problem statement
   - Define measurable success criteria (not vague)
   - List specific in-scope and out-of-scope features
   - Document technical constraints (scale, performance, tech stack)
   - Document business constraints (timeline, budget, team)
   - Identify key stakeholders

4. **Validate the spec**:
   - Problem statement is clear and compelling
   - Goals are measurable
   - Scope is realistic for the timeline
   - Constraints are documented
   - Stakeholders are identified

5. **Provide the output** as a markdown document that:
   - Follows the structure above
   - Is 1-3 pages long
   - Can be directly committed to the repo
   - Serves as the foundation for Phase 2 (Requirements)

6. **Ask for feedback**: "Does this spec accurately capture what you're building? Any adjustments needed?"

## Example Interaction

```
User: /sdd-init-spec "Task Management API"

Claude: I'll help you create an Initial Spec for your Task Management API.

Let me ask a few clarifying questions:
1. What's the main problem this API solves? (e.g., team coordination, deadline tracking)
2. Who are the primary users? (e.g., small teams, enterprises)
3. What's your timeline and expected scale? (e.g., 100 users in 3 months)
4. Any tech stack preferences or constraints?
5. What's your availability/budget? (e.g., self-hosted, cloud, cost constraints)

[User provides answers]

Claude: Great! Here's your Initial Spec:

# Task Management API - Initial Spec
[Full spec document]

This spec is ready for Phase 2: Generate detailed Functional and Non-Functional Requirements.
Would you like me to proceed to Phase 2, or refine anything in this spec?
```

## Tips for Best Results

- **Be specific about scale:** "1,000 users" not "some users"
- **Define success measurably:** "3+ team members daily" not "successful"
- **Know your constraints:** Budget, timeline, team size matter
- **Don't over-scope:** Start with core features, add later
- **Document assumptions:** Why did you choose this tech stack? Why this timeline?

## Related Skills

After completing this spec:
- Next: `/sdd-phase2-requirements` - Generate detailed requirements
- Or: Read [Spec-Driven Development Workflow](../../spec-driven-development-workflow.md) for more context
