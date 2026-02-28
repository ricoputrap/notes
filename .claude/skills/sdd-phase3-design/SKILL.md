---
name: sdd-phase3-design
description: Generate Architecture Design Doc with database schema, API design, modules, auth flow, and security decisions
argument-hint: [requirements-file-path]
user-invocable: true
---

# Phase 3: Design Doc Generation Skill

Generate a comprehensive Design Doc that details how you'll implement the requirements.

## How to Use This Skill

**Invoke manually:**
```
/sdd-phase3-design
/sdd-phase3-design path/to/requirements.md
```

**What AI will do:**
1. Read your Requirements (or ask for them)
2. Generate database schema with ER relationships
3. Design API endpoint structure
4. Define module and service architecture
5. Diagram authentication and authorization flows
6. Document error handling and security strategies
7. Provide output ready for Phase 3.5 (Quantitative Analysis)

## What Constitutes Good Architecture

Your design should:
- **Be implementable:** A developer can code from this design
- **Minimize rework:** Catch architectural issues before coding
- **Support scaling:** Identify scaling points for future growth
- **Address security:** Show how you'll protect data and users
- **Be testable:** Clear contracts between components
- **Follow conventions:** Use established patterns (REST, JWT, etc.)

## Output Format

The generated Design Doc will include:

### 1. Database Schema (ER Diagram in Text)
```
Tables:
- users (id, email, password_hash, created_at, updated_at)
- tasks (id, title, description, priority, due_date, status, created_by, assigned_to, created_at, updated_at)
- task_comments (id, task_id, author_id, content, created_at)

Relationships:
- users.id ← tasks.created_by (one-to-many)
- users.id ← tasks.assigned_to (one-to-many)
- users.id ← task_comments.author_id (one-to-many)

Indexes:
- tasks.created_by (for filtering by creator)
- tasks.assigned_to (for filtering by assignee)
- tasks.due_date (for sorting)
```

### 2. API Endpoint Design
```
| Method | Path | Input | Output | Status | Auth |
|--------|------|-------|--------|--------|------|
| POST | /auth/register | email, password | { id, email, token } | 201 | None |
| POST | /auth/login | email, password | { token, expiresIn } | 200 | None |
| GET | /tasks | query filters | [Task] | 200 | JWT |
| POST | /tasks | title, description, priority | Task | 201 | JWT |
| PUT | /tasks/:id | title, description, priority | Task | 200 | JWT |
| DELETE | /tasks/:id | - | empty | 204 | JWT |
```

### 3. Module & Service Architecture
```
AppModule
├── AuthModule
│   ├── auth.controller (register, login, validate)
│   ├── auth.service (createUser, validateUser, createToken)
│   ├── jwt.strategy (validate tokens)
│   └── dto/ (CreateUserDto, LoginDto)
│
├── TasksModule
│   ├── tasks.controller (CRUD endpoints)
│   ├── tasks.service (business logic)
│   ├── tasks.entity (Task entity)
│   └── dto/ (CreateTaskDto, UpdateTaskDto)
│
└── UsersModule
    ├── users.service (findById, findByEmail)
    └── users.entity (User entity)
```

### 4. Authentication Flow
```
Registration:
1. User POST /auth/register { email, password }
2. AuthService hashes password with bcrypt
3. AuthService creates user in database
4. AuthService generates JWT token
5. Return 201 { id, email, token }

Login:
1. User POST /auth/login { email, password }
2. AuthService finds user by email
3. AuthService compares password with bcrypt
4. If match: generate JWT token
5. Return 200 { token, expiresIn }

Protected Request:
1. Client sends Authorization: Bearer <token>
2. JwtStrategy validates token
3. If valid: attach user to request, proceed
4. If invalid: return 401 Unauthorized
```

### 5. Error Handling Strategy
```
Global Exception Filter catches all errors and returns:
{
  "status": 400,
  "message": "Validation failed",
  "errors": [
    { "field": "email", "reason": "Invalid format" }
  ],
  "timestamp": "2024-02-28T10:00:00Z"
}

Status Codes:
- 400: Bad Request (validation)
- 401: Unauthorized (no auth)
- 403: Forbidden (permission denied)
- 404: Not Found
- 409: Conflict (duplicate, etc.)
- 500: Internal Server Error
```

### 6. Security Decisions
```
- Passwords: Bcrypt with 10 salt rounds
- Authentication: JWT with 24-hour expiration
- Authorization: Owner-based (user owns their tasks)
- Input validation: DTOs with class-validator
- CORS: Frontend origin only
- Rate limiting: 100 req/min per IP
```

## Instructions for Claude

When the user invokes this skill:

1. **Obtain the Requirements**
   - If file path provided: Read it
   - If not provided: Ask user to paste or provide path
   - Understand: All FRs, NFRs, and edge cases

2. **Design Database Schema**
   - Identify entities from requirements (users, tasks, etc.)
   - Define columns for each entity with types
   - Establish relationships (one-to-many, many-to-many)
   - Plan indexes for query performance
   - Avoid data duplication (normalization)

3. **Design API Endpoints**
   - Create endpoint table with:
     - HTTP method (GET, POST, PUT, DELETE, PATCH)
     - Path following REST conventions
     - Request fields and validation
     - Response fields and schema
     - HTTP status codes
     - Authentication requirement (JWT, None, etc.)
   - Follow REST patterns (nouns for resources, verbs as HTTP methods)
   - Group related endpoints

4. **Design Module Structure (for NestJS)**
   - Identify feature modules (Auth, Tasks, Users, etc.)
   - For each module:
     - Controller responsibilities
     - Service responsibilities
     - Entity definition
     - DTOs (Create, Update, Response)
   - Show dependency relationships
   - Keep modules cohesive and loosely coupled

5. **Design Authentication Flow**
   - Step-by-step registration process
   - Step-by-step login process
   - Protected request validation process
   - Token generation and validation
   - Expiration and refresh strategy (if applicable)

6. **Design Error Handling**
   - Central exception filter approach
   - Error response format (consistent across API)
   - HTTP status codes for each error type
   - Error messages (user-friendly, not exposing internals)

7. **Document Security Decisions**
   - Password hashing algorithm and parameters
   - Token strategy (JWT: secret, expiration, claims)
   - Authorization approach (RBAC, ownership-based, etc.)
   - Input validation strategy
   - CORS policy
   - Rate limiting rules
   - Data exposure prevention (what not to return)

8. **Validate the Design**
   - Can a developer build from this? (Yes = good)
   - Does it address all requirements? (Check each FR/NFR)
   - Are security concerns handled? (Check each threat)
   - Is there a clear implementation path? (Modules → Services → Tests)
   - Are scaling points identified? (Where might it break?)

9. **Provide Output** as markdown document:
   - Well-organized sections
   - Text diagrams/tables (readable in markdown)
   - Ready to commit to repo
   - Serves as blueprint for Phase 3.5

10. **Ask for Feedback**: "Does this architecture make sense? Any concerns? Ready to move to Phase 3.5 (Quantitative Analysis)?"

## Example Interaction

```
User: /sdd-phase3-design

Claude: I'll generate a Design Doc from your Requirements.

Please share your Requirements (paste or provide file path):

[User shares requirements]

Claude: Great! Here's your Design Doc:

# Task Management API - Design Doc

## 1. Database Schema
[ER diagram and table definitions]

## 2. API Endpoint Design
[Endpoint table with methods, paths, inputs, outputs]

## 3. Module & Service Architecture
[NestJS module structure]

## 4. Authentication Flow
[Step-by-step registration, login, validation]

## 5. Error Handling Strategy
[Exception filter approach, error response format]

## 6. Security Decisions
[Password hashing, JWT configuration, CORS, rate limiting]

This design doc is ready for Phase 3.5: Quantitative Analysis.
Should we proceed with calculating performance metrics and costs?
```

## Tips for Best Results

- **Think like a developer:** Can they build from this? If no, add more detail
- **Identify scaling points:** Where might this break at scale?
- **Keep it consistent:** Same patterns throughout (e.g., all endpoints follow REST)
- **Document tradeoffs:** Why this approach over alternatives?
- **Security first:** Think about data protection, not an afterthought
- **Be specific about tech:** "NestJS with PostgreSQL" not "database and API"

## Related Skills

After completing design:
- Next: `/sdd-phase3.5-quantitative` - Calculate metrics and costs
- Previous: `/sdd-phase2-requirements` - Generate Requirements
- Reference: [Spec-Driven Development Workflow](../../spec-driven-development-workflow.md)
