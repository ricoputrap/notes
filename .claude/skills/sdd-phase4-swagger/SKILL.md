---
name: sdd-phase4-swagger
description: Generate complete Swagger/OpenAPI 3.0 specification with NestJS decorators for API contracts
argument-hint: [design-file-path]
user-invocable: true
---

# Phase 4: Swagger/OpenAPI Doc Generation Skill

Generate complete Swagger/OpenAPI 3.0 specification with NestJS decorators for your API contract.

## How to Use This Skill

**Invoke manually:**
```
/sdd-phase4-swagger
/sdd-phase4-swagger path/to/design-doc.md
```

**What AI will do:**
1. Read your Design Doc
2. Generate complete OpenAPI 3.0 YAML specification
3. Generate NestJS controller decorators with @Api* annotations
4. Generate request/response schemas
5. Include error responses and examples
6. Provide output ready for frontend teams to mock and Phase 5 implementation

## What Constitutes Good Swagger Docs

Your Swagger spec should:
- **Be complete:** Every endpoint, parameter, and response documented
- **Be testable:** Can import into Swagger Editor and test interactively
- **Have examples:** Request/response examples for each endpoint
- **Be consistent:** Same patterns across all endpoints
- **Include errors:** Document what can go wrong
- **Be implementable:** Developers can code against it

## Output Format

The generated Swagger will include:

### 1. OpenAPI 3.0 YAML Specification
```yaml
openapi: 3.0.0
info:
  title: Task Management API
  version: 1.0.0
  description: REST API for managing tasks and team coordination

paths:
  /auth/register:
    post:
      summary: Register a new user
      tags:
        - Authentication
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RegisterRequest'
      responses:
        '201':
          description: User created successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/AuthResponse'
        '400':
          description: Validation failed
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
        '409':
          description: Email already registered

components:
  schemas:
    RegisterRequest:
      type: object
      properties:
        email:
          type: string
          format: email
        password:
          type: string
          minLength: 8
      required:
        - email
        - password

    AuthResponse:
      type: object
      properties:
        id:
          type: integer
        email:
          type: string
        token:
          type: string

    ErrorResponse:
      type: object
      properties:
        status:
          type: integer
        message:
          type: string
        errors:
          type: array
```

### 2. NestJS Controller Decorators
```typescript
@Controller('auth')
@ApiTags('Authentication')
export class AuthController {
  @Post('register')
  @ApiOperation({ summary: 'Register a new user' })
  @ApiBody({ type: RegisterDto })
  @ApiResponse({
    status: 201,
    description: 'User created successfully',
    type: AuthResponseDto,
  })
  @ApiResponse({
    status: 400,
    description: 'Validation failed',
    type: ErrorResponseDto,
  })
  @ApiResponse({
    status: 409,
    description: 'Email already registered',
  })
  register(@Body() registerDto: RegisterDto) {
    // Implementation
  }

  @Post('login')
  @ApiOperation({ summary: 'Login with email and password' })
  @ApiBody({ type: LoginDto })
  @ApiResponse({ status: 200, type: AuthResponseDto })
  @ApiResponse({ status: 401, description: 'Invalid credentials' })
  login(@Body() loginDto: LoginDto) {
    // Implementation
  }
}
```

### 3. Request/Response Schemas
```typescript
// Request DTOs
export class RegisterDto {
  @ApiProperty({ description: 'User email address' })
  @IsEmail()
  email: string;

  @ApiProperty({ description: 'Password (8+ chars, uppercase, number, special)' })
  @MinLength(8)
  password: string;
}

// Response DTOs
export class AuthResponseDto {
  @ApiProperty()
  id: number;

  @ApiProperty()
  email: string;

  @ApiProperty()
  token: string;

  @ApiProperty()
  expiresIn: number;
}
```

## Instructions for Claude

When the user invokes this skill:

1. **Obtain the Design Doc**
   - If file path provided: Read it
   - If not provided: Ask user to paste or provide path
   - Extract: Endpoints, methods, parameters, responses, errors

2. **Generate OpenAPI 3.0 YAML**
   - Create info section (title, version, description)
   - Create paths section with all endpoints
   - For each endpoint:
     - Summary and description
     - Tags for grouping
     - Request body with schema reference
     - All possible responses (201, 400, 401, 404, 409, 500)
     - Examples for happy path and error cases
   - Create components/schemas section with:
     - All request DTOs
     - All response DTOs
     - Error response format
     - Shared schemas (pagination, timestamps, etc.)

3. **Generate NestJS Controller Decorators**
   - For each endpoint, create:
     - @Controller('route') on class
     - @ApiTags for grouping
     - @Get/@Post/@Put/@Delete for method
     - @ApiOperation for summary/description
     - @ApiBody for request schema
     - @ApiResponse for each possible response
     - @ApiParam/@ApiQuery for parameters
   - Include all validation rules in decorators
   - Include example values

4. **Generate Request/Response DTOs**
   - Create DTO class for each unique request
   - Create DTO class for each unique response
   - Use @ApiProperty decorator on each field
   - Include field descriptions
   - Include validation rules (@IsEmail, @MinLength, etc.)
   - Apply transformations (@Type, @Exclude, etc.)

5. **Document Error Responses**
   - For each endpoint, list possible errors:
     - 400 Bad Request (validation failures)
     - 401 Unauthorized (not authenticated)
     - 403 Forbidden (not permitted)
     - 404 Not Found (resource missing)
     - 409 Conflict (duplicate, state conflict)
     - 500 Internal Server Error
   - Show error response format
   - Include error message examples

6. **Include Examples**
   - For each endpoint:
     - Example request with sample data
     - Example success response
     - Example error response
   - Use realistic values (emails, dates, etc.)
   - Include multiple status codes

7. **Validate the Swagger**
   - Can it be imported into Swagger Editor? (Valid YAML)
   - Are all endpoints documented? (Compare with design doc)
   - Are all parameters documented? (Query, path, body)
   - Are all responses documented? (Success and errors)
   - Are schemas complete? (All fields described)
   - Are decorators correct? (Match NestJS patterns)

8. **Provide Output** as two separate sections:
   - **Section 1:** Complete OpenAPI 3.0 YAML (copyable, importable)
   - **Section 2:** NestJS controller decorators with DTOs (copyable, usable)
   - Both ready to commit to repo

9. **Ask for Feedback**: "Does this Swagger spec match your design? Can you copy this to Swagger Editor to visualize? Ready for Phase 5 (TDD Implementation)?"

## Example Interaction

```
User: /sdd-phase4-swagger

Claude: I'll generate a complete Swagger spec from your Design Doc.

Please share your Design Doc (paste or provide file path):

[User shares design]

Claude: Perfect! Here's your Swagger specification:

# Task Management API - Swagger/OpenAPI 3.0

## OpenAPI 3.0 Specification

```yaml
openapi: 3.0.0
info:
  title: Task Management API
  version: 1.0.0
...
```

Copy this YAML to https://editor.swagger.io to visualize.

## NestJS Controller Decorators

```typescript
@Controller('auth')
@ApiTags('Authentication')
export class AuthController {
  ...
}
```

## Request/Response DTOs

```typescript
export class RegisterDto { ... }
export class AuthResponseDto { ... }
```

**Next steps:**
1. Copy YAML to Swagger Editor to review visually
2. Verify all endpoints are documented
3. Frontend teams can start mocking based on this spec
4. Ready for Phase 5: TDD Implementation
```

## Tips for Best Results

- **Be consistent:** All endpoints follow same patterns
- **Include examples:** Sample data for every endpoint
- **Document errors:** What can realistically go wrong?
- **Use descriptive names:** Field names should be self-explanatory
- **Tag for organization:** Group endpoints by feature (auth, tasks, users)
- **Test in Swagger Editor:** Paste YAML to https://editor.swagger.io
- **NestJS consistency:** Use decorators correctly for auto-doc generation

## Useful Tools

- **Swagger Editor:** https://editor.swagger.io - Paste YAML to visualize
- **OpenAPI Validator:** Validate your YAML is correct
- **Swagger Codegen:** Generate client SDKs from your spec
- **NestJS Swagger:** `npm install @nestjs/swagger swagger-ui-express`

## Related Skills

After completing Swagger:
- Next: `/sdd-phase5-tdd` - Generate tests for TDD implementation
- Previous: `/sdd-phase3.5-quantitative` - Quantitative Analysis
- Reference: [Spec-Driven Development Workflow](../../spec-driven-development-workflow.md) - Phase 4 details
