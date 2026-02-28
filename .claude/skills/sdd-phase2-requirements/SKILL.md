---
name: sdd-phase2-requirements
description: Generate detailed Functional and Non-Functional Requirements from Initial Spec for spec-driven development
argument-hint: [spec-file-path]
user-invocable: true
---

# Phase 2: Requirements Generation Skill

Generate comprehensive Functional Requirements (FR) and Non-Functional Requirements (NFR) from an Initial Spec.

## How to Use This Skill

**Invoke manually:**
```
/sdd-phase2-requirements
/sdd-phase2-requirements path/to/initial-spec.md
```

**What AI will do:**
1. Read your Initial Spec (or ask for it)
2. Generate detailed Functional Requirements with validation rules
3. Generate Non-Functional Requirements with measurable targets
4. Define edge cases and error scenarios
5. Create structured requirements document ready for Phase 3

## What Constitutes Good Requirements

Each requirement should be:
- **Testable:** Can you write a test that verifies it? (Yes = good requirement)
- **Specific:** "User can create task with title, description, priority" not "task management works"
- **Complete:** Include validation rules, response format, error cases
- **Independent:** Requirements don't depend heavily on each other
- **Measurable:** Avoid vague words like "fast," "reliable," "easy"

## Output Format

The generated requirements document will include:

### Functional Requirements (What the system does)
```
FR-1: User Registration
- User can register with email and password
- Email must be unique in system
- Password must be 8+ chars, contain uppercase, number, special char
- Returns 201 with user ID and auth token
- Returns 409 if email already registered

FR-2: Create Task
- Authenticated user can create a task
- Task requires: title (max 200 chars), description (optional, max 1000 chars)
- Task optional: priority (High/Medium/Low, default Medium), due_date (ISO 8601)
- Task auto-assigned to creator
- Returns 201 with task object
- Returns 400 if validation fails
- Returns 401 if not authenticated
```

### Non-Functional Requirements (How well it works)
```
NFR-1: Performance
- API response time must be <200ms for 95th percentile
- Database queries must complete in <100ms

NFR-2: Scalability
- System must handle 10,000 concurrent users
- Must support 1,000,000+ records in database

NFR-3: Security
- All passwords must be hashed with bcrypt
- JWT tokens expire in 24 hours
- Sensitive data never exposed in API responses

NFR-4: Reliability
- Zero data loss (ACID transactions for critical operations)
- 99.9% uptime target
- Graceful error handling with meaningful error messages
```

### Edge Cases & Error Scenarios
```
EC-1: Duplicate Email on Registration
- If email already exists, return 409 Conflict
- Message: "Email already registered"

EC-2: Task Assignment to Non-Existent User
- Return 404 Not Found
- Message: "User not found"

EC-3: Updating Someone Else's Task
- If user doesn't own task, return 403 Forbidden
- Message: "You don't have permission to modify this task"
```

## Instructions for Claude

When the user invokes this skill:

1. **Obtain the Initial Spec**
   - If file path provided: Read it
   - If not provided: Ask user to paste spec or provide path
   - Understand: Problem statement, goals, scope, constraints

2. **Generate Functional Requirements (FR)**
   - Extract main features from spec goals and scope
   - For each feature, create FR with:
     - ID (FR-1, FR-2, etc.)
     - Clear description of what user can do
     - Validation rules (min/max length, required fields, formats)
     - HTTP method and endpoint (if applicable)
     - Request/response format
     - HTTP status codes for success and errors
   - Cover all features in the scope
   - Be specific and testable

3. **Generate Non-Functional Requirements (NFR)**
   - Extract from spec constraints (performance, scale, timeline)
   - Define measurable targets for:
     - **Performance:** Response time SLAs, query time budgets
     - **Scalability:** Concurrent user targets, data volume targets
     - **Security:** Password hashing, token expiration, data exposure rules
     - **Reliability:** Data loss prevention, uptime targets, error handling
     - **Compliance:** If applicable (GDPR, HIPAA, etc.)
   - Make each requirement measurable (numbers, not vague words)

4. **Document Edge Cases & Error Scenarios**
   - For each error condition, specify:
     - What triggers it
     - HTTP status code
     - Error message
     - Recovery action (if any)
   - Cover: Missing fields, invalid formats, permission issues, conflicts

5. **Validate Requirements**
   - Each FR is testable (can write a test?)
   - Each NFR is measurable (includes numbers, not vague)
   - All FRs fit within scope
   - Error scenarios are realistic
   - No contradictions between requirements

6. **Provide Output** as a markdown document:
   - Organized by feature/concern
   - Easy to reference and trace
   - Ready to be committed to repo
   - Serves as foundation for Phase 3 (Design Doc)

7. **Ask for Feedback**: "Are these requirements complete? Missing anything? Should we adjust any thresholds?"

## Example Interaction

```
User: /sdd-phase2-requirements

Claude: I'll generate detailed requirements from your Initial Spec.

Please share your Initial Spec (paste the content or provide the file path):

[User shares spec]

Claude: Perfect! Here's your detailed Requirements document:

# Task Management API - Requirements

## Functional Requirements

FR-1: User Registration
- User can register with email and password
- Email must be unique
- Password must be 8+ chars, uppercase, number, special char
- Returns 201 with { id, email, token }
- Returns 409 if email already exists
- Returns 400 if validation fails

[... more FRs ...]

## Non-Functional Requirements

NFR-1: Performance
- Response time: <200ms p95
- Database queries: <100ms p95

[... more NFRs ...]

This requirements document is now ready for Phase 3: Design Doc generation.
Should we proceed, or would you like to adjust anything?
```

## Tips for Best Results

- **Be specific:** "Password 8+ chars with uppercase" not "strong password"
- **Include error cases:** Think about what can go wrong
- **Use measurable targets:** "99.9% uptime" not "very reliable"
- **Group by feature:** All task-related reqs together, all auth reqs together
- **Think testability:** Can you write a test that verifies this requirement?

## Related Skills

After completing requirements:
- Next: `/sdd-phase3-design` - Generate Design Doc
- Previous: `/sdd-init-spec` - Create Initial Spec
- Reference: [Spec-Driven Development Workflow](../../spec-driven-development-workflow.md)
