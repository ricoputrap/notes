# Software Testing Fundamentals

A practical guide to the 20% of testing strategies that cover 80% of real-world applications.

## 1. Why Testing Matters

**The Cost of Bugs:**
- Bugs caught early are cheap to fix
- Bugs found in production are expensive
- Bugs in production damage trust and reputation
- Testing prevents costly rework and hotfixes

**The Pareto Principle in Testing:**
- 80% of bugs come from 20% of the code
- 80% of test value comes from 20% of test cases
- Focus testing effort on high-risk, frequently-used areas
- Don't aim for 100% coverage; aim for smart coverage

---

## 2. Testing Pyramid

The optimal distribution of test effort:

```
        △
       /|\
      / | \
     /  |  \       E2E Tests (5-10%)
    /   |   \      Complex integration flows
   /────┼────\     Fewer tests, slower, high value
  /     |     \
 / Unit Tests  \   Integration Tests (15-25%)
 20-40%        \  Module interactions
 Fast, many    \─────────
                  40-60%
             Unit Tests
         Fast, isolated, many
```

**Testing Distribution:**
- **Unit Tests (40-60%)** - Test individual functions/methods in isolation
- **Integration Tests (25-35%)** - Test how modules work together
- **E2E Tests (5-10%)** - Test complete user workflows

**Key Principle:** Most tests should be fast unit tests. E2E tests should cover only critical paths.

---

## 3. Unit Tests (Fast & Isolated)

**What:** Test a single function or method in isolation.

**Characteristics:**
- Run in milliseconds
- No external dependencies (database, API, filesystem)
- Test one thing per test
- Heavily use mocking and stubbing

**When to Write:**
- For business logic functions
- For utility functions
- For complex calculations
- For error handling

**When NOT to Write:**
- For trivial getters/setters
- For simple wrappers around libraries
- For UI rendering (use component tests instead)

**Example (Jest + TypeScript):**
```typescript
describe('UsersService', () => {
  let service: UsersService;
  let mockRepository: jest.Mocked<Repository<User>>;

  beforeEach(async () => {
    mockRepository = {
      find: jest.fn(),
      findOne: jest.fn(),
      save: jest.fn(),
    } as any;

    const module = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useValue: mockRepository,
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
  });

  describe('findUserByEmail', () => {
    it('should return user when found', async () => {
      const user = { id: 1, email: 'test@test.com' };
      mockRepository.findOne.mockResolvedValue(user);

      const result = await service.findUserByEmail('test@test.com');

      expect(result).toEqual(user);
      expect(mockRepository.findOne).toHaveBeenCalledWith({
        where: { email: 'test@test.com' },
      });
    });

    it('should return null when user not found', async () => {
      mockRepository.findOne.mockResolvedValue(null);

      const result = await service.findUserByEmail('notfound@test.com');

      expect(result).toBeNull();
    });
  });
});
```

**80/20 Tips:**
- Mock all external dependencies
- Test both success and failure paths
- Use descriptive test names
- One assertion per test (when possible)
- Keep tests under 10 lines

---

## 4. Integration Tests (Cross-Module)

**What:** Test how multiple modules or components work together.

**Characteristics:**
- Test real interactions between modules
- May use test databases or in-memory databases
- Slower than unit tests (seconds, not milliseconds)
- Test workflows, not individual functions

**When to Write:**
- Testing API endpoints end-to-end
- Testing complex workflows across services
- Testing database interactions with real queries
- Testing external service integrations

**Example:**
```typescript
describe('Users API (Integration)', () => {
  let app: INestApplication;
  let usersService: UsersService;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [UsersModule, DatabaseModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
    usersService = moduleFixture.get<UsersService>(UsersService);
  });

  afterAll(async () => {
    await app.close();
  });

  describe('POST /users', () => {
    it('should create and return user', async () => {
      const createUserDto: CreateUserDto = {
        email: 'test@test.com',
        name: 'Test User',
        password: 'password123',
      };

      const response = await request(app.getHttpServer())
        .post('/users')
        .send(createUserDto)
        .expect(201);

      expect(response.body).toEqual(
        expect.objectContaining({
          email: 'test@test.com',
          name: 'Test User',
        }),
      );

      const savedUser = await usersService.findUserByEmail('test@test.com');
      expect(savedUser).toBeDefined();
    });

    it('should reject duplicate email', async () => {
      const createUserDto: CreateUserDto = {
        email: 'duplicate@test.com',
        name: 'User 1',
        password: 'password123',
      };

      await request(app.getHttpServer())
        .post('/users')
        .send(createUserDto)
        .expect(201);

      // Try creating again with same email
      await request(app.getHttpServer())
        .post('/users')
        .send({ ...createUserDto, name: 'User 2' })
        .expect(400);
    });
  });
});
```

**80/20 Tips:**
- Test the happy path and main error cases
- Use test databases (in-memory SQLite, test PostgreSQL)
- Test API contracts (status codes, response shape)
- Don't test every combination; focus on critical workflows

---

## 5. End-to-End (E2E) Tests

**What:** Test complete user workflows from start to finish.

**Characteristics:**
- Test the entire system working together
- Run in real browser (for web apps)
- Slowest type of tests (seconds to minutes)
- Most valuable for critical user paths
- Test what users actually do

**When to Write:**
- Critical user workflows (auth, payments, purchases)
- Cross-system integrations
- Full feature validation
- Before major releases

**When NOT to Write:**
- For every feature (too slow)
- For edge cases (use unit tests)
- For UI details (use component tests)

**Example (Cypress):**
```typescript
describe('User Authentication Flow', () => {
  beforeEach(() => {
    cy.visit('http://localhost:3000/login');
  });

  it('should login user and redirect to dashboard', () => {
    // Type credentials
    cy.get('input[name="email"]').type('user@test.com');
    cy.get('input[name="password"]').type('password123');

    // Submit form
    cy.get('button[type="submit"]').click();

    // Should redirect to dashboard
    cy.url().should('include', '/dashboard');

    // Should show user name
    cy.get('[data-testid="user-name"]').should('contain', 'Test User');
  });

  it('should show error for invalid credentials', () => {
    cy.get('input[name="email"]').type('wrong@test.com');
    cy.get('input[name="password"]').type('wrongpassword');
    cy.get('button[type="submit"]').click();

    cy.get('[role="alert"]').should('contain', 'Invalid credentials');
    cy.url().should('include', '/login');
  });
});
```

**80/20 Tips:**
- Focus on critical user paths (auth, core features)
- Use data-testid attributes for reliable selectors
- Keep E2E tests under 5-10 per feature
- Run E2E tests in CI, not on every save
- Mock external APIs to keep tests fast

---

## 6. Test Coverage - The Right Amount

**Coverage Doesn't Equal Quality:**
- 100% coverage doesn't mean no bugs exist
- 80% coverage of critical paths beats 100% coverage of everything
- Focus on:
  - Business logic paths
  - Error handling
  - Edge cases
  - Critical user workflows

**Coverage Guidelines (Pareto):**
| Code Type | Target Coverage | Importance |
|-----------|-----------------|-----------|
| Business logic | 80-100% | Critical |
| Error handling | 70-90% | High |
| API endpoints | 75-90% | High |
| Utilities | 60-80% | Medium |
| UI rendering | 40-60% | Low |
| Trivial code | 0-30% | Very Low |

**What NOT to Test:**
- Auto-generated code
- Third-party library code
- Simple getters/setters
- Framework boilerplate
- Code you don't control

**80/20 Tip:** A well-tested 20% of your code prevents 80% of bugs. Focus there first.

---

## 7. Mocking & Test Doubles

**Why Mock?**
- Isolate the unit being tested
- Control external behavior
- Speed up tests (no real API/DB calls)
- Test error scenarios

**Types of Test Doubles:**

**Stub** - Returns fixed response:
```typescript
const mockUserRepository = {
  findOne: jest.fn().mockResolvedValue({ id: 1, email: 'test@test.com' }),
};
```

**Mock** - Verify it was called correctly:
```typescript
mockRepository.save.mockResolvedValue(user);
// Later: verify it was called
expect(mockRepository.save).toHaveBeenCalledWith(user);
```

**Spy** - Monitor without changing behavior:
```typescript
jest.spyOn(console, 'log');
// Code runs normally, but we track calls
expect(console.log).toHaveBeenCalled();
```

**Example (Nest.js):**
```typescript
it('should call email service when user created', async () => {
  const mockEmailService = {
    sendWelcomeEmail: jest.fn().mockResolvedValue(true),
  };

  const module = await Test.createTestingModule({
    providers: [
      UsersService,
      {
        provide: EmailService,
        useValue: mockEmailService,
      },
    ],
  }).compile();

  const service = module.get<UsersService>(UsersService);

  await service.create({ email: 'new@test.com', name: 'User' });

  // Verify email was sent
  expect(mockEmailService.sendWelcomeEmail).toHaveBeenCalledWith(
    'new@test.com',
  );
});
```

**80/20 Tips:**
- Mock external dependencies (API, DB, email)
- Don't mock the code you're testing
- Use jest.fn() for simple mocks
- Use useValue for dependency injection mocks

---

## 8. Testing Best Practices

### Arrange-Act-Assert Pattern

Every test should follow this structure:

```typescript
it('should return user by email', async () => {
  // Arrange - Set up test data and mocks
  const mockUser = { id: 1, email: 'test@test.com' };
  mockRepository.findOne.mockResolvedValue(mockUser);

  // Act - Execute the code being tested
  const result = await service.findByEmail('test@test.com');

  // Assert - Verify the result
  expect(result).toEqual(mockUser);
});
```

### Clear Test Names

Test names should describe the behavior:

✅ Good: `should return user when email is found`
❌ Bad: `test1`, `findByEmail works`

### One Assertion per Test (When Possible)

```typescript
// Good - focused test
it('should return user', async () => {
  const result = await service.findById(1);
  expect(result.id).toBe(1);
});

// Okay - related assertions
it('should return user with correct data', async () => {
  const result = await service.findById(1);
  expect(result.id).toBe(1);
  expect(result.email).toBe('test@test.com');
});

// Bad - multiple unrelated tests
it('should work', async () => {
  const user = await service.findById(1);
  expect(user).toBeDefined();
  const updated = await service.update(user);
  expect(updated).toBeDefined();
  const deleted = await service.delete(user);
  expect(deleted).toBeTruthy();
});
```

### Test Edge Cases

```typescript
it('should handle empty array', async () => {
  const result = await service.getUsers([]);
  expect(result).toEqual([]);
});

it('should handle null input', async () => {
  expect(() => service.validate(null)).toThrow();
});

it('should handle very large numbers', async () => {
  const result = service.calculate(Number.MAX_SAFE_INTEGER);
  expect(result).toBeDefined();
});
```

### Keep Tests Fast

```typescript
// Slow - 100ms per test × 1000 tests = 100 seconds
it('should create user', async () => {
  // Actually creates in database
  const user = await service.create(userData);
});

// Fast - milliseconds
it('should create user', async () => {
  // Mocks database, returns instantly
  mockRepository.save.mockResolvedValue(user);
  const result = await service.create(userData);
  expect(result).toEqual(user);
});
```

**80/20 Tips:**
- Follow Arrange-Act-Assert
- Use descriptive names
- Test one thing per test
- Mock external dependencies
- Keep tests under 100ms each

---

## 9. Test-Driven Development (TDD)

**The TDD Cycle:**

1. **Red** - Write a test that fails
2. **Green** - Write minimal code to pass the test
3. **Refactor** - Improve code without breaking tests

**Benefits:**
- Better design (you think about interfaces first)
- Fewer bugs (tested from the start)
- Confidence to refactor
- Living documentation

**Example:**
```typescript
// 1. Red - Test fails (function doesn't exist)
it('should calculate discount for loyal customers', () => {
  const discount = calculateDiscount(150, 'loyal');
  expect(discount).toBe(15); // 10% off
});

// 2. Green - Minimal implementation
function calculateDiscount(amount: number, type: string): number {
  if (type === 'loyal') return amount * 0.1;
  return 0;
}

// 3. Refactor - Improve
function calculateDiscount(amount: number, type: 'loyal' | 'new'): number {
  const rates = { loyal: 0.1, new: 0 };
  return amount * rates[type];
}
```

**When to Use TDD:**
- Complex business logic
- Critical features
- When requirements are clear
- For legacy code fixes

**When NOT to Use:**
- Rapid prototyping
- Unclear requirements
- When you're learning the codebase

---

## 10. Running Tests Effectively

### Local Development
```bash
# Run all tests
npm test

# Run in watch mode (re-run on file change)
npm test -- --watch

# Run single test file
npm test -- users.service.spec.ts

# Run with coverage
npm test -- --coverage
```

### CI/CD Pipeline
```bash
# Run tests with coverage report
npm test -- --coverage --ci

# Run E2E tests separately
npm run test:e2e

# Generate coverage report
npm test -- --coverage
```

### Test Organization
```
src/
├── users/
│   ├── users.service.ts
│   ├── users.service.spec.ts          # Unit tests
│   ├── users.controller.ts
│   └── users.controller.spec.ts
├── __tests__/
│   ├── integration/
│   │   └── users.integration.spec.ts  # Integration tests
│   └── e2e/
│       └── auth.e2e.spec.ts           # E2E tests
```

**80/20 Tips:**
- Run tests locally before committing
- Use --watch mode during development
- Run full suite in CI/CD
- Fail the build if tests fail

---

## 11. Common Testing Mistakes

### Mistake 1: Testing Implementation, Not Behavior

❌ Bad:
```typescript
it('should call repository.find', () => {
  service.getUsers();
  expect(mockRepository.find).toHaveBeenCalled();
});
```

✅ Good:
```typescript
it('should return list of users', () => {
  mockRepository.find.mockResolvedValue([user1, user2]);
  const result = service.getUsers();
  expect(result).toEqual([user1, user2]);
});
```

### Mistake 2: Over-Testing

❌ Bad:
```typescript
it('should return 1 when adding 0 + 1', () => {
  expect(add(0, 1)).toBe(1);
});

it('should return 2 when adding 1 + 1', () => {
  expect(add(1, 1)).toBe(2);
});

it('should return 3 when adding 1 + 2', () => {
  expect(add(1, 2)).toBe(3);
});
// ... 100 more tests
```

✅ Good:
```typescript
it('should add two numbers correctly', () => {
  expect(add(0, 1)).toBe(1);
  expect(add(1, 1)).toBe(2);
  expect(add(1, 2)).toBe(3);
});
```

### Mistake 3: Brittle Tests

❌ Bad:
```typescript
it('should render user', () => {
  const html = component.render();
  expect(html).toContain('<div class="user-card p-4 bg-blue"><p>John');
});
```

✅ Good:
```typescript
it('should display user name', () => {
  component.user = { name: 'John' };
  expect(component.getName()).toBe('John');
});
```

### Mistake 4: Testing the Framework

❌ Bad:
```typescript
it('should call Angular's OnInit', () => {
  expect(component.ngOnInit).toBeDefined();
});
```

✅ Good:
```typescript
it('should load user data on init', () => {
  component.ngOnInit();
  expect(component.user).toBeDefined();
});
```

---

## 12. Pareto Principle Summary

**Focus your testing effort on:**

1. **Core Business Logic (80% of test value)**
   - User authentication
   - Payment processing
   - Data validation
   - Error handling

2. **High-Risk Areas (20% of code, 80% of bugs)**
   - Complex algorithms
   - Concurrent code
   - Database interactions
   - External integrations

3. **Critical User Paths (E2E)**
   - Sign up and login
   - Main feature workflows
   - Checkout process
   - Data operations

**Don't Over-Test:**
- UI rendering details
- Framework functionality
- Auto-generated code
- Simple getters/setters
- Third-party libraries

---

## 13. Testing Checklist

When starting to test a codebase:

- [ ] **Unit Tests**
  - [ ] Test business logic functions
  - [ ] Mock external dependencies
  - [ ] Cover success and error paths
  - [ ] Target 70-80% coverage of core logic

- [ ] **Integration Tests**
  - [ ] Test API endpoints
  - [ ] Test database interactions
  - [ ] Test service interactions
  - [ ] Cover main workflows

- [ ] **E2E Tests**
  - [ ] Test critical user paths
  - [ ] Test across system boundaries
  - [ ] Keep under 10 tests per feature
  - [ ] Run in CI/CD only

- [ ] **Coverage**
  - [ ] Focus on critical code
  - [ ] Don't aim for 100%
  - [ ] Report in CI/CD
  - [ ] Track trends over time

- [ ] **Best Practices**
  - [ ] Use Arrange-Act-Assert
  - [ ] Clear, descriptive names
  - [ ] Fast tests (under 100ms each)
  - [ ] Deterministic (no flakiness)

---

## Key Takeaways

1. **Follow the pyramid** - Most tests should be fast unit tests
2. **Test behavior, not implementation** - Focus on what code does, not how
3. **Mock wisely** - Mock external dependencies, test real interactions
4. **Use Pareto principle** - 20% of tests cover 80% of risk
5. **Keep tests fast** - Unit tests in milliseconds, integration in seconds
6. **Test critical paths** - Focus E2E on what users actually do
7. **Avoid brittle tests** - Test behavior, not implementation details
8. **One concern per test** - Make tests focused and readable
9. **Automate in CI/CD** - Tests should run automatically and fail the build if needed
10. **Maintain tests** - Treat tests as code; keep them clean and updated

---

## Quick Reference

**When to use what:**

| Situation | Test Type |
|-----------|-----------|
| Testing a pure function | Unit |
| Testing service with mocked DB | Unit |
| Testing API endpoint with real DB | Integration |
| Testing user workflow | E2E |
| Testing error handling | Unit |
| Testing database queries | Integration |
| Testing UI component behavior | Unit |
| Testing UI user interaction | E2E |

**Command Reference:**

```bash
# Run all tests
npm test

# Run in watch mode
npm test -- --watch

# Run with coverage
npm test -- --coverage

# Run specific file
npm test -- users.service.spec.ts

# Run E2E tests
npm run test:e2e
```

