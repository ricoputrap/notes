---
name: sdd-phase6-integration
description: Generate E2E integration tests covering complete workflows, permissions, and error scenarios
argument-hint: [swagger-file-path]
user-invocable: true
---

# Phase 6: Integration Testing Generation Skill

Generate comprehensive E2E integration tests covering complete workflows and system interactions.

## How to Use This Skill

**Invoke manually:**
```
/sdd-phase6-integration
/sdd-phase6-integration path/to/swagger-spec.yaml
```

**What AI will do:**
1. Read your Swagger spec
2. Generate E2E test workflows (register → login → create → retrieve, etc.)
3. Generate permission/authorization tests
4. Generate error scenario tests (validation, conflicts, not found, etc.)
5. Generate concurrent operation tests
6. Provide output ready for testing against real code

## Integration Testing Strategy

Integration tests verify:
- **Workflows:** Complete user journeys end-to-end
- **Permissions:** Users can't access others' data
- **Data consistency:** Mutations persist correctly
- **Error handling:** Proper status codes and messages
- **System interactions:** Services work together

## Output Format

The generated integration tests will include:

### 1. E2E Workflow Tests
```typescript
describe('Authentication Flow (E2E)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  afterEach(async () => {
    // Clean up test data
    await cleanDatabase();
  });

  describe('Register → Login → Create Task Flow', () => {
    it('should complete full workflow from registration to task creation', async () => {
      // Step 1: Register user
      const registerRes = await request(app.getHttpServer())
        .post('/auth/register')
        .send({
          email: 'test@example.com',
          password: 'SecurePass123!',
        })
        .expect(201);

      expect(registerRes.body).toHaveProperty('token');
      expect(registerRes.body).toHaveProperty('id');
      const userId = registerRes.body.id;
      const token = registerRes.body.token;

      // Step 2: Login with same credentials
      const loginRes = await request(app.getHttpServer())
        .post('/auth/login')
        .send({
          email: 'test@example.com',
          password: 'SecurePass123!',
        })
        .expect(200);

      expect(loginRes.body).toHaveProperty('token');

      // Step 3: Create task with auth token
      const createTaskRes = await request(app.getHttpServer())
        .post('/tasks')
        .set('Authorization', `Bearer ${token}`)
        .send({
          title: 'Integration test task',
          description: 'Test task created in E2E test',
          priority: 'high',
        })
        .expect(201);

      expect(createTaskRes.body).toHaveProperty('id');
      expect(createTaskRes.body.title).toBe('Integration test task');
      expect(createTaskRes.body.created_by).toBe(userId);
      const taskId = createTaskRes.body.id;

      // Step 4: Retrieve task
      const getTaskRes = await request(app.getHttpServer())
        .get(`/tasks/${taskId}`)
        .set('Authorization', `Bearer ${token}`)
        .expect(200);

      expect(getTaskRes.body.id).toBe(taskId);
      expect(getTaskRes.body.title).toBe('Integration test task');

      // Step 5: Update task
      const updateTaskRes = await request(app.getHttpServer())
        .put(`/tasks/${taskId}`)
        .set('Authorization', `Bearer ${token}`)
        .send({
          status: 'completed',
          priority: 'medium',
        })
        .expect(200);

      expect(updateTaskRes.body.status).toBe('completed');

      // Step 6: Delete task
      await request(app.getHttpServer())
        .delete(`/tasks/${taskId}`)
        .set('Authorization', `Bearer ${token}`)
        .expect(204);

      // Verify task is deleted
      await request(app.getHttpServer())
        .get(`/tasks/${taskId}`)
        .set('Authorization', `Bearer ${token}`)
        .expect(404);
    });
  });

  describe('List Tasks with Filters', () => {
    it('should list tasks and filter by status', async () => {
      // Setup: Create user and multiple tasks
      const registerRes = await request(app.getHttpServer())
        .post('/auth/register')
        .send({ email: 'filter-test@example.com', password: 'Pass123!' })
        .expect(201);

      const token = registerRes.body.token;

      // Create task 1 (pending)
      await request(app.getHttpServer())
        .post('/tasks')
        .set('Authorization', `Bearer ${token}`)
        .send({ title: 'Pending task', priority: 'high' })
        .expect(201);

      // Create task 2 (completed)
      await request(app.getHttpServer())
        .post('/tasks')
        .set('Authorization', `Bearer ${token}`)
        .send({ title: 'Completed task', priority: 'low' })
        .expect(201)
        .then((res) => {
          // Update to completed
          return request(app.getHttpServer())
            .put(`/tasks/${res.body.id}`)
            .set('Authorization', `Bearer ${token}`)
            .send({ status: 'completed' })
            .expect(200);
        });

      // List all tasks
      const allRes = await request(app.getHttpServer())
        .get('/tasks')
        .set('Authorization', `Bearer ${token}`)
        .expect(200);

      expect(allRes.body).toHaveLength(2);

      // Filter by status
      const completedRes = await request(app.getHttpServer())
        .get('/tasks?status=completed')
        .set('Authorization', `Bearer ${token}`)
        .expect(200);

      expect(completedRes.body).toHaveLength(1);
      expect(completedRes.body[0].status).toBe('completed');
    });
  });
});
```

### 2. Permission/Authorization Tests
```typescript
describe('Authorization (E2E)', () => {
  let app: INestApplication;
  let user1Token: string;
  let user2Token: string;
  let user1Id: number;
  let taskOwnedByUser1: number;

  beforeAll(async () => {
    // Setup app
  });

  describe('Task Ownership', () => {
    it('should prevent users from updating others\' tasks', async () => {
      // User 1 creates a task
      const createRes = await request(app.getHttpServer())
        .post('/tasks')
        .set('Authorization', `Bearer ${user1Token}`)
        .send({ title: 'User 1 task' })
        .expect(201);

      const taskId = createRes.body.id;

      // User 2 tries to update User 1's task
      await request(app.getHttpServer())
        .put(`/tasks/${taskId}`)
        .set('Authorization', `Bearer ${user2Token}`)
        .send({ title: 'Hacked!' })
        .expect(403); // Forbidden
    });

    it('should prevent users from deleting others\' tasks', async () => {
      // User 1 creates a task
      const createRes = await request(app.getHttpServer())
        .post('/tasks')
        .set('Authorization', `Bearer ${user1Token}`)
        .send({ title: 'User 1 task' })
        .expect(201);

      const taskId = createRes.body.id;

      // User 2 tries to delete User 1's task
      await request(app.getHttpServer())
        .delete(`/tasks/${taskId}`)
        .set('Authorization', `Bearer ${user2Token}`)
        .expect(403); // Forbidden
    });

    it('should allow users to read others\' tasks if shared', async () => {
      // User 1 creates a task assigned to User 2
      const createRes = await request(app.getHttpServer())
        .post('/tasks')
        .set('Authorization', `Bearer ${user1Token}`)
        .send({
          title: 'Assigned to User 2',
          assigned_to: user2Id,
        })
        .expect(201);

      const taskId = createRes.body.id;

      // User 2 can read the assigned task
      const getRes = await request(app.getHttpServer())
        .get(`/tasks/${taskId}`)
        .set('Authorization', `Bearer ${user2Token}`)
        .expect(200);

      expect(getRes.body.id).toBe(taskId);
    });
  });

  describe('Authentication Requirements', () => {
    it('should return 401 if no token provided', async () => {
      await request(app.getHttpServer())
        .get('/tasks')
        .expect(401);
    });

    it('should return 401 if token is invalid', async () => {
      await request(app.getHttpServer())
        .get('/tasks')
        .set('Authorization', 'Bearer invalid-token')
        .expect(401);
    });

    it('should allow unauthenticated POST /auth/register', async () => {
      await request(app.getHttpServer())
        .post('/auth/register')
        .send({ email: 'test@example.com', password: 'Pass123!' })
        .expect(201 || 409); // 201 created or 409 conflict if exists
    });
  });
});
```

### 3. Error Scenario Tests
```typescript
describe('Error Handling (E2E)', () => {
  let app: INestApplication;
  let token: string;

  describe('Validation Errors', () => {
    it('should return 400 for missing required field', async () => {
      const res = await request(app.getHttpServer())
        .post('/auth/register')
        .send({ email: 'test@example.com' }) // Missing password
        .expect(400);

      expect(res.body).toHaveProperty('message');
      expect(res.body).toHaveProperty('errors');
    });

    it('should return 400 for invalid email format', async () => {
      await request(app.getHttpServer())
        .post('/auth/register')
        .send({
          email: 'not-an-email',
          password: 'SecurePass123!',
        })
        .expect(400);
    });

    it('should return 400 for password too short', async () => {
      await request(app.getHttpServer())
        .post('/auth/register')
        .send({
          email: 'test@example.com',
          password: 'Short1!', // Less than 8 chars
        })
        .expect(400);
    });

    it('should return 400 for task title exceeding max length', async () => {
      await request(app.getHttpServer())
        .post('/tasks')
        .set('Authorization', `Bearer ${token}`)
        .send({
          title: 'a'.repeat(201), // Exceeds 200 char limit
        })
        .expect(400);
    });
  });

  describe('Conflict Errors', () => {
    it('should return 409 when registering with duplicate email', async () => {
      const email = 'duplicate@example.com';

      // First registration
      await request(app.getHttpServer())
        .post('/auth/register')
        .send({ email, password: 'Pass123!' })
        .expect(201);

      // Second registration with same email
      await request(app.getHttpServer())
        .post('/auth/register')
        .send({ email, password: 'Pass123!' })
        .expect(409); // Conflict
    });
  });

  describe('Not Found Errors', () => {
    it('should return 404 for non-existent task', async () => {
      await request(app.getHttpServer())
        .get('/tasks/99999')
        .set('Authorization', `Bearer ${token}`)
        .expect(404);
    });

    it('should return 404 for non-existent user login', async () => {
      await request(app.getHttpServer())
        .post('/auth/login')
        .send({
          email: 'nonexistent@example.com',
          password: 'AnyPass123!',
        })
        .expect(401 || 404); // Either unauthorized or not found
    });
  });

  describe('Authentication Errors', () => {
    it('should return 401 for wrong password', async () => {
      const email = 'test@example.com';

      // Register user
      await request(app.getHttpServer())
        .post('/auth/register')
        .send({ email, password: 'CorrectPass123!' })
        .expect(201);

      // Try to login with wrong password
      await request(app.getHttpServer())
        .post('/auth/login')
        .send({ email, password: 'WrongPass123!' })
        .expect(401); // Unauthorized
    });
  });
});
```

### 4. Concurrent Operation Tests
```typescript
describe('Concurrent Operations (E2E)', () => {
  let app: INestApplication;
  let token: string;

  it('should handle two users creating tasks concurrently', async () => {
    const user1Task = request(app.getHttpServer())
      .post('/tasks')
      .set('Authorization', `Bearer ${token1}`)
      .send({ title: 'User 1 task' });

    const user2Task = request(app.getHttpServer())
      .post('/tasks')
      .set('Authorization', `Bearer ${token2}`)
      .send({ title: 'User 2 task' });

    const [res1, res2] = await Promise.all([user1Task, user2Task]);

    expect(res1.status).toBe(201);
    expect(res2.status).toBe(201);
    expect(res1.body.id).not.toBe(res2.body.id); // Different IDs
  });

  it('should prevent race conditions on duplicate email', async () => {
    const email = `race-test-${Date.now()}@example.com`;

    const register1 = request(app.getHttpServer())
      .post('/auth/register')
      .send({ email, password: 'Pass123!' });

    const register2 = request(app.getHttpServer())
      .post('/auth/register')
      .send({ email, password: 'Pass123!' });

    const [res1, res2] = await Promise.all([register1, register2]);

    // One should succeed (201), one should conflict (409)
    const statuses = [res1.status, res2.status].sort();
    expect(statuses).toEqual([201, 409]);
  });
});
```

## Instructions for Claude

When the user invokes this skill:

1. **Obtain the Swagger Spec**
   - If file path provided: Read it
   - If not provided: Ask user to paste or provide path
   - Extract: All endpoints, workflows, authentication requirements

2. **Generate E2E Workflow Tests**
   - Identify main user journeys from requirements:
     - Registration → Login → Create resource
     - List resources → Filter → Sort
     - Create → Update → Delete
   - For each workflow:
     - Step-by-step REST calls
     - Verify each response status and data
     - Test data persistence (create, retrieve, verify same)
     - Test state transitions
   - Use supertest for HTTP testing
   - Include realistic test data
   - Clean up test data after each test (afterEach hook)

3. **Generate Permission/Authorization Tests**
   - Create two test users
   - Test ownership scenarios:
     - User A can't modify User B's resources
     - User A can modify own resources
   - Test authentication:
     - Unauthenticated requests return 401
     - Invalid tokens return 401
     - Valid tokens allow access
   - Test role-based access (if applicable):
     - Admin can perform admin actions
     - Regular users can't access admin endpoints

4. **Generate Error Scenario Tests**
   - Validation errors:
     - Missing required fields
     - Invalid formats (email, date, etc.)
     - Length violations (too long, too short)
     - Enum violations (invalid priority, status, etc.)
   - Conflict errors:
     - Duplicate email on registration
     - Unique constraint violations
   - Not found errors:
     - Non-existent resource retrieval
     - Non-existent user login
   - State errors:
     - Invalid state transitions
     - Operations on deleted resources

5. **Generate Concurrent Operation Tests**
   - Two users creating same resource type simultaneously
   - Race conditions on unique constraints (email)
   - Multiple updates to same resource
   - Verify data consistency under concurrency

6. **Structure the Output**
   - File: `src/test/e2e/app.e2e-spec.ts` (or by feature)
   - Use describe blocks for categories:
     - Workflow tests
     - Authorization tests
     - Error handling tests
     - Concurrent operation tests
   - Common setup: beforeAll, afterAll, afterEach
   - Database cleanup after each test

7. **Include Test Utilities**
   - Helper function to register test user
   - Helper function to login and get token
   - Helper function to clean database
   - Example:
     ```typescript
     async function registerTestUser(email: string): Promise<{ id: number; token: string }> {
       const res = await request(app.getHttpServer())
         .post('/auth/register')
         .send({ email, password: 'TestPass123!' });
       return { id: res.body.id, token: res.body.token };
     }
     ```

8. **Validate the Output**
   - All main workflows covered
   - Permission checks for all data-modifying endpoints
   - Error cases for all validation rules
   - Concurrent operation scenarios realistic
   - Test names clearly describe what's being tested
   - Tests are independent (can run in any order)

9. **Provide Output** as complete test file:
   - Ready to copy to `test/e2e/`
   - With setup/teardown instructions
   - With database seeding hints
   - Runnable with `npm run test:e2e`

10. **Ask for Feedback**: "These E2E tests verify your entire system works together. Run with `npm run test:e2e`. Ready for Phase 7 (Deployment)?"

## Example Interaction

```
User: /sdd-phase6-integration

Claude: I'll generate comprehensive E2E integration tests from your Swagger spec.

Please share your Swagger spec:

[User shares swagger]

Claude: Perfect! Here's your E2E Integration Test Suite:

# Task Management API - Integration Tests (Phase 6)

## Complete Workflows

File: `test/e2e/workflows.e2e-spec.ts`

```typescript
describe('Register → Login → Create Task (E2E)', () => {
  it('should complete full workflow', async () => {
    // Register
    const registerRes = await request(app.getHttpServer())
      .post('/auth/register')
      .send({ email: 'test@example.com', password: 'Pass123!' })
      .expect(201);

    // Login and create task
    ...
  });
});
```

## Authorization Tests

File: `test/e2e/authorization.e2e-spec.ts`

```typescript
describe('Task Ownership', () => {
  it('should prevent users from updating others\' tasks', async () => {
    // User 1 creates task
    // User 2 tries to update
    // Expect 403 Forbidden
  });
});
```

## Error Handling Tests

File: `test/e2e/error-handling.e2e-spec.ts`

## Concurrent Operations Tests

File: `test/e2e/concurrency.e2e-spec.ts`

Run all tests:
```bash
npm run test:e2e
```

Everything should pass! If any integration test fails, it reveals issues in your implementation before deploying.

Ready for Phase 7: Deployment setup?
```

## Tips for Best Results

- **Test complete workflows:** Don't test in isolation
- **Verify data persistence:** Create, retrieve, verify data matches
- **Test error paths:** Validation, permissions, conflicts
- **Use realistic data:** Email that looks like email, dates that are valid
- **Keep tests independent:** One test shouldn't depend on another
- **Clean up after:** Delete test data so tests can repeat
- **Concurrent tests catch race conditions:** Run them!

## Related Skills

After completing integration tests:
- Next: `/sdd-phase7-deploy` - Generate deployment setup
- Previous: `/sdd-phase5-tdd` - Generate unit test stubs
- Reference: [Spec-Driven Development Workflow](../../spec-driven-development-workflow.md) - Phase 6 details
