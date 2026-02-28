# Authentication Core Concepts

**The 20% of authentication patterns and techniques used in 80% of real-world applications.**

This guide focuses on practical authentication concepts that cover the majority of use cases. We'll skip academic theory and focus on what works in production.

---

## Table of Contents

1. [The Three Core Patterns](#the-three-core-patterns)
2. [Session-Based Authentication](#session-based-authentication)
3. [Token-Based Authentication (JWT)](#token-based-authentication-jwt)
4. [OAuth 2.0 and Social Login](#oauth-20-and-social-login)
5. [Password Security](#password-security)
6. [Authorization (Access Control)](#authorization-access-control)
7. [Refresh Token Pattern](#refresh-token-pattern)
8. [Common Implementation Patterns](#common-implementation-patterns)

---

## The Three Core Patterns

There are three authentication approaches that cover 95% of all web applications:

| Pattern | Best For | State |
|---------|----------|-------|
| **Session-Based** | Server-rendered apps, simple projects | Stateful (server) |
| **JWT (Token-Based)** | SPAs, mobile apps, microservices | Stateless (client) |
| **OAuth 2.0** | Social login, third-party integration | Delegated (external) |

Most modern applications use **JWT** or **OAuth 2.0**. Session-based is still valid but less common in new projects.

---

## Session-Based Authentication

### How It Works

```
1. User submits username/password
2. Server validates credentials
3. Server creates session and stores in memory/database
4. Server sends session ID as HTTP-only cookie
5. Client automatically sends cookie with each request
6. Server validates session ID and allows/denies access
7. User logs out → server deletes session
```

### Key Characteristics

- **Stateful**: Server must remember all active sessions
- **Automatic**: Cookie is sent automatically by browser
- **Secure by default**: HTTP-only cookies prevent XSS attacks
- **Server-side logout**: Session can be immediately invalidated

### Simple Example (Express.js)

```javascript
// Setup
const session = require('express-session');
const store = new MemoryStore(); // or use database store

app.use(session({
  secret: process.env.SESSION_SECRET,
  store,
  cookie: {
    httpOnly: true,        // Cannot access via JavaScript
    secure: true,          // Only over HTTPS
    sameSite: 'strict',    // CSRF protection
    maxAge: 24 * 60 * 60 * 1000 // 24 hours
  }
}));

// Login
app.post('/login', (req, res) => {
  const user = validateCredentials(req.body.username, req.body.password);
  if (user) {
    req.session.userId = user.id;
    res.send({ success: true });
  } else {
    res.status(401).send({ error: 'Invalid credentials' });
  }
});

// Protected route
app.get('/profile', (req, res) => {
  if (!req.session.userId) {
    return res.status(401).send({ error: 'Not authenticated' });
  }
  res.send({ userId: req.session.userId });
});

// Logout
app.post('/logout', (req, res) => {
  req.session.destroy();
  res.send({ success: true });
});
```

### When to Use

- Server-rendered applications (Django, Rails, traditional Node.js)
- Simple authentication needs (no mobile apps)
- Maximum security with minimal complexity

---

## Token-Based Authentication (JWT)

### How It Works

```
1. User submits username/password
2. Server validates credentials
3. Server generates JWT token (contains user info + signature)
4. Server sends token to client
5. Client stores token (localStorage, sessionStorage, or memory)
6. Client sends token in Authorization header: "Bearer <token>"
7. Server verifies token signature and extracts user info
8. User logs out → client deletes token (server doesn't need to do anything)
```

### JWT Structure

```
Header.Payload.Signature

header:    { "alg": "HS256", "typ": "JWT" }
payload:   { "userId": 123, "email": "user@example.com", "iat": 1516239022, "exp": 1516325422 }
signature: HMACSHA256(base64(header) + "." + base64(payload), secret)
```

### Key Characteristics

- **Stateless**: Server doesn't store anything (no session database needed)
- **Self-contained**: Token contains user info, no server lookup needed
- **Portable**: Works across domains and services (microservices)
- **Mobile-friendly**: Works with any HTTP client
- **No built-in logout**: Must implement token blacklist for immediate logout

### Simple Example (NestJS)

```typescript
// Generate token
import { sign } from 'jsonwebtoken';

const generateToken = (userId: string) => {
  return sign(
    { userId, email: 'user@example.com' },
    process.env.JWT_SECRET,
    { expiresIn: '24h' }
  );
};

// Login endpoint
@Post('/login')
async login(@Body() body: LoginDto) {
  const user = await this.validateCredentials(body.username, body.password);
  if (!user) throw new UnauthorizedException();

  return {
    accessToken: generateToken(user.id),
    userId: user.id
  };
}

// Protected endpoint with guard
@UseGuards(JwtAuthGuard)
@Get('/profile')
getProfile(@Request() req) {
  // req.user is set by the guard from token
  return { userId: req.user.userId };
}

// JWT Guard
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization?.replace('Bearer ', '');

    if (!token) return false;

    try {
      const decoded = this.jwtService.verify(token);
      request.user = decoded;
      return true;
    } catch {
      return false;
    }
  }
}
```

### When to Use

- Single Page Applications (React, Vue, Angular)
- Mobile applications
- Microservices architecture
- APIs consumed by multiple clients
- **Most modern applications use JWT**

---

## OAuth 2.0 and Social Login

### How It Works (Google/GitHub Login)

```
1. User clicks "Login with Google"
2. App redirects to Google's authorization server
3. User logs in to Google (if not already)
4. Google asks user for permission ("This app wants access to your profile")
5. User approves
6. Google redirects back to app with authorization code
7. App exchanges code for access token (server-to-server)
8. App uses token to fetch user info from Google
9. App creates session/JWT for the user
10. User is now logged in
```

### Key Concepts

- **Resource Owner**: The user
- **Client**: Your application
- **Authorization Server**: Google, GitHub, etc.
- **Resource Server**: Provides user data (part of authorization server)
- **Authorization Code**: Single-use code (secure, only used server-to-server)
- **Access Token**: Used to access user data

### Simple Example (NestJS with Passport)

```typescript
// Strategy definition
import { Strategy } from 'passport-google-oauth20';

@Injectable()
export class GoogleStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      clientID: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
      callbackURL: 'http://localhost:3000/auth/google/callback',
      scope: ['profile', 'email'],
    });
  }

  async validate(accessToken: string, refreshToken: string, profile: any) {
    // Called after successful OAuth redirect
    // profile contains user info from Google
    let user = await this.usersService.findByGoogleId(profile.id);

    if (!user) {
      // First login - create user
      user = await this.usersService.create({
        googleId: profile.id,
        email: profile.emails[0].value,
        name: profile.displayName,
      });
    }

    return user;
  }
}

// Controller
@Controller('auth')
export class AuthController {
  @Get('google')
  @UseGuards(AuthGuard('google'))
  async googleAuth() {
    // Redirects to Google
  }

  @Get('google/callback')
  @UseGuards(AuthGuard('google'))
  async googleAuthRedirect(@Request() req) {
    // req.user is set by strategy
    const token = this.authService.generateToken(req.user);

    // Redirect to frontend with token
    return res.redirect(`http://localhost:3001/auth-success?token=${token}`);
  }
}
```

### When to Use

- Reduce friction (users don't create new passwords)
- Leverage existing identity providers
- Enterprise authentication (OAuth with internal identity server)
- Social features (access user's social media data)

---

## Password Security

### The Only Thing You Need to Know: Use bcrypt

```typescript
import * as bcrypt from 'bcrypt';

// When user signs up - hash the password
const hashedPassword = await bcrypt.hash(plainPassword, 10);
// Store hashedPassword in database

// When user logs in - compare passwords
const isPasswordValid = await bcrypt.compare(plainPassword, hashedPassword);
if (isPasswordValid) {
  // Allow login
}
```

### Why Not MD5/SHA-1?

- They're fast (this is bad for passwords - easy to brute force)
- They don't have a "salt" (dictionary attacks work)
- They're broken (collisions found)

### Why bcrypt?

- ✅ Slow by design (take ~0.1s per hash, making brute force impractical)
- ✅ Automatically handles salt
- ✅ Adaptive cost factor (can increase as computers get faster)
- ✅ Industry standard

### Password Requirements (Controversial)

Most security experts now recommend:
- **Minimum 8-12 characters** (length > complexity)
- **No arbitrary special character requirements** (users write worse passwords)
- **Check against common password lists** (is password in top 100,000 used passwords?)
- **Allow password managers** (don't block copy-paste)

```typescript
// Good password validation
const isGoodPassword = (password: string) => {
  const commonPasswords = ['password', '123456', 'qwerty', ...];

  return password.length >= 8 &&
         !commonPasswords.includes(password.toLowerCase());
};
```

---

## Authorization (Access Control)

**Authentication** = "Who are you?" (Identity)
**Authorization** = "What are you allowed to do?" (Permissions)

### Role-Based Access Control (RBAC) - 80% of Cases

```typescript
// Define roles
enum Role {
  ADMIN = 'admin',
  USER = 'user',
  MODERATOR = 'moderator',
}

// Attach role to user
interface User {
  id: string;
  email: string;
  role: Role; // Single role per user
}

// Protect endpoints
@UseGuards(JwtAuthGuard, RoleGuard)
@Roles(Role.ADMIN) // Only admins
@Get('/admin/users')
getAllUsers() {
  // Only executed if user has admin role
}

// Role Guard implementation
@Injectable()
export class RoleGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const requiredRoles = Reflect.getMetadata('roles', context.getHandler());

    if (!requiredRoles) return true; // No role requirement

    return requiredRoles.includes(request.user.role);
  }
}
```

### Permission-Based Access Control (PBAC) - 20% of Cases

Use when you need fine-grained permissions like "user can edit post 123":

```typescript
// Users have permissions (many-to-many)
interface Permission {
  id: string;
  action: string;  // 'read', 'edit', 'delete'
  resource: string; // 'post', 'user', 'settings'
  resourceId?: string; // specific post/user id (optional)
}

// Check permission
@UseGuards(JwtAuthGuard, PermissionGuard)
@RequirePermission('edit', 'post', ':postId')
@Patch('/posts/:postId')
updatePost(@Param('postId') postId: string) {
  // Only executed if user has 'edit' permission for this specific post
}
```

### Key Rule

**Authorize on every operation, not just endpoints:**

```typescript
// WRONG - only checks at endpoint level
@Patch('/posts/:postId')
updatePost(@Param('postId') postId: string) {
  // Assuming user can edit - but what if they don't own the post?
  return this.postsService.update(postId, newData);
}

// CORRECT - check in business logic
@Patch('/posts/:postId')
updatePost(@Param('postId') postId: string, @Request() req) {
  const post = await this.postsService.findById(postId);

  // Verify authorization in the service
  if (post.ownerId !== req.user.id && req.user.role !== Role.ADMIN) {
    throw new ForbiddenException('Cannot edit this post');
  }

  return this.postsService.update(postId, newData);
}
```

---

## Refresh Token Pattern

### The Problem

JWT tokens have expiration times (usually 15 minutes). If you set long expiration:
- Security risk: stolen token valid for days
- Can't revoke immediately (no server-side logout)

### The Solution

Use two tokens:

| Token | Duration | Purpose | Storage |
|-------|----------|---------|---------|
| **Access Token** | 15 min | Call APIs | Memory (cleared on refresh) |
| **Refresh Token** | 7 days | Get new access token | HTTP-only cookie (persistent) |

### How It Works

```
1. User logs in → get both tokens
2. Client stores access token in memory
3. Client stores refresh token in HTTP-only cookie
4. Make API calls with access token
5. Access token expires → client gets 401
6. Client uses refresh token to get new access token
7. Continue making API calls
8. User logs out → server deletes refresh token + clear cookie
```

### Implementation (NestJS)

```typescript
// Generate both tokens
@Post('/login')
async login(@Body() body: LoginDto) {
  const user = await this.validateUser(body.username, body.password);

  const accessToken = this.jwtService.sign(
    { userId: user.id },
    { expiresIn: '15m' }
  );

  const refreshToken = this.jwtService.sign(
    { userId: user.id, type: 'refresh' },
    { expiresIn: '7d' }
  );

  // Store refresh token in database (for revocation)
  await this.authService.saveRefreshToken(user.id, refreshToken);

  return {
    accessToken,
    refreshToken, // Send in HTTP-only cookie
    expiresIn: 15 * 60, // 15 minutes in seconds
  };
}

// Refresh endpoint
@Post('/refresh')
async refresh(@Request() req) {
  const refreshToken = req.cookies.refreshToken;

  if (!refreshToken) {
    throw new UnauthorizedException();
  }

  try {
    const decoded = this.jwtService.verify(refreshToken);

    // Check if token is blacklisted (was user logged out?)
    const isValid = await this.authService.isRefreshTokenValid(refreshToken);
    if (!isValid) throw new UnauthorizedException();

    const newAccessToken = this.jwtService.sign(
      { userId: decoded.userId },
      { expiresIn: '15m' }
    );

    return { accessToken: newAccessToken };
  } catch {
    throw new UnauthorizedException();
  }
}

// Logout
@Post('/logout')
async logout(@Request() req) {
  const refreshToken = req.cookies.refreshToken;

  // Blacklist the refresh token
  await this.authService.blacklistRefreshToken(refreshToken);

  // Clear cookie
  res.clearCookie('refreshToken');
  return { success: true };
}
```

### When to Use

- Any JWT-based system with long-lived tokens
- Need to support revocation (immediate logout)
- Balancing security (short access token) with UX (don't require frequent re-login)

---

## Common Implementation Patterns

### Pattern 1: Simple Password Login (Most Common)

```typescript
@Post('/login')
async login(@Body() body: LoginDto) {
  const user = await this.usersService.findByEmail(body.email);

  if (!user || !(await bcrypt.compare(body.password, user.passwordHash))) {
    throw new UnauthorizedException('Invalid email or password');
  }

  const token = this.jwtService.sign({ userId: user.id });

  return { accessToken: token };
}
```

### Pattern 2: Social Login (Google/GitHub)

Use Passport strategies (as shown above). Passport handles OAuth flow complexity.

### Pattern 3: Two-Factor Authentication (2FA)

```typescript
// After password validation
@Post('/login')
async login(@Body() body: LoginDto) {
  const user = await this.validatePassword(body.email, body.password);

  if (!user) throw new UnauthorizedException();

  if (user.has2FAEnabled) {
    // Don't give full token yet
    const tempToken = this.jwtService.sign(
      { userId: user.id, partial: true },
      { expiresIn: '5m' } // Very short - only for 2FA step
    );

    await this.sendTwoFactorCode(user.email); // SMS or email

    return { requiresTwoFactor: true, tempToken };
  }

  // Normal login if no 2FA
  return { accessToken: this.jwtService.sign({ userId: user.id }) };
}

// Verify 2FA code
@Post('/verify-2fa')
async verify2FA(@Body() body: { code: string }, @Request() req) {
  const tempToken = req.headers.authorization?.replace('Bearer ', '');
  const decoded = this.jwtService.verify(tempToken);

  if (!decoded.partial) throw new UnauthorizedException();

  const isValid = await this.authService.verify2FACode(decoded.userId, body.code);
  if (!isValid) throw new UnauthorizedException();

  // Now give full token
  return {
    accessToken: this.jwtService.sign({ userId: decoded.userId })
  };
}
```

### Pattern 4: API Key Authentication (Service-to-Service)

```typescript
// For backend services calling your API
@UseGuards(ApiKeyGuard)
@Get('/data')
getData() {
  return { data: [...] };
}

@Injectable()
export class ApiKeyGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const apiKey = request.headers['x-api-key'];

    if (!apiKey) return false;

    // Check if API key is valid and not revoked
    return this.authService.validateApiKey(apiKey);
  }
}
```

---

## 80/20 Checklist

When implementing authentication, ensure you have:

- ✅ **Password Hashing**: Using bcrypt (never store plain passwords)
- ✅ **HTTPS Enforcement**: All auth endpoints use HTTPS
- ✅ **Token Expiration**: Access tokens expire within 15-60 minutes
- ✅ **Secure Cookies**: Set `httpOnly`, `secure`, `sameSite` flags
- ✅ **CSRF Protection**: If using session-based auth
- ✅ **Authorization Checks**: Every endpoint validates user permissions
- ✅ **Error Messages**: Don't reveal if email exists ("Invalid email or password" not "Email not found")
- ✅ **Rate Limiting**: Limit login attempts to prevent brute force
- ✅ **Logout**: Clear tokens/sessions on logout
- ✅ **Remember Me**: Optional, use refresh tokens instead

---

## Key Takeaways

1. **Choose one pattern**: JWT for modern apps, sessions for server-rendered, OAuth for social login
2. **Hash passwords with bcrypt**: Non-negotiable
3. **Use short access token lifetimes**: 15-60 minutes
4. **Implement refresh tokens**: For revocation and UX balance
5. **Authorize at every boundary**: Don't just check at endpoints
6. **Use HTTPS everywhere**: Non-negotiable
7. **Keep auth simple**: Don't over-engineer with custom solutions
8. **Test edge cases**: Expired tokens, revoked access, concurrent login

---

## Quick Reference

| Scenario | Approach |
|----------|----------|
| Traditional web app | Session-based auth |
| Single Page App | JWT + Refresh Token |
| Mobile app | JWT + Refresh Token |
| Microservices | JWT |
| Social login | OAuth 2.0 + JWT |
| Service-to-service | API Keys or Mutual TLS |
| Enterprise SSO | OAuth 2.0 or SAML |

---

## Further Reading

- [OAuth 2.0 Simplified](https://aaronparecki.com/oauth-2-simplified/) - Great visual explanation
- [JWT Best Practices](https://tools.ietf.org/html/rfc8949) - Official RFC
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) - Security best practices
- [Passport.js Documentation](http://www.passportjs.org/) - Strategy-based auth for Node.js
