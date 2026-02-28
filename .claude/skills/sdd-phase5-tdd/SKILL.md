---
name: sdd-phase5-tdd
description: Generate test stubs, DTOs, and service signatures for TDD implementation - Red phase tests that fail
argument-hint: [swagger-file-path]
user-invocable: true
---

# Phase 5: TDD Implementation Generation Skill

Generate test stubs, request/response DTOs, and service method signatures for Test-Driven Development.

## How to Use This Skill

**Invoke manually:**
```
/sdd-phase5-tdd
/sdd-phase5-tdd path/to/swagger-spec.yaml
/sdd-phase5-tdd sdd-phase5-tdd FR-1
```

**What AI will do:**
1. Read your Swagger spec (or ask for it)
2. Generate unit test stubs for services (Red phase - should fail)
3. Generate request/response DTOs with validation decorators
4. Generate service method signatures
5. Generate controller method stubs with Swagger decorators
6. Provide output ready for you to implement (Green phase)

## TDD Cycle Reminder

```
RED Phase:   Write test that fails (AI generates this)
GREEN Phase: Write code to make test pass (you do this)
REFACTOR:    Improve code without changing behavior (you do this)
```

This skill generates **RED phase** artifacts.

## What Constitutes Good TDD Test Stubs

Your tests should:
- **Test behavior, not implementation:** "User can create task" not "database save called"
- **Be specific:** One assertion per test, clear test name
- **Cover happy path:** Main success scenario
- **Cover error cases:** Validation errors, permissions, not found
- **Be independent:** Tests don't depend on each other
- **Be maintainable:** Easy to understand what's being tested

## Output Format

The generated TDD artifacts will include:

### 1. Unit Test Stubs (Jest)
```typescript
describe('TasksService', () => {
  let service: TasksService;
  let repository: MockRepository<Task>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        TasksService,
        {
          provide: getRepositoryToken(Task),
          useClass: MockRepository,
        },
      ],
    }).compile();

    service = module.get<TasksService>(TasksService);
    repository = module.get(getRepositoryToken(Task));
  });

  describe('create', () => {
    it('should create a task with valid input', async () => {
      // Arrange
      const createTaskDto = {
        title: 'Test task',
        description: 'Test description',
        priority: 'high',
      };

      // Act
      const result = await service.create(createTaskDto, userId);

      // Assert
      expect(result).toBeDefined();
      expect(result.id).toBeDefined();
      expect(result.title).toBe(createTaskDto.title);
      expect(repository.save).toHaveBeenCalledWith(
        expect.objectContaining(createTaskDto),
      );
    });

    it('should throw BadRequestException if title is missing', async () => {
      const createTaskDto = { description: 'No title' };

      expect(async () => {
        await service.create(createTaskDto, userId);
      }).rejects.toThrow(BadRequestException);
    });

    it('should throw BadRequestException if title exceeds max length', async () => {
      const createTaskDto = {
        title: 'a'.repeat(201),
        priority: 'high',
      };

      expect(async () => {
        await service.create(createTaskDto, userId);
      }).rejects.toThrow(BadRequestException);
    });

    it('should auto-assign task to creator', async () => {
      const createTaskDto = { title: 'Test task', priority: 'high' };

      await service.create(createTaskDto, userId);

      expect(repository.save).toHaveBeenCalledWith(
        expect.objectContaining({ created_by: userId }),
      );
    });
  });

  describe('findAll', () => {
    it('should return all tasks for authenticated user', async () => {
      const tasks = [
        { id: 1, title: 'Task 1' },
        { id: 2, title: 'Task 2' },
      ];
      repository.find.mockResolvedValue(tasks);

      const result = await service.findAll(userId);

      expect(result).toEqual(tasks);
      expect(repository.find).toHaveBeenCalled();
    });

    it('should filter by status if provided', async () => {
      await service.findAll(userId, { status: 'completed' });

      expect(repository.find).toHaveBeenCalledWith({
        where: expect.objectContaining({ status: 'completed' }),
      });
    });
  });
});
```

### 2. Request/Response DTOs
```typescript
// Request DTOs
export class CreateTaskDto {
  @ApiProperty({ description: 'Task title', maxLength: 200 })
  @IsString()
  @IsNotEmpty()
  @MaxLength(200)
  title: string;

  @ApiProperty({
    description: 'Task description (optional)',
    maxLength: 1000,
    required: false,
  })
  @IsString()
  @IsOptional()
  @MaxLength(1000)
  description?: string;

  @ApiProperty({ description: 'Priority level', enum: ['high', 'medium', 'low'], default: 'medium' })
  @IsEnum(['high', 'medium', 'low'])
  @IsOptional()
  priority?: string = 'medium';

  @ApiProperty({ description: 'Due date (ISO 8601)', required: false })
  @IsISO8601()
  @IsOptional()
  due_date?: string;
}

export class UpdateTaskDto {
  @ApiProperty({ required: false })
  @IsString()
  @IsOptional()
  @MaxLength(200)
  title?: string;

  @ApiProperty({ required: false })
  @IsString()
  @IsOptional()
  @MaxLength(1000)
  description?: string;

  @ApiProperty({ required: false })
  @IsEnum(['high', 'medium', 'low'])
  @IsOptional()
  priority?: string;

  @ApiProperty({ required: false })
  @IsISO8601()
  @IsOptional()
  due_date?: string;
}

// Response DTOs
export class TaskResponseDto {
  @ApiProperty()
  id: number;

  @ApiProperty()
  title: string;

  @ApiProperty()
  description?: string;

  @ApiProperty()
  priority: string;

  @ApiProperty()
  status: string;

  @ApiProperty()
  due_date?: string;

  @ApiProperty()
  created_by: number;

  @ApiProperty()
  assigned_to?: number;

  @ApiProperty()
  created_at: Date;

  @ApiProperty()
  updated_at: Date;
}
```

### 3. Service Method Signatures
```typescript
@Injectable()
export class TasksService {
  constructor(
    @InjectRepository(Task)
    private tasksRepository: Repository<Task>,
  ) {}

  async create(
    createTaskDto: CreateTaskDto,
    userId: number,
  ): Promise<TaskResponseDto> {
    // TODO: Implement
    // 1. Validate input (done by DTO)
    // 2. Auto-assign to creator (created_by = userId)
    // 3. Set default priority if not provided
    // 4. Save to database
    // 5. Return response DTO
  }

  async findAll(
    userId: number,
    filters?: { status?: string; priority?: string },
  ): Promise<TaskResponseDto[]> {
    // TODO: Implement
    // 1. Build query with filters
    // 2. Return all tasks for user
  }

  async findOne(id: number, userId: number): Promise<TaskResponseDto> {
    // TODO: Implement
    // 1. Find task by ID
    // 2. Verify user owns task (or throw 403)
    // 3. Return response DTO
  }

  async update(
    id: number,
    updateTaskDto: UpdateTaskDto,
    userId: number,
  ): Promise<TaskResponseDto> {
    // TODO: Implement
    // 1. Find task by ID
    // 2. Verify user owns task (or throw 403)
    // 3. Update fields
    // 4. Save to database
    // 5. Return response DTO
  }

  async remove(id: number, userId: number): Promise<void> {
    // TODO: Implement
    // 1. Find task by ID
    // 2. Verify user owns task (or throw 403)
    // 3. Delete from database
  }
}
```

### 4. Controller Method Stubs
```typescript
@Controller('tasks')
@ApiTags('Tasks')
@UseGuards(JwtAuthGuard)
export class TasksController {
  constructor(private readonly tasksService: TasksService) {}

  @Post()
  @ApiOperation({ summary: 'Create a new task' })
  @ApiBody({ type: CreateTaskDto })
  @ApiResponse({
    status: 201,
    description: 'Task created',
    type: TaskResponseDto,
  })
  async create(
    @Body() createTaskDto: CreateTaskDto,
    @Req() req: any,
  ): Promise<TaskResponseDto> {
    return this.tasksService.create(createTaskDto, req.user.id);
  }

  @Get()
  @ApiOperation({ summary: 'List all tasks' })
  @ApiQuery({ name: 'status', required: false })
  @ApiResponse({
    status: 200,
    description: 'Task list',
    type: [TaskResponseDto],
  })
  async findAll(
    @Query('status') status?: string,
    @Req() req?: any,
  ): Promise<TaskResponseDto[]> {
    return this.tasksService.findAll(req.user.id, { status });
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get task by ID' })
  @ApiParam({ name: 'id', type: 'number' })
  @ApiResponse({ status: 200, type: TaskResponseDto })
  @ApiResponse({ status: 404, description: 'Task not found' })
  async findOne(
    @Param('id') id: string,
    @Req() req: any,
  ): Promise<TaskResponseDto> {
    return this.tasksService.findOne(+id, req.user.id);
  }

  @Put(':id')
  @ApiOperation({ summary: 'Update a task' })
  @ApiBody({ type: UpdateTaskDto })
  @ApiResponse({ status: 200, type: TaskResponseDto })
  async update(
    @Param('id') id: string,
    @Body() updateTaskDto: UpdateTaskDto,
    @Req() req: any,
  ): Promise<TaskResponseDto> {
    return this.tasksService.update(+id, updateTaskDto, req.user.id);
  }

  @Delete(':id')
  @ApiOperation({ summary: 'Delete a task' })
  @ApiResponse({ status: 204, description: 'Task deleted' })
  async remove(@Param('id') id: string, @Req() req: any): Promise<void> {
    return this.tasksService.remove(+id, req.user.id);
  }
}
```

## Instructions for Claude

When the user invokes this skill:

1. **Obtain the Swagger Spec**
   - If file path provided: Read it
   - If not provided: Ask user to paste spec or provide path
   - Extract: All endpoints, methods, request/response schemas

2. **Generate Unit Test Stubs (RED Phase)**
   - For each service method (not controller):
     - Create describe block for method name
     - Add setUp/tearDown hooks (beforeEach/afterEach)
     - Add test for happy path
     - Add tests for all validation errors
     - Add tests for edge cases (mentioned in requirements)
     - Add tests for error responses
   - Use Jest testing patterns:
     - Arrange-Act-Assert structure
     - Mock external dependencies (repository, services)
     - Use expect() assertions
   - Tests should FAIL when run initially (Red phase)
   - Mark as "TODO: Implement" where validation/logic needed

3. **Generate Request DTOs**
   - For each POST/PUT/PATCH request body:
     - Create class with @ApiProperty decorators
     - Add validation decorators (@IsEmail, @MinLength, @IsEnum, etc.)
     - Set @IsOptional for optional fields
     - Include descriptions for each field
     - Include validation rules in description (length, enum values)
   - Group related fields
   - Reuse common patterns (email, password, etc.)

4. **Generate Response DTOs**
   - For each response schema:
     - Create class with @ApiProperty decorators
     - Include all fields from schema
     - Mark sensitive fields as @Exclude if needed
     - Include descriptions for each field
     - Never expose passwords, hashes, or sensitive data
   - Create separate response DTOs for different contexts if needed
   - Ensure consistency (same field always returned same way)

5. **Generate Service Method Signatures**
   - For each endpoint:
     - Create corresponding service method
     - Define parameters (DTOs, user ID, filters, etc.)
     - Define return type (response DTO)
     - Add JSDoc comments with TODO implementing instructions
     - Include comments about business logic needed
   - Group related methods in same service
   - Keep services single-responsibility

6. **Generate Controller Method Stubs**
   - For each endpoint:
     - Create controller method
     - Add all @Api* decorators (Operation, Body, Response, etc.)
     - Extract parameters (@Body, @Param, @Query, @Req)
     - Call corresponding service method
     - Return response DTO
     - Add authentication guards (@UseGuards if needed)

7. **Structure the Output**
   - Organized by feature (tasks, users, auth, etc.)
   - File suggestions:
     - `src/modules/tasks/dto/create-task.dto.ts`
     - `src/modules/tasks/dto/update-task.dto.ts`
     - `src/modules/tasks/dto/task-response.dto.ts`
     - `src/modules/tasks/services/tasks.service.ts`
     - `src/modules/tasks/controllers/tasks.controller.ts`
     - `src/modules/tasks/services/tasks.service.spec.ts`

8. **Validate the Output**
   - All endpoints from Swagger are covered
   - All request/response schemas are defined
   - All validation rules are in DTOs
   - Tests cover happy path + error cases
   - Service methods have clear TODO comments
   - Controllers call services properly

9. **Provide Output** in organized sections:
   - **Section 1:** All test stubs (ready to copy to .spec.ts files)
   - **Section 2:** All DTOs (ready to copy to dto/ folder)
   - **Section 3:** Service method signatures (ready to copy to service.ts)
   - **Section 4:** Controller method stubs (ready to copy to controller.ts)
   - Each with file path suggestions

10. **Ask for Feedback**: "This is the RED phase - tests should fail when run. Ready to implement the GREEN phase (make tests pass)?"

## Example Interaction

```
User: /sdd-phase5-tdd

Claude: I'll generate TDD artifacts (test stubs, DTOs, method signatures) from your Swagger spec.

Please share your Swagger spec (YAML or paste):

[User shares swagger]

Claude: Perfect! Here's your TDD implementation guide:

# Task Management API - TDD Implementation (Phase 5)

## Test Stubs (RED Phase)

File: `src/modules/tasks/services/tasks.service.spec.ts`

```typescript
describe('TasksService', () => {
  ...
  describe('create', () => {
    it('should create a task with valid input', async () => {
      // Should FAIL initially - you implement the service
    });
  });
});
```

## Request/Response DTOs

File: `src/modules/tasks/dto/create-task.dto.ts`

```typescript
export class CreateTaskDto {
  @ApiProperty()
  @IsString()
  @MaxLength(200)
  title: string;
  ...
}
```

## Service Method Signatures

File: `src/modules/tasks/services/tasks.service.ts`

```typescript
@Injectable()
export class TasksService {
  async create(createTaskDto: CreateTaskDto, userId: number): Promise<TaskResponseDto> {
    // TODO: Implement
  }
  ...
}
```

## Controller Method Stubs

File: `src/modules/tasks/controllers/tasks.controller.ts`

```typescript
@Controller('tasks')
export class TasksController {
  @Post()
  async create(@Body() createTaskDto: CreateTaskDto, @Req() req: any): Promise<TaskResponseDto> {
    return this.tasksService.create(createTaskDto, req.user.id);
  }
  ...
}
```

## Next Steps

1. Copy tests to your project
2. Run `npm run test` - tests should FAIL (Red phase)
3. Implement service methods to make tests PASS (Green phase)
4. Refactor code as needed
5. All tests passing = ready for Phase 6 (Integration Testing)
```

## Tips for Best Results

- **Focus on behavior:** Test "what" not "how"
- **Happy path first:** Get main scenario working first
- **Then error cases:** Add validation error tests
- **Use descriptive names:** Test name should explain what's being tested
- **Mock dependencies:** Don't hit real database in unit tests
- **Keep assertions simple:** One assertion per test is ideal, max 2-3
- **DTOs are contracts:** Request DTOs validate input, response DTOs expose data

## Related Skills

After completing TDD:
- Next: `/sdd-phase6-integration` - Generate E2E integration tests
- Previous: `/sdd-phase4-swagger` - Generate Swagger spec
- Reference: [Spec-Driven Development Workflow](../../spec-driven-development-workflow.md) - Phase 5 details
