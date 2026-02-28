# TypeScript Core Concepts

A practical guide to the 20% of TypeScript concepts that cover 80% of real-world development.

## 1. What is TypeScript?

TypeScript is a typed superset of JavaScript that compiles to plain JavaScript.

**Key Benefits:**
- Catch errors at compile time (not at runtime)
- Better IDE support and autocomplete
- Self-documenting code through types
- Easier refactoring with type safety
- Great for large codebases and teams

**Pareto Principle:**
- 20% of TypeScript features solve 80% of type problems
- Master the fundamentals; advanced features are optional
- Most code uses basic types, interfaces, and generics

---

## 2. Basic Types

The foundation of TypeScript. Master these first.

### Primitive Types

```typescript
// String
const name: string = 'John';
const message = `Hello, ${name}`; // Type inferred as string

// Number
const age: number = 25;
const price: number = 9.99;

// Boolean
const isActive: boolean = true;
const hasPermission: boolean = false;

// Null and Undefined
const nothing: null = null;
const notDefined: undefined = undefined;

// Any (avoid!)
let anything: any = 'could be anything';
anything = 42; // No type checking
// Use only when you have no other choice

// Unknown (better than any)
let value: unknown = 'string';
// Must type-guard before using
if (typeof value === 'string') {
  console.log(value.toUpperCase()); // Safe
}
```

### Type Inference

TypeScript infers types automatically when possible:

```typescript
// Inferred as string
const name = 'John';

// Inferred as number
const age = 25;

// Inferred as boolean
const isActive = true;

// Inferred from return type
function getUser() {
  return { id: 1, name: 'John' };
}
const user = getUser(); // Type: { id: number; name: string }

// 80/20 Tip: Let TypeScript infer types when obvious
// Only add explicit types when it helps clarity
```

### Type Annotations

```typescript
// Function parameters and return type
function add(a: number, b: number): number {
  return a + b;
}

// Function with object parameter
function getUserInfo(user: { id: number; name: string }): string {
  return `${user.name} (${user.id})`;
}

// Arrow function
const multiply = (a: number, b: number): number => a * b;

// Optional parameters (with ?)
function greet(name: string, greeting?: string): string {
  return `${greeting || 'Hello'}, ${name}`;
}

// Default parameters
function createUser(name: string, role: string = 'user'): void {
  console.log(`Created user: ${name} with role: ${role}`);
}
```

**80/20 Tip:** Type parameters and return types are most important. Variable types are often inferred.

---

## 3. Union Types

Allow a value to be one of multiple types.

```typescript
// Union of primitives
const status: 'active' | 'inactive' | 'pending' = 'active';

// Union of types
function processValue(value: string | number): void {
  if (typeof value === 'string') {
    console.log(value.toUpperCase());
  } else {
    console.log(value.toFixed(2));
  }
}

// Common patterns: value or null/undefined
function findUser(id: number): User | null {
  // Returns user or null
}

const user: User | undefined = getUser();

// Use type guard to narrow
if (user) {
  console.log(user.name); // Safe to access
}

// Discriminated unions (pattern matching)
type Response =
  | { status: 'success'; data: User }
  | { status: 'error'; message: string }
  | { status: 'loading' };

function handleResponse(response: Response) {
  if (response.status === 'success') {
    console.log(response.data); // Typed as User
  } else if (response.status === 'error') {
    console.log(response.message); // Typed as string
  }
  // response.status === 'loading' in else branch
}
```

**80/20 Tip:** Union types with type guards are powerful. Use them for optional values and state machines.

---

## 4. Interfaces

Define the shape of objects.

```typescript
// Basic interface
interface User {
  id: number;
  name: string;
  email: string;
}

const user: User = {
  id: 1,
  name: 'John',
  email: 'john@example.com',
};

// Optional properties (with ?)
interface User {
  id: number;
  name: string;
  email: string;
  phone?: string; // Optional
  nickname?: string; // Optional
}

// Readonly properties
interface User {
  readonly id: number; // Can't be changed after creation
  readonly createdAt: Date;
  name: string; // Can be changed
}

// Method signatures
interface UserService {
  getUser(id: number): User | null;
  createUser(name: string, email: string): User;
  deleteUser(id: number): boolean;
}

// Extending interfaces
interface Admin extends User {
  role: 'admin';
  permissions: string[];
}

const admin: Admin = {
  id: 1,
  name: 'John',
  email: 'john@example.com',
  role: 'admin',
  permissions: ['read', 'write', 'delete'],
};

// Multiple inheritance
interface Timestamped {
  createdAt: Date;
  updatedAt: Date;
}

interface Post extends User, Timestamped {
  title: string;
  content: string;
}
```

**80/20 Tip:** Interfaces are best for defining object shapes. Use them for classes and data structures.

---

## 5. Type Aliases

Define reusable types (similar to interfaces).

```typescript
// Simple type alias
type Status = 'active' | 'inactive' | 'pending';

// Union type alias
type ID = string | number;

// Object type alias
type User = {
  id: number;
  name: string;
  email: string;
};

// Function type alias
type UserHandler = (user: User) => void;
type APICall = (endpoint: string) => Promise<any>;

// Generic type alias
type Response<T> = {
  success: boolean;
  data: T;
  error?: string;
};

// Using generic type alias
const userResponse: Response<User> = {
  success: true,
  data: { id: 1, name: 'John', email: 'john@example.com' },
};

const usersResponse: Response<User[]> = {
  success: true,
  data: [{ id: 1, name: 'John', email: 'john@example.com' }],
};
```

**Interface vs Type Alias:**

| Feature | Interface | Type Alias |
|---------|-----------|-----------|
| Objects | ✅ Better | ✅ Good |
| Unions | ❌ No | ✅ Yes |
| Primitives | ❌ No | ✅ Yes |
| Generics | ✅ Yes | ✅ Yes |
| Extend | ✅ extends | ✅ & |
| Declaration merge | ✅ Yes | ❌ No |

**80/20 Tip:** Use interfaces for objects/classes. Use type aliases for unions and primitives.

---

## 6. Generics

Write reusable, flexible code while maintaining type safety.

```typescript
// Generic function
function getFirst<T>(array: T[]): T {
  return array[0];
}

const firstNumber = getFirst<number>([1, 2, 3]); // Type: number
const firstName = getFirst<string>(['John', 'Jane']); // Type: string
const first = getFirst([1, 2, 3]); // Type inferred: number

// Generic with constraints
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { name: 'John', age: 25 };
const name = getProperty(user, 'name'); // Type: string
// getProperty(user, 'email'); // Error: 'email' is not a property

// Generic interface
interface Container<T> {
  item: T;
  getItem(): T;
  setItem(item: T): void;
}

const stringContainer: Container<string> = {
  item: 'hello',
  getItem() {
    return this.item;
  },
  setItem(item) {
    this.item = item;
  },
};

// Generic class
class Stack<T> {
  private items: T[] = [];

  push(item: T): void {
    this.items.push(item);
  }

  pop(): T | undefined {
    return this.items.pop();
  }

  isEmpty(): boolean {
    return this.items.length === 0;
  }
}

const numberStack = new Stack<number>();
numberStack.push(1);
numberStack.push(2);
const popped = numberStack.pop(); // Type: number | undefined

// Generic with default
type Result<T = string> = {
  success: boolean;
  data: T;
};

const stringResult: Result = { success: true, data: 'hello' }; // T defaults to string
const numberResult: Result<number> = { success: true, data: 42 };

// Generic with multiple constraints
function merge<T extends object, U extends object>(
  obj1: T,
  obj2: U,
): T & U {
  return { ...obj1, ...obj2 };
}

const merged = merge({ a: 1 }, { b: 2 }); // Type: { a: number } & { b: number }
```

**80/20 Tip:** Generics are essential for reusable components. Master them for arrays, containers, API responses.

---

## 7. Classes

Object-oriented programming with TypeScript.

```typescript
// Basic class
class User {
  id: number;
  name: string;
  email: string;

  constructor(id: number, name: string, email: string) {
    this.id = id;
    this.name = name;
    this.email = email;
  }

  displayInfo(): string {
    return `${this.name} (${this.email})`;
  }
}

const user = new User(1, 'John', 'john@example.com');
console.log(user.displayInfo());

// Access modifiers
class SecureUser {
  public id: number; // Accessible everywhere
  private password: string; // Only within class
  protected role: string; // Within class and subclasses
  readonly createdAt: Date; // Can't be changed after creation

  constructor(id: number, password: string, role: string) {
    this.id = id;
    this.password = password;
    this.role = role;
    this.createdAt = new Date();
  }

  checkPassword(pwd: string): boolean {
    return this.password === pwd;
  }
}

// Constructor shorthand
class ShorthandUser {
  constructor(
    public id: number,
    public name: string,
    private password: string,
  ) {}

  checkPassword(pwd: string): boolean {
    return this.password === pwd;
  }
}

const shortUser = new ShorthandUser(1, 'John', 'secret');
console.log(shortUser.id); // 1
console.log(shortUser.name); // 'John'
// console.log(shortUser.password); // Error: private

// Inheritance
class Admin extends SecureUser {
  permissions: string[];

  constructor(id: number, password: string, permissions: string[]) {
    super(id, password, 'admin');
    this.permissions = permissions;
  }

  hasPermission(permission: string): boolean {
    return this.permissions.includes(permission);
  }
}

// Static members
class Logger {
  static instance: Logger;

  static getInstance(): Logger {
    if (!Logger.instance) {
      Logger.instance = new Logger();
    }
    return Logger.instance;
  }

  log(message: string): void {
    console.log(`[LOG] ${message}`);
  }
}

const logger = Logger.getInstance();
logger.log('Application started');

// Abstract class
abstract class Animal {
  name: string;

  constructor(name: string) {
    this.name = name;
  }

  abstract makeSound(): void;

  sleep(): void {
    console.log(`${this.name} is sleeping`);
  }
}

class Dog extends Animal {
  makeSound(): void {
    console.log('Woof!');
  }
}

const dog = new Dog('Buddy');
dog.makeSound(); // Woof!
dog.sleep(); // Buddy is sleeping
```

**80/20 Tip:** Use access modifiers (public, private, protected) to encapsulate data. Classes are great for services and models.

---

## 8. Enums

Define a set of named constants.

```typescript
// Numeric enum
enum Status {
  Active = 0,
  Inactive = 1,
  Pending = 2,
}

const userStatus: Status = Status.Active;

// String enum (preferred)
enum Role {
  Admin = 'admin',
  User = 'user',
  Guest = 'guest',
}

const userRole: Role = Role.User;

// Computed enum
enum FileSize {
  Small = 1,
  Medium = Small * 10,
  Large = Medium * 10,
}

// Using enums
function isAdmin(role: Role): boolean {
  return role === Role.Admin;
}

// Switch with enums
function getPermissions(role: Role): string[] {
  switch (role) {
    case Role.Admin:
      return ['read', 'write', 'delete'];
    case Role.User:
      return ['read', 'write'];
    case Role.Guest:
      return ['read'];
  }
}

// Constant enum (no object generated at runtime)
const enum Direction {
  Up = 'UP',
  Down = 'DOWN',
  Left = 'LEFT',
  Right = 'RIGHT',
}

const direction: Direction = Direction.Up;
```

**80/20 Tip:** Use string enums for clarity. Enums are great for status, roles, and fixed sets of values.

---

## 9. Utility Types

Built-in types for common patterns.

### Partial<T> - Make all properties optional

```typescript
interface User {
  id: number;
  name: string;
  email: string;
}

type PartialUser = Partial<User>; // All properties optional

function updateUser(id: number, updates: Partial<User>): void {
  // Can update any combination of properties
}

updateUser(1, { name: 'Jane' }); // Valid
updateUser(1, { email: 'jane@example.com' }); // Valid
```

### Required<T> - Make all properties required

```typescript
interface User {
  id: number;
  name: string;
  email?: string;
}

type FullUser = Required<User>; // All properties required, including email

const user: FullUser = {
  id: 1,
  name: 'John',
  email: 'john@example.com', // Must provide
};
```

### Readonly<T> - Make all properties readonly

```typescript
interface User {
  id: number;
  name: string;
}

type ReadonlyUser = Readonly<User>;

const user: ReadonlyUser = { id: 1, name: 'John' };
// user.name = 'Jane'; // Error: readonly
```

### Pick<T, K> - Select specific properties

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

type UserPreview = Pick<User, 'id' | 'name'>; // Only id and name
type PublicUser = Pick<User, 'id' | 'name' | 'email'>; // No password

const preview: UserPreview = { id: 1, name: 'John' };
```

### Omit<T, K> - Exclude specific properties

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
}

type UserWithoutPassword = Omit<User, 'password'>;
type CreateUserInput = Omit<User, 'id'>; // No id, user creates it

const input: CreateUserInput = {
  name: 'John',
  email: 'john@example.com',
  password: 'secret',
};
```

### Record<K, T> - Create object with specific keys

```typescript
type Status = 'active' | 'inactive' | 'pending';

type StatusConfig = Record<Status, {
  label: string;
  color: string;
}>;

const statusConfig: StatusConfig = {
  active: { label: 'Active', color: 'green' },
  inactive: { label: 'Inactive', color: 'gray' },
  pending: { label: 'Pending', color: 'yellow' },
};
```

### Exclude<T, U> - Remove types from union

```typescript
type Status = 'active' | 'inactive' | 'pending';

type EditableStatus = Exclude<Status, 'pending'>; // 'active' | 'inactive'

function editStatus(status: EditableStatus): void {
  // Can only edit active or inactive
}
```

### Extract<T, U> - Keep only matching types

```typescript
type Status = 'active' | 'inactive' | 'pending';

type QuickStatus = Extract<Status, 'active' | 'inactive'>; // 'active' | 'inactive'
```

**80/20 Tip:** `Pick`, `Omit`, `Partial`, and `Record` are most useful. Master these four.

---

## 10. Type Assertion (Casting)

Tell TypeScript about types it can't infer.

```typescript
// Using 'as' keyword
const value: unknown = 'hello';
const stringValue: string = value as string;

// Using angle bracket syntax (less common in React)
const stringValue2 = <string>value;

// Casting to more specific type
const element = document.getElementById('app') as HTMLDivElement;
element.style.backgroundColor = 'blue';

// Casting with generic
const items = JSON.parse('[1, 2, 3]') as number[];

// Double assertion (dangerous, only when necessary)
const value2: unknown = 42;
const stringValue3: string = (value2 as any) as string;

// 80/20 Tip: Use sparingly. Prefer type guards. Excessive use defeats type safety.
```

**Type Guards (Safer):**

```typescript
// Check typeof
if (typeof value === 'string') {
  console.log(value.toUpperCase()); // Safe
}

// Check instanceof
if (element instanceof HTMLDivElement) {
  element.style.backgroundColor = 'blue'; // Safe
}

// Custom type guard
function isUser(obj: any): obj is User {
  return (
    typeof obj === 'object' &&
    typeof obj.id === 'number' &&
    typeof obj.name === 'string'
  );
}

if (isUser(value)) {
  console.log(value.name); // Safe
}
```

---

## 11. Decorators

Add metadata and functionality to classes and methods (experimental feature).

```typescript
// Class decorator
function Component(config: { name: string }) {
  return function <T extends { new (...args: any[]): {} }>(
    constructor: T,
  ) {
    return class extends constructor {
      name = config.name;
    };
  };
}

@Component({ name: 'UserComponent' })
class UserList {
  title = 'Users';
}

// Method decorator
function Logged(
  target: any,
  propertyKey: string,
  descriptor: PropertyDescriptor,
) {
  const originalMethod = descriptor.value;

  descriptor.value = function (...args: any[]) {
    console.log(`Calling ${propertyKey}`);
    return originalMethod.apply(this, args);
  };

  return descriptor;
}

class Calculator {
  @Logged
  add(a: number, b: number): number {
    return a + b;
  }
}

// Property decorator
function Required(target: any, propertyKey: string) {
  // Mark property as required
}

class User {
  @Required
  name: string;
}
```

**80/20 Tip:** Decorators are experimental. Most projects don't need them. Good for frameworks (Angular, NestJS) but avoid in libraries.

---

## 12. Module System

Import and export types and code.

```typescript
// Named exports
export interface User {
  id: number;
  name: string;
}

export class UserService {
  getUser(id: number): User {
    return { id, name: 'John' };
  }
}

export const DEFAULT_USER: User = { id: 0, name: 'Guest' };

// Default export
export default UserService;

// Re-export
export { User, UserService } from './user.service';
export type { User as IUser }; // Export only type

// Importing
import UserService, { User, DEFAULT_USER } from './user.service';
import type { User as IUser } from './user.service'; // Import only type

// Import all
import * as UserModule from './user.service';
const service = new UserModule.default();
```

**80/20 Tip:** Use named exports for most things. Default exports for main class/function. Use `export type` for types.

---

## 13. Async/Await with Types

Typed promises and async functions.

```typescript
// Promise with generic
function fetchUser(id: number): Promise<User> {
  return new Promise((resolve, reject) => {
    if (id > 0) {
      resolve({ id, name: 'John' });
    } else {
      reject(new Error('Invalid ID'));
    }
  });
}

// Async function
async function getUser(id: number): Promise<User> {
  try {
    const user = await fetchUser(id);
    return user;
  } catch (error) {
    console.error(error);
    throw error;
  }
}

// Type for async function with error handling
async function loadUsers(): Promise<User[]> {
  const response = await fetch('/api/users');
  const users: User[] = await response.json();
  return users;
}

// Async generator
async function* fetchUsersPaginated(
  pageSize: number = 10,
): AsyncGenerator<User[], void, unknown> {
  let page = 1;
  while (true) {
    const response = await fetch(`/api/users?page=${page}&limit=${pageSize}`);
    const users: User[] = await response.json();
    if (users.length === 0) break;
    yield users;
    page++;
  }
}
```

**80/20 Tip:** Always type promise return values. Use async/await, not .then() for cleaner code.

---

## 14. Common Patterns

Real-world patterns you'll use frequently.

### Pattern 1: API Response Handler

```typescript
type ApiResponse<T> =
  | { status: 'success'; data: T }
  | { status: 'error'; message: string }
  | { status: 'loading' };

async function fetchUsers(): Promise<ApiResponse<User[]>> {
  try {
    const response = await fetch('/api/users');
    const data = await response.json();
    return { status: 'success', data };
  } catch (error) {
    return { status: 'error', message: String(error) };
  }
}
```

### Pattern 2: Generic Service

```typescript
class Repository<T> {
  constructor(private endpoint: string) {}

  async getAll(): Promise<T[]> {
    const response = await fetch(`/api/${this.endpoint}`);
    return response.json();
  }

  async getById(id: number): Promise<T | null> {
    const response = await fetch(`/api/${this.endpoint}/${id}`);
    return response.ok ? response.json() : null;
  }

  async create(item: Omit<T, 'id'>): Promise<T> {
    const response = await fetch(`/api/${this.endpoint}`, {
      method: 'POST',
      body: JSON.stringify(item),
    });
    return response.json();
  }
}

const userRepository = new Repository<User>('users');
const users = await userRepository.getAll();
```

### Pattern 3: React Component Props

```typescript
interface ComponentProps {
  title: string;
  isLoading?: boolean;
  onSubmit: (data: FormData) => void;
  children?: React.ReactNode;
}

function MyComponent({
  title,
  isLoading = false,
  onSubmit,
  children,
}: ComponentProps): React.ReactElement {
  return (
    <div>
      <h1>{title}</h1>
      {isLoading && <Spinner />}
      {children}
    </div>
  );
}
```

### Pattern 4: Type-Safe Event Handler

```typescript
type EventHandler = {
  onSuccess: (data: unknown) => void;
  onError: (error: Error) => void;
};

function handleEvent(event: string, handlers: EventHandler): void {
  // Handlers properly typed
}
```

---

## 15. Strict Mode

Enable TypeScript's strictest checking.

```json
{
  "compilerOptions": {
    "strict": true, // Enables all strict options
    "noImplicitAny": true, // Error on implicit any
    "strictNullChecks": true, // null/undefined checking
    "strictFunctionTypes": true, // Strict function types
    "noImplicitThis": true, // Error on implicit this
    "alwaysStrict": true, // "use strict" in output
    "noUnusedLocals": true, // Error on unused variables
    "noUnusedParameters": true, // Error on unused parameters
    "noImplicitReturns": true, // Error on missing returns
    "noFallthroughCasesInSwitch": true // Error on missing breaks
  }
}
```

**80/20 Tip:** Enable `strict: true` in tsconfig.json. It catches most bugs automatically.

---

## 16. Type Checking Tips

Practical advice for effective type checking.

### Tip 1: Return Types Over Inference

```typescript
// ❌ Less clear
function calculate(a: number, b: number) {
  return a + b;
}

// ✅ Better: explicit return type
function calculate(a: number, b: number): number {
  return a + b;
}
```

### Tip 2: Type Guard for null/undefined

```typescript
// ❌ Risky
function getName(user: User | null): string {
  return user.name; // Error if user is null
}

// ✅ Better: check first
function getName(user: User | null): string | null {
  return user ? user.name : null;
}

// Or use optional chaining
function getName(user: User | null): string | undefined {
  return user?.name;
}
```

### Tip 3: Use Const Assertions for Literals

```typescript
// ❌ Inferred as string
const status = 'active';

// ✅ Better: keep literal type
const status = 'active' as const;

// Use in union
type Status = typeof status; // Type: 'active'
```

### Tip 4: Avoid Type Assertions When Possible

```typescript
// ❌ Type assertion (dangerous)
const value = JSON.parse(data) as User;

// ✅ Better: validate first
function parseUser(data: string): User {
  const parsed = JSON.parse(data);
  if (!isUser(parsed)) {
    throw new Error('Invalid user data');
  }
  return parsed;
}
```

---

## 17. Best Practices

### Do ✅

- **Enable strict mode** - Catches more errors
- **Type function parameters and returns** - Most important
- **Use interfaces/types** - Document your API
- **Prefer type inference** - For variables
- **Use utility types** - Reduce duplication
- **Write type guards** - For runtime safety
- **Keep types DRY** - Reuse with generics and inheritance
- **Test your types** - Use type assertions to verify

### Don't ❌

- **Overuse `any`** - Use `unknown` instead
- **Avoid type annotations** - Types help, don't hurt
- **Use type assertions excessively** - Defeats the purpose
- **Forget about null/undefined** - Check explicitly
- **Create overly complex types** - Keep it readable
- **Disable strict checks** - They're there to help
- **Ignore compiler warnings** - Fix them
- **Forget to export types** - Others need them

---

## 18. Quick Reference

### Essential Types

```typescript
// Primitives
string, number, boolean, null, undefined, unknown, any

// Collections
string[], Array<string>, Record<string, number>, Map<K, V>

// Unions
string | number, 'active' | 'inactive'

// Objects
{ prop: string }, interface, type

// Functions
(arg: string) => void, (arg: string) => string

// Generics
<T>, <T extends something>

// Utility Types
Partial<T>, Required<T>, Pick<T, K>, Omit<T, K>, Record<K, V>
```

### File Structure

```
src/
├── types/
│   ├── user.ts       # User-related types
│   ├── api.ts        # API response types
│   └── index.ts      # Re-export all types
├── services/
│   └── user.service.ts
└── components/
    └── User.tsx
```

### Configuration

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

---

## Key Takeaways

1. **Type safety prevents bugs** - Catch errors at compile time
2. **Master primitives first** - string, number, boolean, null, undefined
3. **Use interfaces for shapes** - Define object structures
4. **Use type aliases for unions** - For variants and options
5. **Generics are powerful** - Reuse code with type safety
6. **Utility types save time** - Partial, Pick, Omit, Record
7. **Type inference helps** - Let TypeScript figure it out
8. **Return types matter most** - Always type function returns
9. **Enable strict mode** - It catches more errors
10. **Use type guards** - Check types at runtime when needed
11. **Classes are great for services** - Encapsulate functionality
12. **Enums for constants** - Fixed sets of values
13. **Decorators optional** - Use only in frameworks
14. **Async/await with typing** - Always type promises
15. **Keep types DRY** - Reuse with generics

---

## Common Errors & Solutions

### Error: Object is possibly 'undefined'

```typescript
// ❌ Error
const user: User | undefined = getUser();
console.log(user.name);

// ✅ Solution 1: Check
if (user) {
  console.log(user.name);
}

// ✅ Solution 2: Optional chaining
console.log(user?.name);

// ✅ Solution 3: Nullish coalescing
console.log(user?.name ?? 'Unknown');
```

### Error: Type 'X' is not assignable to type 'Y'

```typescript
// ❌ Error
const value: string = 42;

// ✅ Solution: Convert type
const value: string = String(42);
const value: number = 42;
```

### Error: Cannot find name 'Type'

```typescript
// ❌ Error (forgot export)
// user.ts: interface User { ... }
import { User } from './user'; // Error if not exported

// ✅ Solution: Export type
// user.ts: export interface User { ... }
import { User } from './user'; // Works
import type { User } from './user'; // Also works
```

