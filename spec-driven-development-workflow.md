# Spec-Driven Development Workflow: 7 Phases

A complete guide to building software systematically using specs as your source of truth, with AI assistance at each phase.

---

## Overview: Why This Workflow?

**Traditional approach:** Start coding → Fix bugs → Refactor → Deploy (chaotic, expensive)

**Spec-Driven approach:** Plan → Spec → Design → Code → Test → Deploy (predictable, quality)

**AI's role:** Accelerate each phase without sacrificing clarity—AI generates drafts, you review and refine.

---

## Phase 1: Initial Spec

### 🎯 Purpose
Define **what problem you're solving** and **why it matters**. This is your North Star.

### 📝 What to Include

**1. Problem Statement**
- What pain point exists today?
- Who has this problem?
- Why is it worth solving?

**2. Project Goals**
- What will this software achieve?
- Success criteria (how do you know it's done?)
- Main features at a high level

**3. Scope & Constraints**
- What's included? (in-scope)
- What's explicitly NOT included? (out-of-scope)
- Technical constraints (performance, scale, tech stack)
- Timeline/budget constraints

**4. Key Stakeholders**
- Who's using this? (end users)
- Who's building this? (team)
- Who's approving? (decision makers)

### 📋 Example Spec (Task Management API)

```markdown
# Task Management API - Initial Spec

## Problem Statement
Teams struggle to track who's responsible for what and when tasks are due.
Manual spreadsheets and email chains lead to missed deadlines and unclear ownership.

## Project Goals
Build a lightweight REST API that allows teams to:
- Create, assign, and track tasks
- See who's responsible for what
- Know which tasks are overdue
Success Criteria: 3+ team members using it daily, zero data loss

## Scope (In-Scope)
- User accounts (sign up, login)
- Task CRUD (create, assign, complete)
- Task filtering (by status, assignee, due date)
- Basic notifications (email when assigned)

## Out-of-Scope
- Mobile app (web API only)
- AI-powered task suggestions
- Advanced reporting/analytics
- Multi-organization support

## Constraints
- Must handle 1,000 concurrent users
- Response time < 200ms for 95th percentile
- Tech stack: NestJS, PostgreSQL, TypeScript
- Timeline: 4 weeks
```

### 🤖 How AI Helps

**Prompt to AI:**
```
I'm building a task management system for small teams. Users can create
tasks, assign them to team members, set due dates, and mark them complete.

Generate an Initial Spec including:
- Problem statement
- Project goals & success criteria
- Scope (in/out)
- Key constraints

Keep it concise, 1-2 pages max.
```

**What to do with AI output:**
- Use as a starting template
- Review and refine based on your actual requirements
- Make sure scope is realistic for your timeline
- Get team alignment on the spec before moving forward

### ✅ Definition of Done for Phase 1
- [ ] Problem statement is clear
- [ ] Goals are measurable
- [ ] Scope is defined (in/out)
- [ ] Constraints are documented
- [ ] Team agrees this is the right problem to solve

---

## Phase 2: Requirements

### 🎯 Purpose
Translate the spec into **detailed, testable requirements**. This is what developers will build against.

### 📝 What to Include

**Functional Requirements (FR)** - What the system does
```
FR-1: User Registration
- User can register with email and password
- Password must be 8+ chars, contain uppercase, number, special char
- Email must be unique in system
- Returns 201 with user ID and auth token

FR-2: Create Task
- Authenticated user can create a task
- Task requires: title (string, max 200 chars), description (optional, max 1000 chars)
- Task optional: priority (High/Medium/Low, default Medium), due_date (ISO 8601)
- Task auto-assigned to creator
- Returns 201 with task object including id, createdAt, createdBy
```

**Non-Functional Requirements (NFR)** - How well the system works
```
NFR-1: Performance
- API response time must be < 200ms for 95th percentile
- Database queries must complete in < 100ms

NFR-2: Scalability
- System must handle 1,000 concurrent users
- Must support 10,000+ tasks in database

NFR-3: Security
- All passwords must be hashed with bcrypt
- JWT tokens expire in 24 hours
- Sensitive data (passwords) never exposed in responses

NFR-4: Reliability
- Zero data loss (ACID transactions)
- 99.9% uptime target
- Graceful error handling
```

**Edge Cases & Error Scenarios**
```
EC-1: Duplicate email on registration
- If email already exists, return 409 Conflict
- Message: "Email already registered"

EC-2: Task assignment to non-existent user
- Return 404 Not Found
- Message: "User not found"

EC-3: Updating someone else's task
- If user doesn't own task, return 403 Forbidden
- Message: "You don't have permission to modify this task"
```

### 🤖 How AI Helps

**Prompt to AI:**
```
Based on this spec:
[Paste Initial Spec here]

Generate detailed Functional Requirements covering:
- User registration & authentication
- Task creation & management
- Task assignment & filtering

For each requirement, include:
- ID (FR-1, FR-2, etc.)
- Description
- Validation rules
- Response format
- Error cases

Use this format:
FR-X: [Feature Name]
- User can [action]
- [Field] must/should [validation]
- Returns [status code] with [data]
```

**What to do with AI output:**
- Review for completeness (did AI miss anything?)
- Validate against your actual needs
- Add edge cases AI might have missed
- Make requirements specific and testable (avoid vague language like "fast" or "reliable")

### ✅ Definition of Done for Phase 2
- [ ] All functional requirements documented (FR-1 through FR-N)
- [ ] Non-functional requirements specified (performance, security, scalability)
- [ ] Edge cases and error scenarios covered
- [ ] Each requirement is testable (can write a test that passes/fails)
- [ ] Team reviewed and approved requirements

---

## Phase 3: Design Doc

### 🎯 Purpose
**Architect** how you'll implement the requirements. This prevents "let's refactor everything" later.

### 📝 What to Include

**1. Database Design (Entity-Relationship Diagram)**
```
Tables needed:
- users (id, email, password_hash, created_at, updated_at)
- tasks (id, title, description, priority, due_date, status, created_by, assigned_to, created_at, updated_at)
- task_comments (id, task_id, author_id, content, created_at)

Relationships:
- users.id ← tasks.created_by (one-to-many)
- users.id ← tasks.assigned_to (one-to-many)
- users.id ← task_comments.author_id (one-to-many)
- tasks.id ← task_comments.task_id (one-to-many)

Indexes:
- tasks.created_by (for filtering)
- tasks.assigned_to (for filtering)
- tasks.due_date (for sorting)
- tasks.status (for filtering)
```

**2. API Endpoint Design**
```
Endpoint Table:

| Method | Path | Input | Output | Status | Auth |
|--------|------|-------|--------|--------|------|
| POST | /auth/register | email, password | { id, email, token } | 201 | None |
| POST | /auth/login | email, password | { token, expiresIn } | 200 | None |
| GET | /tasks | query: status, assignee, due_date | [{ id, title, ... }] | 200 | JWT |
| POST | /tasks | title, description, priority, due_date | { id, title, ... } | 201 | JWT |
| GET | /tasks/:id | - | { id, title, ... } | 200 | JWT |
| PUT | /tasks/:id | title, description, priority, due_date | { id, title, ... } | 200 | JWT |
| DELETE | /tasks/:id | - | empty | 204 | JWT |
| PATCH | /tasks/:id/assign | assignee_id | { id, assigned_to, ... } | 200 | JWT |
```

**3. Module & Service Architecture**
```
Modules:
├── AuthModule
│   ├── auth.controller.ts (register, login)
│   ├── auth.service.ts (validateUser, createToken)
│   ├── jwt.strategy.ts (JWT validation)
│   └── dto/ (CreateUserDto, LoginDto)
│
├── TasksModule
│   ├── tasks.controller.ts (CRUD endpoints)
│   ├── tasks.service.ts (business logic)
│   ├── tasks.entity.ts (Task, Comment entities)
│   └── dto/ (CreateTaskDto, UpdateTaskDto)
│
└── UsersModule
    ├── users.service.ts (findById, findByEmail)
    └── users.entity.ts (User entity)
```

**4. Authentication Flow**
```
1. User POST /auth/register { email, password }
2. Service hashes password with bcrypt
3. Service creates user in DB
4. Service generates JWT token
5. Return token to client

6. Client POST /auth/login { email, password }
7. Service finds user by email
8. Service compares password with bcrypt
9. If match, generate new JWT token
10. Return token to client

11. Client includes token in Authorization header: "Bearer <token>"
12. JwtStrategy validates token
13. If valid, attach user to request
14. If invalid, return 401 Unauthorized
```

**5. Error Handling Strategy**
```
- All errors caught by global exception filter
- Return consistent error format:
  {
    "status": 400,
    "message": "Validation failed",
    "errors": [
      { "field": "email", "reason": "Invalid email format" }
    ],
    "timestamp": "2024-02-28T10:00:00Z"
  }

- HTTP status codes:
  - 400: Bad Request (validation errors)
  - 401: Unauthorized (not logged in)
  - 403: Forbidden (not permitted)
  - 404: Not Found
  - 409: Conflict (duplicate email, etc.)
  - 500: Internal Server Error
```

**6. Security Decisions**
```
- Passwords: Bcrypt with salt rounds 10
- Authentication: JWT with 24-hour expiration
- Authorization: Owner-based (user can only modify own tasks)
- Input validation: DTOs with class-validator
- CORS: Allow only frontend origin
- Rate limiting: 100 requests per minute per IP
```

**7. Database Migration Strategy**
```
- Use TypeORM migrations
- Migration per feature:
  - 001_create_users_table.ts
  - 002_create_tasks_table.ts
  - 003_create_task_comments_table.ts
- Migrations run on deploy
- Rollback capability for each migration
```

### 🤖 How AI Helps

**Prompt to AI:**
```
Based on these requirements:
[Paste FR-1 through FR-N here]

Generate a Design Doc including:

1. Database Schema
   - Table definitions with columns and types
   - Relationships between tables
   - Indexes needed for performance

2. API Endpoint Design
   - Table with: Method, Path, Input, Output, Status, Auth Required

3. Module Structure
   - NestJS modules and their responsibilities
   - Which services live in which modules

4. Authentication Flow
   - Step-by-step flow for register, login, and request validation

5. Error Handling
   - Error response format
   - HTTP status codes and meanings

6. Security Architecture
   - Password hashing strategy
   - JWT token management
   - Authorization approach

Keep it concise, focus on implementation decisions that matter.
```

**What to do with AI output:**
- Review database schema for normalization (no data duplication)
- Check API design follows REST conventions
- Verify authentication flow is secure
- Make sure error handling is consistent
- Adjust based on your specific tech choices
- Add any custom requirements (e.g., soft deletes, audit logging)

### ✅ Definition of Done for Phase 3
- [ ] Database schema defined (tables, relationships, indexes)
- [ ] All API endpoints specified in table format
- [ ] Module structure and responsibility clear
- [ ] Authentication/authorization strategy documented
- [ ] Error handling approach defined
- [ ] Security decisions documented
- [ ] Tech lead/architect reviewed and approved

---

## Phase 4: Swagger Doc

### 🎯 Purpose
**Formalize the API contract** before coding. This is what clients will integrate against.

### 📝 What to Include

**Complete OpenAPI/Swagger specification:**
```yaml
openapi: 3.0.0
info:
  title: Task Management API
  version: 1.0.0

paths:
  /auth/register:
    post:
      summary: Register a new user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                email:
                  type: string
                  format: email
                password:
                  type: string
                  minLength: 8
      responses:
        '201':
          description: User created successfully
          content:
            application/json:
              schema:
                type: object
                properties:
                  id:
                    type: integer
                  email:
                    type: string
                  token:
                    type: string
        '400':
          description: Validation failed
        '409':
          description: Email already registered

  /tasks:
    get:
      summary: List all tasks
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, completed, overdue]
        - name: assignee_id
          in: query
          schema:
            type: integer
      responses:
        '200':
          description: Task list
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Task'

    post:
      summary: Create a new task
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateTaskRequest'
      responses:
        '201':
          description: Task created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Task'

components:
  schemas:
    Task:
      type: object
      properties:
        id:
          type: integer
        title:
          type: string
        description:
          type: string
        priority:
          type: string
          enum: [high, medium, low]
        status:
          type: string
          enum: [pending, completed, overdue]
        due_date:
          type: string
          format: date-time
        created_by:
          type: integer
        assigned_to:
          type: integer
        created_at:
          type: string
          format: date-time
```

**In NestJS with decorators:**
```typescript
@Controller('tasks')
export class TasksController {
  @Get()
  @ApiOperation({ summary: 'List all tasks' })
  @ApiQuery({ name: 'status', required: false, enum: ['pending', 'completed'] })
  @ApiResponse({ status: 200, type: [TaskResponseDto] })
  findAll(@Query('status') status?: string) {
    // Implementation
  }

  @Post()
  @ApiOperation({ summary: 'Create a new task' })
  @ApiBody({ type: CreateTaskDto })
  @ApiResponse({ status: 201, type: TaskResponseDto })
  @ApiResponse({ status: 400, description: 'Validation failed' })
  create(@Body() createTaskDto: CreateTaskDto) {
    // Implementation
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get task by ID' })
  @ApiParam({ name: 'id', type: 'number' })
  @ApiResponse({ status: 200, type: TaskResponseDto })
  @ApiResponse({ status: 404, description: 'Task not found' })
  findOne(@Param('id') id: number) {
    // Implementation
  }
}
```

### 🤖 How AI Helps

**Prompt to AI:**
```
Based on this API Design:
[Paste API endpoint table here]

Generate a complete Swagger/OpenAPI 3.0 specification including:

1. All endpoints (path, method, parameters, request/response)
2. Request/response schemas
3. Error responses (400, 401, 404, 409)
4. HTTP status codes
5. Authentication (Bearer token)

For each endpoint, include:
- Clear summary and description
- Required and optional parameters
- Example request/response bodies
- Possible error codes

Output as YAML or JSON.
```

**What to do with AI output:**
- Import the YAML/JSON into Swagger Editor (swagger.io/tools/swagger-editor)
- Verify all endpoints render correctly
- Test examples (do they make sense?)
- Check status codes match design doc
- Review error response format
- Commit the spec file to git

### ✅ Definition of Done for Phase 4
- [ ] All endpoints documented in Swagger
- [ ] Request/response schemas defined
- [ ] All error cases documented
- [ ] Examples for each endpoint
- [ ] Viewable in Swagger UI
- [ ] Team can see and review API contract
- [ ] Frontend team can mock API based on spec

---

## Phase 5: Implementation with TDD

### 🎯 Purpose
**Write the code** using Test-Driven Development: Red → Green → Refactor

### 📝 TDD Cycle Explained

**Red Phase:** Write a test that fails
```typescript
describe('UsersService', () => {
  it('should create a user with valid email and password', async () => {
    const createUserDto = {
      email: 'test@example.com',
      password: 'SecurePass123!',
    };

    const user = await usersService.create(createUserDto);

    expect(user).toBeDefined();
    expect(user.email).toBe(createUserDto.email);
    expect(user.id).toBeDefined();
    // Password should be hashed, not stored as plain text
    expect(user.password).not.toBe(createUserDto.password);
  });
});
```

**Test runs:** ❌ FAILS (because service doesn't exist yet)

**Green Phase:** Write minimal code to make test pass
```typescript
@Injectable()
export class UsersService {
  async create(createUserDto: CreateUserDto) {
    const hashedPassword = await bcrypt.hash(createUserDto.password, 10);
    const user = await this.usersRepository.save({
      email: createUserDto.email,
      password: hashedPassword,
    });
    return user;
  }
}
```

**Test runs:** ✅ PASSES

**Refactor Phase:** Improve code without changing behavior
```typescript
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
    private readonly logger: Logger,
  ) {}

  async create(createUserDto: CreateUserDto): Promise<UserResponseDto> {
    this.logger.debug(`Creating user with email: ${createUserDto.email}`);

    const hashedPassword = await this._hashPassword(createUserDto.password);
    const user = await this.usersRepository.save({
      email: createUserDto.email,
      password: hashedPassword,
    });

    return this._toResponseDto(user);
  }

  private async _hashPassword(password: string): Promise<string> {
    return bcrypt.hash(password, 10);
  }

  private _toResponseDto(user: User): UserResponseDto {
    const { password, ...userWithoutPassword } = user;
    return userWithoutPassword;
  }
}
```

**Test still passes:** ✅

### Implementation Order (By Module)

```
1. AuthModule
   - Write tests for user registration
   - Implement registration
   - Write tests for login
   - Implement login
   - Write tests for JWT validation
   - Implement JWT strategy

2. UsersModule
   - Write tests for findById
   - Implement findById
   - Write tests for findByEmail
   - Implement findByEmail

3. TasksModule
   - Write tests for task creation
   - Implement task creation
   - Write tests for task retrieval
   - Implement task retrieval
   - Write tests for task updates
   - Implement task updates
   - Write tests for task deletion
   - Implement task deletion
```

### Test Structure

```typescript
describe('TasksController', () => {
  let controller: TasksController;
  let service: TasksService;

  beforeEach(async () => {
    // Setup test module
  });

  describe('create', () => {
    it('should create a task and return 201', async () => {
      // Arrange
      const createTaskDto = { title: 'Test task', priority: 'high' };

      // Act
      const result = await controller.create(createTaskDto);

      // Assert
      expect(result).toBeDefined();
      expect(result.id).toBeDefined();
    });

    it('should return 400 if title is missing', async () => {
      const createTaskDto = { priority: 'high' };

      expect(async () => {
        await controller.create(createTaskDto);
      }).rejects.toThrow(BadRequestException);
    });
  });
});
```

### 🤖 How AI Helps

**Prompt to AI:**
```
Based on this requirement:
FR-2: Create Task
- Authenticated user can create a task
- Task requires: title (string, max 200 chars), description (optional, max 1000 chars)
- Task optional: priority (High/Medium/Low, default Medium), due_date (ISO 8601)
- Task auto-assigned to creator
- Returns 201 with task object including id, createdAt, createdBy

Generate:
1. CreateTaskDto with validation decorators
2. TaskResponseDto with fields to expose
3. Unit tests for TasksService.create() covering:
   - Happy path (valid input)
   - Validation errors (missing title, invalid due_date)
   - Authorization (user can only create for themselves)
4. Unit tests for TasksController.create()

Use Jest and expect() syntax.
```

**What to do with AI output:**
- Copy tests and DTOs as starting templates
- Add any missing edge cases
- Run tests to verify they fail
- Implement code to make tests pass
- Run refactoring
- Commit with test coverage

### ✅ Definition of Done for Phase 5
- [ ] All requirements have corresponding tests
- [ ] All tests pass (green)
- [ ] Code is refactored (clean, readable, no duplication)
- [ ] Test coverage > 80%
- [ ] No console.log statements
- [ ] Logging follows structured format
- [ ] Error handling is consistent
- [ ] DTOs validate all inputs

---

## Phase 6: Integration Testing

### 🎯 Purpose
**Test the entire flow** - controllers, services, database together. Not just units.

### 📝 What to Test

```typescript
describe('Tasks E2E', () => {
  let app: INestApplication;

  beforeAll(async () => {
    // Setup full app with real DB connection (or in-memory for tests)
  });

  describe('Create task flow', () => {
    it('should create a task and retrieve it', async () => {
      // Register user
      const registerRes = await request(app.getHttpServer())
        .post('/auth/register')
        .send({ email: 'test@example.com', password: 'SecurePass123!' });

      const token = registerRes.body.token;

      // Create task
      const createRes = await request(app.getHttpServer())
        .post('/tasks')
        .set('Authorization', `Bearer ${token}`)
        .send({ title: 'Integration test task', priority: 'high' });

      expect(createRes.status).toBe(201);
      const taskId = createRes.body.id;

      // Retrieve task
      const getRes = await request(app.getHttpServer())
        .get(`/tasks/${taskId}`)
        .set('Authorization', `Bearer ${token}`);

      expect(getRes.status).toBe(200);
      expect(getRes.body.title).toBe('Integration test task');
    });
  });

  describe('Permission checks', () => {
    it('should prevent users from modifying others\' tasks', async () => {
      // User A creates task
      // User B tries to update task
      // Should return 403 Forbidden
    });
  });
});
```

### Types of Integration Tests

```
1. User Registration → Login → Create Task (full workflow)
2. Task creation → Assignment → Email notification
3. Task update → Permission checks
4. Task deletion → Cascade behavior
5. Error scenarios (invalid token, missing fields, duplicate email)
6. Concurrent operations (two users updating same task)
```

### 🤖 How AI Helps

**Prompt to AI:**
```
Based on these API endpoints:
[Paste Swagger spec here]

Generate integration tests using supertest that test:

1. Complete workflow: Register user → Login → Create task → Get task
2. Permission checks: User A cannot delete User B's task
3. Error cases:
   - Invalid JWT token returns 401
   - Duplicate email on register returns 409
   - Missing required field returns 400
4. Data persistence: Task created → Retrieved → Has same data

Use jest and supertest syntax.
```

**What to do with AI output:**
- Update with real API endpoints
- Run against your implementation
- Fix failing tests
- Commit test suite

### ✅ Definition of Done for Phase 6
- [ ] All major workflows tested
- [ ] Permission checks verified
- [ ] Error cases covered
- [ ] Data consistency verified
- [ ] Tests pass against real code
- [ ] Integration tests run in CI/CD pipeline

---

## Phase 7: Deploy

### 🎯 Purpose
**Release to production** safely with confidence from all previous phases.

### 📝 Pre-Deployment Checklist

```
Code Quality:
- [ ] All tests pass locally
- [ ] All tests pass in CI/CD
- [ ] Code review approved
- [ ] No console.log or debug code
- [ ] No hardcoded secrets

Configuration:
- [ ] Environment variables configured
- [ ] Database migrations ready
- [ ] Rollback plan documented

Documentation:
- [ ] README.md updated
- [ ] API docs deployed (Swagger)
- [ ] Deployment runbook created
- [ ] Secrets management verified

Performance:
- [ ] Database indexes created
- [ ] Slow queries identified and optimized
- [ ] Load test results acceptable

Security:
- [ ] Secrets not in code/logs
- [ ] HTTPS enforced
- [ ] CORS properly configured
- [ ] Rate limiting enabled
- [ ] Input validation active

Monitoring:
- [ ] Error tracking enabled (Sentry)
- [ ] Performance monitoring enabled
- [ ] Log aggregation configured
- [ ] Alerts configured for failures
```

### Deployment Steps

```bash
# 1. Run final tests
npm run test
npm run test:integration

# 2. Build production image
docker build -t app:v1.0.0 .

# 3. Run database migrations
npm run typeorm migration:run

# 4. Deploy container
docker push app:v1.0.0
kubectl apply -f deployment.yml

# 5. Verify deployment
curl https://api.example.com/health
# Expected: { "status": "ok" }

# 6. Monitor for errors
# Watch logs for next 30 minutes
```

### Rollback Plan

```
If issues occur post-deploy:

1. Immediate: Switch traffic to previous version
   kubectl set image deployment/app app=app:v0.9.9

2. Investigation: Check logs and metrics
   kubectl logs -f deployment/app

3. If data schema changed:
   - Run rollback migration
   - npm run typeorm migration:revert
   - Deploy previous version

4. Notify team
   - Post-mortem after stability confirmed
```

### 🤖 How AI Helps

**Prompt to AI:**
```
I'm deploying a NestJS API to Kubernetes. Generate:

1. Dockerfile for production build (multi-stage)
2. Deployment manifest (deployment.yml)
3. Service manifest (service.yml)
4. Pre-deployment checklist
5. Deployment script with error handling
6. Rollback procedure

Tech stack: NestJS, PostgreSQL, Docker, Kubernetes, GitHub Actions
```

**What to do with AI output:**
- Review security (no exposed secrets)
- Test locally with Docker first
- Verify database migrations run
- Set up monitoring/alerting before deploy
- Deploy to staging first, then production

### ✅ Definition of Done for Phase 7
- [ ] All pre-deployment checks passed
- [ ] Tests pass in CI/CD pipeline
- [ ] Health check returns 200
- [ ] API responds to requests
- [ ] Database queries execute correctly
- [ ] Logs appear in aggregation system
- [ ] Monitoring alerts fire correctly
- [ ] Team aware of deployment

---

## Quick Reference: AI Prompts by Phase

### Phase 1: Initial Spec
```
Generate an initial spec for [your project idea].
Include: problem statement, goals, scope (in/out), constraints.
```

### Phase 2: Requirements
```
Based on this spec: [spec]
Generate functional and non-functional requirements with:
ID, description, validation rules, response format, error cases.
```

### Phase 3: Design Doc
```
Based on these requirements: [requirements]
Generate database schema, API endpoints, module architecture, auth flow, error handling.
```

### Phase 4: Swagger Doc
```
Based on this API design: [design]
Generate complete OpenAPI 3.0 spec with all endpoints, parameters, schemas, examples.
```

### Phase 5: TDD Implementation
```
Based on this requirement: [requirement]
Generate:
1. DTOs with validation
2. Unit tests (should fail initially)
3. Hints for implementation (don't write full code)
```

### Phase 6: Integration Tests
```
Based on these workflows: [workflows]
Generate E2E tests using supertest covering happy paths and error cases.
```

### Phase 7: Deploy
```
Generate deployment artifacts for [platform]:
Dockerfile, deployment manifest, pre-deployment checklist, rollback procedure.
```

---

## Workflow Summary

| Phase | Input | Output | AI Role |
|-------|-------|--------|---------|
| 1. Initial Spec | Ideas | Clear problem & goals | Generate spec template |
| 2. Requirements | Spec | Detailed requirements | Generate FR/NFR list |
| 3. Design Doc | Requirements | Architecture decisions | Generate design template |
| 4. Swagger Doc | Design | API contract | Generate OpenAPI spec |
| 5. TDD Implementation | Swagger | Working code + tests | Generate tests & DTOs |
| 6. Integration Tests | Code | Confidence in workflows | Generate E2E tests |
| 7. Deploy | Code + Tests | Production system | Generate deployment code |

---

## 80/20 Takeaway

**The order matters because each phase builds on the previous one:**
1. Spec = everyone agrees what we're building
2. Requirements = everyone knows what success looks like
3. Design = everyone sees how we'll build it
4. Swagger = frontend/backend teams can work in parallel
5. TDD = code exists and is tested
6. Integration = systems work together
7. Deploy = users can use it

**AI's role:** Accelerate each phase by generating drafts. Your role: validate, refine, and decide.

---

## Next Steps

Once you confirm this workflow matches your vision, we can:
1. Create templates for each phase
2. Build AI prompts optimized for your project
3. Set up automation (CI/CD, pre-commit hooks, etc.)
4. Test the workflow on a small project first
