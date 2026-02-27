# Nest.js Core Concepts Guide

A focused guide to the 20% of Nest.js concepts used in 80% of real-world applications.

## 1. Modules

**What:** Organizing unit for your application code. Everything in Nest.js must belong to a module.

**Key Points:**
- A module is a class decorated with `@Module()`
- Modules group related functionality together
- The app requires a root `AppModule`
- Each module declares its `controllers`, `providers`, and `imports`

**Example:**
```typescript
@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService], // Makes service available to importing modules
})
export class UsersModule {}
```

**80/20 Tip:** Structure modules by feature (UsersModule, PostsModule, etc.). Keep modules cohesive.

---

## 2. Controllers

**What:** Handle incoming HTTP requests and return responses.

**Key Points:**
- Decorated with `@Controller('route')`
- Define route handlers with decorators: `@Get()`, `@Post()`, `@Put()`, `@Delete()`, `@Patch()`
- Extract data using `@Param()`, `@Query()`, `@Body()`, `@Req()`, `@Res()`
- Delegate business logic to services

**Example:**
```typescript
@Controller('users')
export class UsersController {
  constructor(private usersService: UsersService) {}

  @Get()
  findAll(@Query('role') role: string) {
    return this.usersService.findAll(role);
  }

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(+id);
  }
}
```

**80/20 Tip:** Controllers should be thin. Keep them focused on routing and request validation. Move logic to services.

---

## 3. Services (Providers)

**What:** Contain business logic and handle data operations.

**Key Points:**
- Decorated with `@Injectable()`
- Singleton by default (single instance for the entire app)
- Can be injected into controllers or other services
- Perfect place for database queries, external API calls, calculations

**Example:**
```typescript
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  create(createUserDto: CreateUserDto) {
    const user = this.usersRepository.create(createUserDto);
    return this.usersRepository.save(user);
  }

  findAll(role?: string) {
    if (role) {
      return this.usersRepository.find({ where: { role } });
    }
    return this.usersRepository.find();
  }

  findOne(id: number) {
    return this.usersRepository.findOneBy({ id });
  }
}
```

**80/20 Tip:** Services are where your app's logic lives. Keep them testable and focused.

---

## 4. Dependency Injection (DI)

**What:** Nest.js automatically provides dependencies without manual instantiation.

**Key Points:**
- Constructor-based injection is standard
- Declare dependencies in the constructor
- Nest resolves and injects them automatically
- Eliminates tight coupling and improves testability

**Example:**
```typescript
@Controller('posts')
export class PostsController {
  // Nest injects UsersService and PostsService automatically
  constructor(
    private usersService: UsersService,
    private postsService: PostsService,
  ) {}

  @Post()
  create(@Body() createPostDto: CreatePostDto) {
    return this.postsService.create(createPostDto);
  }
}
```

**80/20 Tip:** Always use constructor injection. It's clean and works seamlessly with Nest's DI container.

---

## 5. DTOs (Data Transfer Objects)

**What:** Define the shape of data coming in (and sometimes going out).

**Key Points:**
- Plain TypeScript classes (not decorated)
- Used for request body validation
- Often paired with class-validator decorators
- Improve type safety and API documentation

**Example:**
```typescript
import { IsString, IsEmail, MinLength } from 'class-validator';

export class CreateUserDto {
  @IsString()
  @MinLength(3)
  name: string;

  @IsEmail()
  email: string;

  @IsString()
  @MinLength(6)
  password: string;
}
```

**80/20 Tip:** Always use DTOs for request bodies. They provide validation, type safety, and documentation.

---

## 6. Pipes

**What:** Transform and validate data before it reaches your route handler.

**Key Points:**
- Decorated with `@Injectable()`
- Common pipes: `ValidationPipe`, `ParseIntPipe`, `ParseUUIDPipe`
- Can be applied globally, to a controller, or to a single parameter
- ValidationPipe automatically validates request bodies against DTOs

**Example:**
```typescript
// Global validation
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true, // Remove extra properties
    forbidNonWhitelisted: true, // Throw error if extra properties exist
    transform: true, // Auto-transform payloads
  }),
);

// Controller-level or param-level
@Controller('users')
export class UsersController {
  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    // ID is guaranteed to be a number
    return this.usersService.findOne(id);
  }
}
```

**80/20 Tip:** Use `ValidationPipe` globally with `whitelist` and `transform` options enabled.

---

## 7. Middleware

**What:** Functions that execute before route handlers. Used for logging, authentication, request preprocessing.

**Key Points:**
- Implemented as classes with `NestMiddleware` or functions
- Apply to routes using `forRoutes()`
- Execute in order: middleware → guards → pipes → interceptors → route handler
- Can be conditional with `exclude()`

**Example:**
```typescript
@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`[${new Date().toISOString()}] ${req.method} ${req.path}`);
    next();
  }
}

@Module({
  imports: [UsersModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes('*'); // or specific routes
  }
}
```

**80/20 Tip:** Middleware is great for logging and request timing. For auth, use Guards instead.

---

## 8. Guards

**What:** Determine whether a request should be handled by the route handler (authorization).

**Key Points:**
- Decorated with `@Injectable()` and implement `CanActivate`
- Execute after middleware but before pipes
- Return true (allow) or false (deny)
- Can be applied globally, to a controller, or to a specific route

**Example:**
```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.split(' ')[1];

    if (!token) {
      throw new UnauthorizedException('No token provided');
    }

    // Verify token
    return true;
  }
}

@Controller('posts')
export class PostsController {
  @Post()
  @UseGuards(AuthGuard)
  create(@Body() createPostDto: CreatePostDto) {
    return this.postsService.create(createPostDto);
  }
}
```

**80/20 Tip:** Use Guards for authentication and authorization checks.

---

## 9. Interceptors

**What:** Intercept requests/responses to transform them, log, or add functionality.

**Key Points:**
- Decorated with `@Injectable()` and implement `NestInterceptor`
- Can modify request before reaching handler and response before returning
- Great for logging, performance tracking, error handling
- Execute in order applied

**Example:**
```typescript
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler) {
    console.log('Before request handler');
    const now = Date.now();

    return next.handle().pipe(
      tap(() => {
        console.log(`After request completed in ${Date.now() - now}ms`);
      }),
    );
  }
}

// Apply globally
app.useGlobalInterceptors(new LoggingInterceptor());
```

**80/20 Tip:** Use interceptors for cross-cutting concerns: logging, error handling, response transformation.

---

## 10. Exception Filters

**What:** Catch exceptions and transform them into user-friendly HTTP responses.

**Key Points:**
- Decorated with `@Catch()` and implement `ExceptionFilter`
- Nest has built-in exceptions: `BadRequestException`, `UnauthorizedException`, `NotFoundException`, etc.
- Apply globally, to a controller, or to specific routes

**Example:**
```typescript
@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      message: exception.getResponse(),
      timestamp: new Date().toISOString(),
    });
  }
}

// Apply globally
app.useGlobalFilters(new HttpExceptionFilter());

// Or throw exceptions in your code
@Get(':id')
findOne(@Param('id') id: string) {
  const user = this.usersService.findOne(+id);
  if (!user) {
    throw new NotFoundException(`User ${id} not found`);
  }
  return user;
}
```

**80/20 Tip:** Use built-in exceptions. Apply a global exception filter to standardize error responses.

---

## 11. Decorators

**What:** Add metadata and functionality to classes and methods.

**Key Points:**
- `@Controller()`, `@Module()`, `@Injectable()` - define component types
- `@Get()`, `@Post()`, etc. - route handlers
- `@Param()`, `@Query()`, `@Body()` - extract request data
- `@UseGuards()`, `@UseInterceptors()`, `@UsePipes()` - apply middleware/guards/interceptors

**Example:**
```typescript
@Controller('users')
@UseGuards(AuthGuard)
@UseInterceptors(LoggingInterceptor)
export class UsersController {
  @Get(':id')
  @UseInterceptors(CacheInterceptor)
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findOne(id);
  }
}
```

**80/20 Tip:** Master the common decorators. They're the building blocks of every Nest.js app.

---

## 12. Configuration and Environment Variables

**What:** Manage app configuration based on environment (dev, prod, etc.).

**Key Points:**
- Use `@nestjs/config` package
- `.env` files for local development
- `process.env` for production
- Create a ConfigModule to centralize configuration
- Use strongly-typed config objects

**Example:**
```typescript
// .env
DATABASE_URL=postgresql://user:password@localhost/dbname
JWT_SECRET=your-secret-key
NODE_ENV=development

// config.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '.env',
    }),
  ],
})
export class ConfigModule {}

// In a service
@Injectable()
export class DatabaseService {
  constructor(private configService: ConfigService) {}

  connect() {
    const url = this.configService.get<string>('DATABASE_URL');
    // Connect to database
  }
}
```

**80/20 Tip:** Always use ConfigService for environment-specific values. Never hardcode secrets.

---

## 13. Database Integration (TypeORM)

**What:** Connect to and interact with databases using TypeORM.

**Key Points:**
- `@nestjs/typeorm` provides TypeORM integration
- Define entities as TypeScript classes with `@Entity()` decorator
- Use repositories to query entities
- Supports migrations for schema changes

**Example:**
```typescript
// user.entity.ts
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  email: string;

  @Column()
  password: string;

  @Column({ default: 'user' })
  role: string;
}

// app.module.ts
@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: 'localhost',
      port: 5432,
      username: 'user',
      password: 'password',
      database: 'nest_db',
      entities: [User],
      synchronize: true, // false in production
    }),
    TypeOrmModule.forFeature([User]),
  ],
  controllers: [AppController],
  providers: [AppService],
})
export class AppModule {}

// users.service.ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>,
  ) {}

  findAll() {
    return this.usersRepository.find();
  }
}
```

**80/20 Tip:** Use TypeORM repositories for clean, type-safe database queries.

---

## 14. Authentication & JWT

**What:** Verify user identity and maintain sessions.

**Key Points:**
- `@nestjs/jwt` and `@nestjs/passport` for JWT-based auth
- Strategy pattern for different auth methods (JWT, local, OAuth, etc.)
- JWT tokens are stateless and verified on each request
- Combine with Guards to protect routes

**Example:**
```typescript
// jwt.strategy.ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get<string>('JWT_SECRET'),
    });
  }

  validate(payload: any) {
    return { userId: payload.sub, username: payload.username };
  }
}

// auth.guard.ts
@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}

// auth.service.ts
@Injectable()
export class AuthService {
  constructor(
    private usersService: UsersService,
    private jwtService: JwtService,
  ) {}

  async login(user: any) {
    const payload = { username: user.username, sub: user.id };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}

// Protected route
@Controller('posts')
@UseGuards(JwtAuthGuard)
export class PostsController {
  @Get()
  findAll() {
    return this.postsService.findAll();
  }
}
```

**80/20 Tip:** Use JWT for stateless authentication. Strategies and Guards work well together.

---

## 15. Testing

**What:** Write unit tests for your services and controllers.

**Key Points:**
- Nest.js uses Jest by default
- Use `Test.createTestingModule()` for unit tests
- Mock dependencies to isolate what you're testing
- Test services and controllers separately

**Example:**
```typescript
describe('UsersService', () => {
  let service: UsersService;
  let mockRepository: jest.Mocked<Repository<User>>;

  beforeEach(async () => {
    mockRepository = {
      find: jest.fn(),
      findOne: jest.fn(),
      create: jest.fn(),
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

  it('should find all users', async () => {
    const users = [{ id: 1, email: 'test@test.com' }];
    mockRepository.find.mockResolvedValue(users);

    expect(await service.findAll()).toEqual(users);
  });
});
```

**80/20 Tip:** Test services thoroughly. Mock external dependencies (databases, APIs).

---

## Request/Response Lifecycle

Understanding the order of execution helps debug issues:

```
1. Middleware (in order)
2. Guards (in order)
3. Pipes (validation happens here)
4. Interceptors (before request)
5. Route Handler
6. Interceptors (after response)
7. Exception Filters (if exception thrown)
8. Response sent
```

---

## Quick Start Checklist

When building a new Nest.js feature:

- [ ] Create a module to organize the feature
- [ ] Add controllers to handle HTTP requests
- [ ] Create services for business logic
- [ ] Define DTOs for request validation
- [ ] Add guards for authentication/authorization if needed
- [ ] Use interceptors for cross-cutting concerns
- [ ] Add exception filters for error handling
- [ ] Write tests for services and controllers
- [ ] Connect to database using TypeORM if needed

---

## Key Takeaways

1. **Modules** organize your app; keep them feature-focused
2. **Controllers** are thin route handlers; logic goes in **Services**
3. **Dependency Injection** eliminates coupling and improves testing
4. **DTOs** and **Pipes** validate and transform data
5. **Guards** protect routes; **Middleware** logs and preprocesses requests
6. **Interceptors** handle cross-cutting concerns; **Filters** handle errors
7. **Database** integration via TypeORM keeps queries clean and type-safe
8. **Auth** via JWT and Strategies is stateless and secure
9. **Testing** your services keeps your app reliable
10. Master the **request lifecycle** to understand execution order

