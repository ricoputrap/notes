# NestJS REST API Development Checklist

A practical checklist covering everything you need to create, consider, and prepare when building a production-ready REST API with NestJS.

---

## Phase 1: Project Setup & Foundation

### Architecture & Structure
- [ ] **Define your module structure** - Organize by feature (users, posts, auth, etc.)
  - Each feature has: `feature.module.ts`, `feature.controller.ts`, `feature.service.ts`, DTOs, entities
  - Keep modules cohesive and loosely coupled
- [ ] **Create folder structure**
  ```
  src/
  ├── modules/          # Feature modules
  │   ├── users/
  │   ├── posts/
  │   └── auth/
  ├── common/           # Shared utilities
  │   ├── decorators/
  │   ├── guards/
  │   ├── interceptors/
  │   └── filters/
  ├── config/           # Configuration
  ├── database/         # DB setup
  └── main.ts
  ```
- [ ] **Set up environment variables** - `.env` for development, separate configs for prod/test
  - Use `@nestjs/config` with validation
  - Never commit `.env` files (add to `.gitignore`)

### Database Setup
- [ ] **Choose your database** - PostgreSQL (recommended for production), MySQL, MongoDB, SQLite
- [ ] **Set up TypeORM or Prisma** - Object-relational mapping
  - Install: `npm install @nestjs/typeorm typeorm`
  - Create database module with connection configuration
  - Use entities to define your data models
- [ ] **Create your first entity/model**
  ```typescript
  @Entity()
  export class User {
    @PrimaryGeneratedColumn()
    id: number;

    @Column()
    email: string;

    @CreateDateColumn()
    createdAt: Date;
  }
  ```
- [ ] **Set up migrations** - Track database schema changes
  - Create initial migration: `npm run typeorm migration:create`
  - Test migration rollback/forward

### Core Modules
- [ ] **AppModule** - Root module that imports all feature modules
- [ ] **DatabaseModule** - Handles TypeORM/Prisma configuration
- [ ] **ConfigModule** - Environment variables and app configuration
- [ ] **First feature module** - Users, Auth, or core entity (depends on your project)

---

## Phase 2: API Fundamentals

### Controllers & Routing
- [ ] **Create controllers** - Thin routing layer
  - Define routes with `@Get()`, `@Post()`, `@Put()`, `@Delete()`, `@Patch()`
  - Use route parameters: `@Param('id')`, `@Query()`, `@Body()`
  - Keep controllers focused on HTTP concerns only
- [ ] **HTTP status codes** - Return appropriate status codes
  - 200 OK (successful GET, PUT, PATCH)
  - 201 Created (successful POST)
  - 204 No Content (successful DELETE)
  - 400 Bad Request (validation error)
  - 401 Unauthorized (authentication required)
  - 403 Forbidden (not permitted)
  - 404 Not Found
  - 500 Internal Server Error

### Services & Business Logic
- [ ] **Create services** - Handle all business logic
  - Database queries
  - External API calls
  - Calculations and transformations
  - Validation of business rules
- [ ] **Dependency injection** - Inject services into controllers
  ```typescript
  constructor(private usersService: UsersService) {}
  ```
- [ ] **Error handling in services** - Throw appropriate exceptions
  ```typescript
  throw new NotFoundException('User not found');
  throw new BadRequestException('Email already exists');
  throw new UnauthorizedException('Invalid credentials');
  ```

### DTOs (Data Transfer Objects)
- [ ] **Create DTOs for request validation**
  ```typescript
  export class CreateUserDto {
    @IsEmail()
    email: string;

    @MinLength(8)
    password: string;

    @IsString()
    name: string;
  }
  ```
- [ ] **Create DTOs for responses** - Expose only necessary fields
  ```typescript
  export class UserResponseDto {
    id: number;
    email: string;
    name: string;
    // Don't expose: password, passwordHash, internalNotes, etc.
  }
  ```
- [ ] **Use class-validator** - Validate incoming data
  - Install: `npm install class-validator class-transformer`
  - Apply `@UseFilters(ValidationPipe)` globally in `main.ts`

---

## Phase 3: Data Validation & Type Safety

### Request Validation
- [ ] **Global validation pipe**
  ```typescript
  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,           // Remove unknown properties
    forbidNonWhitelisted: true, // Throw error on unknown properties
    transform: true,           // Auto-transform payloads to DTO instances
  }));
  ```
- [ ] **Validate all incoming data** - DTOs with decorators
  - Email format, string lengths, number ranges
  - Custom validators for business rules
- [ ] **Handle validation errors** - Return meaningful error messages
  - User-friendly error responses
  - Log validation failures

### TypeScript Best Practices
- [ ] **Use strong typing** - Avoid `any`
  - Explicitly type service methods
  - Type all function parameters and returns
- [ ] **Enums for constants** - Status, roles, types
  ```typescript
  enum UserRole {
    ADMIN = 'admin',
    USER = 'user',
    GUEST = 'guest',
  }
  ```

---

## Phase 4: Authentication & Authorization

### Authentication (If Required)
- [ ] **Choose auth strategy**
  - JWT (tokens) - stateless, scalable, good for APIs
  - Sessions - stateful, traditional approach
  - OAuth2 - third-party integrations
- [ ] **Implement JWT (recommended for REST APIs)**
  - Install: `npm install @nestjs/jwt @nestjs/passport passport passport-jwt`
  - Create auth module with JWT strategy
  - Hash passwords: `npm install bcrypt`
- [ ] **Create auth service**
  - User registration
  - Login with credentials
  - Token generation and validation
- [ ] **Protect routes** - Use JWT guard
  ```typescript
  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Req() req) {
    return req.user;
  }
  ```

### Authorization (If Required)
- [ ] **Role-based access control (RBAC)**
  - Define user roles (admin, user, guest)
  - Create role guard: `@UseGuards(RolesGuard)`
  - Use decorators: `@Roles(UserRole.ADMIN)`
- [ ] **Permission checks** - Verify user can access resource
  - Owned resources (user can only access their own data)
  - Role-based endpoints (only admins can delete users)

---

## Phase 5: Error Handling & Logging

### Exception Handling
- [ ] **Create custom exception filter**
  ```typescript
  @Catch()
  export class AllExceptionsFilter implements ExceptionFilter {
    catch(exception: unknown, host: ArgumentsHost) {
      const ctx = host.switchToHttp();
      const response = ctx.getResponse<Response>();

      // Handle different exception types
      // Log the error
      // Return appropriate status code
    }
  }
  ```
- [ ] **Use NestJS built-in exceptions**
  - `NotFoundException`, `BadRequestException`, `UnauthorizedException`
  - Throw with meaningful messages
- [ ] **Global error handling** - Register filter in `main.ts`
  ```typescript
  app.useGlobalFilters(new AllExceptionsFilter());
  ```

### Logging
- [ ] **Set up logging library** - `@nestjs/common` Logger or Winston
  - Log errors with full context
  - Log important operations (auth, creation, deletion)
  - Use appropriate log levels (ERROR, WARN, INFO, DEBUG)
- [ ] **Structured logging** - See [Logging Best Practices](./logging-best-practices.md)
  - Include traceId for request correlation
  - Log user context (userId, email)
  - Log operation details (what changed, timestamps)
- [ ] **Log levels by environment**
  - Production: ERROR and WARN
  - Staging: INFO
  - Development: DEBUG

---

## Phase 6: Database Operations

### CRUD Operations
- [ ] **Create (POST)**
  - Validate input with DTOs
  - Check for duplicates
  - Save to database
  - Return created entity (201 status)
- [ ] **Read (GET)**
  - Fetch one: by ID with `findOne()`
  - Fetch many: list with pagination/filtering
  - Return appropriate status (200 OK, 404 Not Found)
- [ ] **Update (PUT/PATCH)**
  - Verify resource exists
  - Validate updated data
  - Update and return modified entity
  - Handle optimistic locking (if concurrent updates possible)
- [ ] **Delete (DELETE)**
  - Verify resource exists
  - Check permissions
  - Soft delete vs hard delete decision
  - Return 204 No Content or deleted entity

### Query Optimization
- [ ] **Use pagination** - Limit results to prevent data overload
  ```typescript
  @Get()
  findAll(@Query('page') page = 1, @Query('limit') limit = 10) {
    return this.service.findAll(page, limit);
  }
  ```
- [ ] **Implement filtering** - Allow users to filter results
- [ ] **Select specific fields** - Don't fetch unnecessary columns
- [ ] **Use relationships efficiently** - Eager vs lazy loading
- [ ] **Index databases appropriately** - Create indexes on frequently queried fields

### Transactions
- [ ] **Handle multi-step operations** - Wrap in transactions
  ```typescript
  // Create user → Send email → Update status (all or nothing)
  ```
- [ ] **Implement rollback logic** - If one step fails, revert all

---

## Phase 7: API Design & Documentation

### RESTful Design
- [ ] **Follow REST conventions**
  - Resources as nouns: `/users`, `/posts`, not `/getUsers`
  - HTTP methods for operations: GET (read), POST (create), PUT/PATCH (update), DELETE (delete)
  - Nested resources: `/users/:id/posts` for relationships
- [ ] **Consistent naming** - Use kebab-case for URL paths
  ```
  GET    /users
  GET    /users/:id
  POST   /users
  PUT    /users/:id
  DELETE /users/:id
  ```
- [ ] **Versioning (optional)** - `/api/v1/users` for backwards compatibility

### API Documentation
- [ ] **Set up Swagger/OpenAPI** - Auto-generated documentation
  - Install: `npm install @nestjs/swagger swagger-ui-express`
  - Add decorators to endpoints
  - Generate interactive API docs at `/api/docs`
- [ ] **Document DTOs** - Add descriptions for request/response fields
  ```typescript
  export class CreateUserDto {
    @ApiProperty({ description: 'User email address' })
    @IsEmail()
    email: string;
  }
  ```
- [ ] **Document error responses** - What can go wrong with each endpoint

---

## Phase 8: Testing

### Unit Tests (Services)
- [ ] **Test all business logic** - See [Testing Fundamentals](./software-testing-fundamentals.md)
  - Test create, read, update, delete
  - Test error cases and edge cases
  - Mock external dependencies
- [ ] **Use Jest** - NestJS default testing framework
  - Create `.spec.ts` files alongside source
  - Run: `npm run test`

### Integration Tests (Controllers + Services)
- [ ] **Test HTTP endpoints** - Request/response cycle
  - Use supertest: `npm install supertest`
  - Test status codes, response format, validation
  - Test with real/in-memory database

### Test Database
- [ ] **Use separate test database** - Don't modify production data
  - SQLite in-memory for fast tests
  - Or separate PostgreSQL instance
- [ ] **Seed test data** - Fixtures for each test
- [ ] **Clean up after tests** - Reset database state

---

## Phase 9: Security

### Input Validation
- [ ] **Validate all inputs** - Prevent injection attacks
  - SQL injection: Use parameterized queries (TypeORM does this)
  - XSS: Validate and sanitize inputs
  - Command injection: Avoid shell commands
- [ ] **Use DTOs with validation** - Already done in Phase 3
- [ ] **Whitelist approach** - Only allow expected fields

### Authentication Security
- [ ] **Hash passwords** - Never store plain text
  - Use bcrypt: `npm install bcrypt`
  - Hash on registration, compare on login
- [ ] **JWT security**
  - Use strong secret key (32+ characters)
  - Set token expiration (15 min for access token)
  - Refresh tokens for long sessions
  - Never expose secret in code
- [ ] **HTTPS only** - Enforce in production
  - All API calls over HTTPS
  - Set secure cookie flags

### Data Protection
- [ ] **Never expose sensitive data** - Use response DTOs
  - Don't return passwords, tokens, internal IDs
  - Filter data in responses
- [ ] **Implement rate limiting** - Prevent brute force attacks
  - Install: `npm install @nestjs/throttler`
  - Apply to auth endpoints especially
- [ ] **CORS configuration** - Restrict origins
  ```typescript
  app.enableCors({
    origin: process.env.ALLOWED_ORIGINS?.split(','),
    credentials: true,
  });
  ```

---

## Phase 10: Configuration & Deployment

### Environment Configuration
- [ ] **Create `.env.example`** - Document required variables
- [ ] **Use ConfigModule** - Load and validate environment variables
  ```typescript
  ConfigModule.forRoot({
    isGlobal: true,
    validationSchema: Joi.object({
      DATABASE_URL: Joi.string().required(),
      JWT_SECRET: Joi.string().required(),
    }),
  });
  ```
- [ ] **Separate configs by environment** - dev, staging, production

### Production Readiness
- [ ] **Database connection pooling** - Reuse connections efficiently
- [ ] **Error monitoring** - Sentry or similar service
  - Catch and report production errors
  - Track error frequency and impacts
- [ ] **Performance monitoring** - Track API response times
- [ ] **Health check endpoint**
  ```typescript
  @Get('health')
  healthCheck() {
    return { status: 'ok', timestamp: new Date() };
  }
  ```
- [ ] **Graceful shutdown** - Close connections on exit
  ```typescript
  app.enableShutdownHooks();
  ```

### Docker & Containerization
- [ ] **Create Dockerfile** - Build reproducible environment
- [ ] **Create docker-compose.yml** - Local dev with services (DB, Redis, etc.)
- [ ] **Multi-stage build** - Optimize image size

### Deployment
- [ ] **Choose platform** - Heroku, AWS, DigitalOcean, Vercel, etc.
- [ ] **Set up CI/CD** - GitHub Actions, GitLab CI, etc.
  - Run tests on push
  - Build Docker image
  - Deploy to staging/production
- [ ] **Database migrations** - Run migrations on deploy
- [ ] **Secrets management** - Use environment variables, not hardcoded

---

## Phase 11: Monitoring & Maintenance

### Observability
- [ ] **Centralized logging** - See [Logging Best Practices](./logging-best-practices.md)
  - All logs to single location (ELK, CloudWatch, etc.)
  - Structured logging with context
  - Long-term retention for analysis
- [ ] **Application metrics** - Track key indicators
  - Request count, error rate, response time
  - Database query performance
  - Memory/CPU usage
- [ ] **Distributed tracing** - Track requests across services
  - Use traceId in all logs
  - Correlate related operations

### Feature Flags
- [ ] **Implement feature flags** - See [Feature Flags Guide](./feature-flags.md)
  - Safely deploy new features
  - A/B test changes
  - Quick rollback without redeployment
  - Gradual rollout to users

### Documentation
- [ ] **API documentation** - Swagger/OpenAPI (done in Phase 7)
- [ ] **Architecture documentation** - Module structure, decisions
- [ ] **Deployment guide** - How to set up and deploy
- [ ] **CONTRIBUTING.md** - Guidelines for contributors
- [ ] **Runbook** - Common operational tasks

---

## Quick Reference: File Checklist

### Per Feature Module
- [ ] `user.module.ts` - Module definition
- [ ] `user.controller.ts` - HTTP routes
- [ ] `user.service.ts` - Business logic
- [ ] `user.entity.ts` - Database model
- [ ] `dto/create-user.dto.ts` - Input validation
- [ ] `dto/user-response.dto.ts` - Output format
- [ ] `user.service.spec.ts` - Service tests
- [ ] `user.controller.spec.ts` - Controller tests

### Root-Level
- [ ] `main.ts` - Application entry point
- [ ] `app.module.ts` - Root module
- [ ] `.env` - Environment variables (don't commit)
- [ ] `.env.example` - Template with required vars
- [ ] `Dockerfile` - Container definition
- [ ] `docker-compose.yml` - Local dev environment
- [ ] `.gitignore` - Exclude sensitive/temporary files
- [ ] `package.json` - Dependencies and scripts
- [ ] `.eslintrc.js` - Code style rules
- [ ] `jest.config.js` - Testing configuration

---

## 80/20 Summary

**Start with these 5 things:**
1. **Module + Controller + Service + Entity** - Get basic CRUD working
2. **DTOs + Validation** - Input/output safety
3. **Error handling** - Consistent, meaningful error responses
4. **Logging** - Understand what's happening in production
5. **Testing** - Confidence that code works as intended

**Then add (as needed):**
6. **Authentication** - Only if your API requires it
7. **Authorization** - Only for role-based features
8. **Database optimization** - When performance becomes an issue
9. **Monitoring** - When you have real users
10. **Documentation** - When others need to use your API

---

## Useful References
- [NestJS Official Docs](https://docs.nestjs.com)
- [NestJS Core Concepts Guide](./nestjs-core-concepts.md) - Your notes
- [Logging Best Practices](./logging-best-practices.md) - Structured logging
- [Software Testing Fundamentals](./software-testing-fundamentals.md) - Testing strategies
- [Feature Flags Guide](./feature-flags.md) - Safe deployments
- [TypeScript Core Concepts](./typescript-core-concepts.md) - Type safety
