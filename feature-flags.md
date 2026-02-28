# Feature Flags

A practical guide to implementing feature flags for controlled feature rollout, A/B testing, and gradual deployment.

## 1. What Are Feature Flags?

Feature flags (also called feature toggles or feature switches) are conditional logic that allows you to enable/disable features without deploying code.

**Key Characteristics:**
- Control features at runtime
- No deployment needed to toggle
- Gradual rollout to users
- Easy rollback on issues
- Enable A/B testing
- Perfect for beta features

**Common Use Cases:**
- Gradual feature rollout (5% → 25% → 50% → 100% of users)
- Beta testing with specific users
- A/B testing variations
- Kill switch for failing features
- Testing in production with limited users
- Canary deployments
- Legacy code deprecation

**Pareto Principle:**
- 20% of your codebase gets 80% of toggles (high-risk, new, experimental)
- Don't over-engineer: simple flags for simple needs, complex solutions when needed

---

## 2. Types of Feature Flags

### 1. Boolean Flags (Simplest)

On/off switch for a feature:

```typescript
// Backend
if (featureFlags.newCheckout) {
  // New code
} else {
  // Old code
}
```

**Use for:** Simple on/off features, quick rollouts

### 2. Percentage-Based Flags (Gradual Rollout)

Enable for percentage of users:

```typescript
// 10% of users see the new feature
const shouldShowNewUI = isUserInPercentage('newUI', 10);
```

**Use for:** Gradual rollouts, testing with subset of users

### 3. User-Targeted Flags (Specific Users)

Enable for specific users or user groups:

```typescript
// Enable for specific users
const canAccessBeta = isUserInTargetList('betaFeature', userId);

// Enable for users with specific properties
const canAccessPremium = user.tier === 'premium';
```

**Use for:** Beta testing, premium features, internal testing

### 4. Cohort-Based Flags (User Segments)

Enable based on user properties/segments:

```typescript
// Enable for users in a specific region
const canAccessFeature = user.region === 'US' && user.isPremium;
```

**Use for:** Localized features, premium features, role-based access

### 5. Context-Based Flags (Conditional)

Enable based on runtime context:

```typescript
// Enable based on time, environment, feature state, etc.
const canAccessFeature =
  isProduction() &&
  !isHighLoad() &&
  featureIsHealthy('newPayment');
```

**Use for:** Performance-sensitive rollouts, emergency shutdown

---

## 3. Feature Flag Strategies

### Strategy 1: Simple Boolean

Simplest implementation, good for yes/no decisions:

```typescript
// Easy to understand
// Easy to remove later
// Limited flexibility
const showNewDashboard = featureFlags.newDashboard === true;
```

**When to use:** Quick experiments, simple toggles

### Strategy 2: User Percentage

Gradual rollout to increasing percentages:

```typescript
// Consistent: same user always sees same variation
// Gradual: can increase percentage over time
// Fair distribution
const rolloutPercentage = 25; // 25% of users
const userHash = hashUserId(userId);
const userPercentage = userHash % 100;
const isEnabled = userPercentage < rolloutPercentage;
```

**When to use:** Safe rollouts, feature releases, A/B tests

### Strategy 3: User ID List

Whitelist/blacklist specific users:

```typescript
// Exact control
// Best for beta testing
// Manual management required
const betaUsers = ['user-123', 'user-456', 'user-789'];
const isEnabled = betaUsers.includes(userId);
```

**When to use:** Beta testing, internal team features, VIP access

### Strategy 4: Server-Side Evaluation

Evaluate on backend, return to frontend:

```typescript
// More secure (rules hidden from users)
// Consistent across all clients
// Requires API call
const flags = await fetchFlagsForUser(userId);
```

**When to use:** Security-sensitive features, complex logic, consistent state

---

## 4. Backend Implementation (NestJS)

### Step 1: Feature Flag Service

```typescript
// src/features/feature-flags/feature-flag.service.ts
import { Injectable } from '@nestjs/common';

interface FeatureFlag {
  name: string;
  enabled: boolean;
  rolloutPercentage?: number;
  targetUsers?: string[];
  targetRoles?: string[];
  metadata?: Record<string, any>;
}

@Injectable()
export class FeatureFlagService {
  private flags: Map<string, FeatureFlag> = new Map([
    [
      'new-checkout',
      {
        name: 'new-checkout',
        enabled: true,
        rolloutPercentage: 50,
      },
    ],
    [
      'advanced-analytics',
      {
        name: 'advanced-analytics',
        enabled: true,
        targetRoles: ['premium', 'admin'],
      },
    ],
    [
      'dark-mode',
      {
        name: 'dark-mode',
        enabled: true,
        rolloutPercentage: 100,
      },
    ],
    [
      'ai-suggestions',
      {
        name: 'ai-suggestions',
        enabled: false, // Beta, not yet enabled
        targetUsers: ['user-123', 'user-456'],
      },
    ],
  ]);

  /**
   * Check if a feature is enabled for a user
   */
  isEnabled(
    featureName: string,
    context?: {
      userId?: string;
      userRole?: string;
      percentage?: number; // Override rollout percentage
    },
  ): boolean {
    const flag = this.flags.get(featureName);

    if (!flag) {
      // Feature doesn't exist, default to disabled
      return false;
    }

    // If feature is completely disabled
    if (!flag.enabled) {
      return false;
    }

    // If no context provided, check only global enabled status
    if (!context) {
      return true;
    }

    // Check user-specific targeting
    if (flag.targetUsers && context.userId) {
      return flag.targetUsers.includes(context.userId);
    }

    // Check role-based targeting
    if (flag.targetRoles && context.userRole) {
      return flag.targetRoles.includes(context.userRole);
    }

    // Check percentage-based rollout
    if (flag.rolloutPercentage !== undefined && context.userId) {
      return this.isUserInPercentage(
        context.userId,
        featureName,
        context.percentage || flag.rolloutPercentage,
      );
    }

    return true;
  }

  /**
   * Consistent hash to determine if user is in rollout percentage
   */
  private isUserInPercentage(
    userId: string,
    featureName: string,
    percentage: number,
  ): boolean {
    // Create consistent hash for user + feature
    const hash = this.hashUserFeature(userId, featureName);
    const userPercentage = hash % 100;
    return userPercentage < percentage;
  }

  /**
   * Simple hash function for consistent distribution
   */
  private hashUserFeature(userId: string, featureName: string): number {
    const combined = `${userId}:${featureName}`;
    let hash = 0;

    for (let i = 0; i < combined.length; i++) {
      const char = combined.charCodeAt(i);
      hash = (hash << 5) - hash + char;
      hash = hash & hash; // Convert to 32-bit integer
    }

    return Math.abs(hash);
  }

  /**
   * Get all enabled features for a user
   */
  getAllEnabledFeatures(context?: {
    userId?: string;
    userRole?: string;
  }): string[] {
    const enabled: string[] = [];

    for (const [name] of this.flags) {
      if (this.isEnabled(name, context)) {
        enabled.push(name);
      }
    }

    return enabled;
  }

  /**
   * Get flag details (for admin/debugging)
   */
  getFlag(featureName: string): FeatureFlag | undefined {
    return this.flags.get(featureName);
  }

  /**
   * Get all flags (for admin panel)
   */
  getAllFlags(): FeatureFlag[] {
    return Array.from(this.flags.values());
  }

  /**
   * Update flag (for admin panel)
   */
  updateFlag(featureName: string, updates: Partial<FeatureFlag>): void {
    const flag = this.flags.get(featureName);

    if (!flag) {
      throw new Error(`Feature flag '${featureName}' not found`);
    }

    this.flags.set(featureName, { ...flag, ...updates });
  }

  /**
   * Add new flag
   */
  createFlag(flag: FeatureFlag): void {
    this.flags.set(flag.name, flag);
  }
}
```

### Step 2: Feature Flag Controller

```typescript
// src/features/feature-flags/feature-flags.controller.ts
import { Controller, Get, Req, Post, Body, Param } from '@nestjs/common';
import { FeatureFlagService } from './feature-flag.service';

@Controller('api/feature-flags')
export class FeatureFlagsController {
  constructor(private featureFlagService: FeatureFlagService) {}

  /**
   * Get all enabled features for current user
   * GET /api/feature-flags
   */
  @Get()
  getEnabledFeatures(@Req() request: any) {
    const userId = request.user?.id;
    const userRole = request.user?.role;

    const enabledFeatures = this.featureFlagService.getAllEnabledFeatures({
      userId,
      userRole,
    });

    return {
      features: enabledFeatures,
      timestamp: new Date().toISOString(),
    };
  }

  /**
   * Check if specific feature is enabled
   * GET /api/feature-flags/:featureName
   */
  @Get(':featureName')
  isFeatureEnabled(@Param('featureName') featureName: string, @Req() request: any) {
    const userId = request.user?.id;
    const userRole = request.user?.role;

    const isEnabled = this.featureFlagService.isEnabled(featureName, {
      userId,
      userRole,
    });

    return {
      feature: featureName,
      enabled: isEnabled,
      userId,
    };
  }

  /**
   * Get feature details (admin only)
   * GET /api/feature-flags/admin/:featureName
   */
  @Get('admin/:featureName')
  getFeatureDetails(@Param('featureName') featureName: string) {
    const flag = this.featureFlagService.getFlag(featureName);

    if (!flag) {
      return { error: 'Feature flag not found' };
    }

    return flag;
  }

  /**
   * Get all flags (admin only)
   * GET /api/feature-flags/admin/all
   */
  @Get('admin/all')
  getAllFlags() {
    return this.featureFlagService.getAllFlags();
  }

  /**
   * Update flag (admin only)
   * POST /api/feature-flags/admin/:featureName
   */
  @Post('admin/:featureName')
  updateFlag(
    @Param('featureName') featureName: string,
    @Body() updates: Record<string, any>,
  ) {
    try {
      this.featureFlagService.updateFlag(featureName, updates);
      return { success: true, feature: featureName };
    } catch (error) {
      return { error: error.message };
    }
  }
}
```

### Step 3: Guard/Middleware for Protected Routes

```typescript
// src/features/feature-flags/feature-flag.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  ForbiddenException,
} from '@nestjs/common';
import { FeatureFlagService } from './feature-flag.service';

@Injectable()
export class FeatureFlagGuard implements CanActivate {
  constructor(private featureFlagService: FeatureFlagService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();

    // Get feature name from metadata or query param
    const featureName =
      Reflect.getMetadata('featureName', context.getHandler()) ||
      request.query.featureName;

    if (!featureName) {
      return true; // No feature flag requirement
    }

    const isEnabled = this.featureFlagService.isEnabled(featureName, {
      userId: request.user?.id,
      userRole: request.user?.role,
    });

    if (!isEnabled) {
      throw new ForbiddenException(
        `Feature '${featureName}' is not available for this user`,
      );
    }

    return true;
  }
}

// Usage in controller:
// @Get('new-feature')
// @UseGuards(FeatureFlagGuard)
// @SetMetadata('featureName', 'new-checkout')
// newFeature() { ... }
```

### Step 4: Module Setup

```typescript
// src/features/feature-flags/feature-flags.module.ts
import { Module } from '@nestjs/common';
import { FeatureFlagService } from './feature-flag.service';
import { FeatureFlagsController } from './feature-flags.controller';

@Module({
  providers: [FeatureFlagService],
  controllers: [FeatureFlagsController],
  exports: [FeatureFlagService], // Export for other modules
})
export class FeatureFlagsModule {}
```

---

## 5. Frontend Implementation (React)

### Step 1: Feature Flag Hook

```typescript
// src/hooks/useFeatureFlag.ts
import { useEffect, useState } from 'react';
import { useAuth } from './useAuth'; // Your auth context

interface FeatureFlagsContextType {
  flags: Record<string, boolean>;
  loading: boolean;
  error: string | null;
  isEnabled: (featureName: string) => boolean;
  refresh: () => Promise<void>;
}

export const useFeatureFlag = (): FeatureFlagsContextType => {
  const { user, isAuthenticated } = useAuth();
  const [flags, setFlags] = useState<Record<string, boolean>>({});
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  // Fetch feature flags from backend
  const fetchFlags = async () => {
    if (!isAuthenticated) {
      setFlags({});
      setLoading(false);
      return;
    }

    try {
      setLoading(true);
      const response = await fetch('/api/feature-flags', {
        headers: {
          Authorization: `Bearer ${user?.token}`,
        },
      });

      if (!response.ok) {
        throw new Error('Failed to fetch feature flags');
      }

      const data = await response.json();

      // Convert array to object for easy lookup
      const flagsMap: Record<string, boolean> = {};
      data.features?.forEach((feature: string) => {
        flagsMap[feature] = true;
      });

      setFlags(flagsMap);
      setError(null);
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Unknown error');
      console.error('Feature flag fetch error:', err);
    } finally {
      setLoading(false);
    }
  };

  // Fetch on mount and when user changes
  useEffect(() => {
    fetchFlags();
  }, [isAuthenticated, user?.id]);

  // Check if feature is enabled (default to false if not found)
  const isEnabled = (featureName: string): boolean => {
    return flags[featureName] === true;
  };

  return {
    flags,
    loading,
    error,
    isEnabled,
    refresh: fetchFlags,
  };
};
```

### Step 2: Feature Flag Context Provider

```typescript
// src/context/FeatureFlagContext.tsx
import React, { createContext, useContext, ReactNode } from 'react';
import { useFeatureFlag } from '../hooks/useFeatureFlag';

interface FeatureFlagContextType {
  flags: Record<string, boolean>;
  loading: boolean;
  error: string | null;
  isEnabled: (featureName: string) => boolean;
  refresh: () => Promise<void>;
}

const FeatureFlagContext = createContext<FeatureFlagContextType | undefined>(
  undefined,
);

export const FeatureFlagProvider: React.FC<{ children: ReactNode }> = ({
  children,
}) => {
  const featureFlags = useFeatureFlag();

  return (
    <FeatureFlagContext.Provider value={featureFlags}>
      {children}
    </FeatureFlagContext.Provider>
  );
};

export const useFeatureFlags = (): FeatureFlagContextType => {
  const context = useContext(FeatureFlagContext);

  if (context === undefined) {
    throw new Error(
      'useFeatureFlags must be used within FeatureFlagProvider',
    );
  }

  return context;
};
```

### Step 3: Conditional Components

```typescript
// src/components/FeatureGate.tsx
import React, { ReactNode } from 'react';
import { useFeatureFlags } from '../context/FeatureFlagContext';

interface FeatureGateProps {
  feature: string;
  children: ReactNode;
  fallback?: ReactNode;
}

/**
 * Render children only if feature is enabled
 * Usage:
 * <FeatureGate feature="newCheckout">
 *   <NewCheckoutFlow />
 * </FeatureGate>
 */
export const FeatureGate: React.FC<FeatureGateProps> = ({
  feature,
  children,
  fallback = null,
}) => {
  const { isEnabled, loading } = useFeatureFlags();

  if (loading) {
    return <>{fallback}</>;
  }

  return isEnabled(feature) ? <>{children}</> : <>{fallback}</>;
};

// Usage with fallback:
// <FeatureGate
//   feature="newCheckout"
//   fallback={<OldCheckoutFlow />}
// >
//   <NewCheckoutFlow />
// </FeatureGate>
```

### Step 4: Practical Examples

```typescript
// Example 1: Simple conditional rendering
import { useFeatureFlags } from '../context/FeatureFlagContext';

function Dashboard() {
  const { isEnabled } = useFeatureFlags();

  return (
    <div>
      <h1>Dashboard</h1>

      {isEnabled('newDashboard') && <NewDashboardLayout />}
      {!isEnabled('newDashboard') && <LegacyDashboardLayout />}

      {isEnabled('advancedAnalytics') && <AnalyticsPanel />}
    </div>
  );
}

// Example 2: Using FeatureGate component
function CheckoutFlow() {
  return (
    <div>
      <FeatureGate
        feature="new-checkout"
        fallback={<OldCheckoutFlow />}
      >
        <NewCheckoutFlow />
      </FeatureGate>
    </div>
  );
}

// Example 3: Feature-based navigation
function Navigation() {
  const { isEnabled } = useFeatureFlags();

  return (
    <nav>
      <Link to="/dashboard">Dashboard</Link>

      {isEnabled('advancedAnalytics') && (
        <Link to="/analytics">Advanced Analytics</Link>
      )}

      {isEnabled('aiSuggestions') && (
        <Link to="/ai-assistant">AI Assistant (Beta)</Link>
      )}
    </nav>
  );
}

// Example 4: Conditional feature button with experimental badge
function FeatureButton() {
  const { isEnabled } = useFeatureFlags();

  return (
    <button>
      New Feature
      {isEnabled('betaFeature') && (
        <span className="badge-beta">BETA</span>
      )}
    </button>
  );
}

// Example 5: Progressive enhancement
function UserProfile() {
  const { isEnabled } = useFeatureFlags();

  return (
    <div className="profile">
      {/* Always available */}
      <h1>User Profile</h1>

      {/* Available to some users */}
      {isEnabled('darkMode') && (
        <button>🌙 Toggle Dark Mode</button>
      )}

      {/* Premium feature */}
      {isEnabled('advancedPrivacy') && (
        <PrivacySettings />
      )}

      {/* Very new feature */}
      {isEnabled('aiPersonalization') && (
        <PersonalizationPanel />
      )}
    </div>
  );
}
```

### Step 5: Setup in App.tsx

```typescript
// src/App.tsx
import { BrowserRouter } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';
import { FeatureFlagProvider } from './context/FeatureFlagContext';
import Routes from './routes';

function App() {
  return (
    <BrowserRouter>
      <AuthProvider>
        <FeatureFlagProvider>
          <Routes />
        </FeatureFlagProvider>
      </AuthProvider>
    </BrowserRouter>
  );
}

export default App;
```

---

## 6. Advanced Patterns

### Pattern 1: A/B Testing

```typescript
// Backend
const flag = this.featureFlagService.getFlag('checkoutUI');
// rolloutPercentage controls percentage split

// Frontend
function CheckoutPage() {
  const { isEnabled } = useFeatureFlags();
  const variant = isEnabled('checkoutUI') ? 'new' : 'old';

  return (
    <Analytics event="checkout_loaded" variant={variant}>
      {variant === 'new' ? <NewCheckout /> : <OldCheckout />}
    </Analytics>
  );
}
```

### Pattern 2: Progressive Rollout

```typescript
// Week 1: Enable for 5% of users
featureFlagService.updateFlag('newPayment', {
  rolloutPercentage: 5,
});

// Week 2: Expand to 25%
featureFlagService.updateFlag('newPayment', {
  rolloutPercentage: 25,
});

// Week 3: Full rollout
featureFlagService.updateFlag('newPayment', {
  rolloutPercentage: 100,
});

// Week 4: Remove flag (cleanup)
// Delete the rollout logic from code
```

### Pattern 3: Kill Switch

```typescript
// Backend
@Get('critical-operation')
@UseGuards(FeatureFlagGuard)
@SetMetadata('featureName', 'critical-operation-enabled')
criticalOperation() {
  // Operation logic
}

// If operation starts failing:
// POST /api/feature-flags/admin/critical-operation-enabled
// { enabled: false }
// Instantly disables for all users
```

### Pattern 4: Role-Based Access

```typescript
// Backend
const flag: FeatureFlag = {
  name: 'advancedReporting',
  enabled: true,
  targetRoles: ['admin', 'analyst'],
};

// Frontend
{isEnabled('advancedReporting') && <AdvancedReporting />}
```

### Pattern 5: Time-Based Rollout

```typescript
// Backend: Custom evaluation logic
isEnabled(featureName: string, context: any): boolean {
  const flag = this.flags.get(featureName);

  if (flag?.metadata?.scheduledRollout) {
    const now = Date.now();
    const { startTime, endTime } = flag.metadata.scheduledRollout;

    if (now < startTime || now > endTime) {
      return false;
    }
  }

  return true;
}
```

---

## 7. Feature Flag Lifecycle

### Phase 1: Development
```
- Create feature flag (disabled by default)
- Wrap new code in flag
- Test locally with flag enabled
```

### Phase 2: Beta Testing
```
- Enable for specific beta users
- Monitor metrics and errors
- Gather feedback
```

### Phase 3: Gradual Rollout
```
- Week 1: 5% of users
- Week 2: 25% of users
- Week 3: 50% of users
- Week 4: 100% of users
```

### Phase 4: Monitoring
```
- Watch error rates
- Monitor performance metrics
- Be ready to kill switch if needed
```

### Phase 5: Cleanup
```
- Once stable, remove flag from code
- Clean up conditional logic
- Remove flag from system
```

---

## 8. Best Practices

### Do ✅
- **Start with flags disabled** - Safer default
- **Use consistent hashing** - Same user always sees same variation
- **Monitor metrics** - Track errors, performance, adoption
- **Have a kill switch** - Ability to disable instantly
- **Document flags** - Add metadata explaining purpose
- **Clean up old flags** - Remove when no longer needed
- **Test all variations** - Test both flag enabled and disabled
- **Gradual rollouts** - Don't jump to 100% immediately

### Don't ❌
- **Over-engineer** - Start simple, add complexity if needed
- **Leave flags indefinitely** - Creates technical debt
- **Toggle in production manually** - Causes inconsistency
- **Hide entire features** - Use flags for implementation, not UI
- **Forget to test disabled path** - The old code is important too
- **Ignore monitoring** - You need visibility into rollouts
- **Use for permissions** - Flags control features, not access

---

## 9. Testing with Feature Flags

### Unit Tests

```typescript
describe('FeatureFlagService', () => {
  let service: FeatureFlagService;

  beforeEach(() => {
    service = new FeatureFlagService();
  });

  it('should return true for enabled feature', () => {
    expect(service.isEnabled('newCheckout')).toBe(true);
  });

  it('should return false for disabled feature', () => {
    expect(service.isEnabled('aiSuggestions')).toBe(false);
  });

  it('should respect user percentage rollout', () => {
    // Same user should always get same result
    const user1Enabled = service.isEnabled('newUI', {
      userId: 'user-1',
    });
    const user1EnabledAgain = service.isEnabled('newUI', {
      userId: 'user-1',
    });

    expect(user1Enabled).toBe(user1EnabledAgain);
  });

  it('should respect user targeting', () => {
    const isEnabled = service.isEnabled('betaFeature', {
      userId: 'beta-user-1',
    });

    expect(isEnabled).toBe(true);
  });
});
```

### Integration Tests

```typescript
describe('Feature Flags API', () => {
  it('should return enabled features for user', async () => {
    const response = await request(app.getHttpServer())
      .get('/api/feature-flags')
      .set('Authorization', 'Bearer token')
      .expect(200);

    expect(response.body).toHaveProperty('features');
    expect(Array.isArray(response.body.features)).toBe(true);
  });

  it('should check specific feature', async () => {
    const response = await request(app.getHttpServer())
      .get('/api/feature-flags/new-checkout')
      .set('Authorization', 'Bearer token')
      .expect(200);

    expect(response.body).toHaveProperty('enabled');
  });
});
```

### React Component Tests

```typescript
import { render, screen } from '@testing-library/react';
import { FeatureFlagProvider } from '../context/FeatureFlagContext';
import { FeatureGate } from '../components/FeatureGate';

jest.mock('../hooks/useFeatureFlag', () => ({
  useFeatureFlag: () => ({
    flags: { newCheckout: true },
    loading: false,
    error: null,
    isEnabled: (name: string) => name === 'newCheckout',
    refresh: jest.fn(),
  }),
}));

describe('FeatureGate', () => {
  it('should render children when feature enabled', () => {
    render(
      <FeatureFlagProvider>
        <FeatureGate feature="newCheckout">
          <div>New Checkout</div>
        </FeatureGate>
      </FeatureFlagProvider>,
    );

    expect(screen.getByText('New Checkout')).toBeInTheDocument();
  });

  it('should render fallback when feature disabled', () => {
    render(
      <FeatureFlagProvider>
        <FeatureGate
          feature="disabledFeature"
          fallback={<div>Old Checkout</div>}
        >
          <div>New Checkout</div>
        </FeatureGate>
      </FeatureFlagProvider>,
    );

    expect(screen.getByText('Old Checkout')).toBeInTheDocument();
  });
});
```

---

## 10. Monitoring & Observability

### Backend Logging

```typescript
// Log when features are accessed
@Get(':featureName')
isFeatureEnabled(@Param('featureName') featureName: string, @Req() request: any) {
  const userId = request.user?.id;
  const isEnabled = this.featureFlagService.isEnabled(featureName, { userId });

  // Log for monitoring
  this.logger.log({
    event: 'feature_flag_check',
    feature: featureName,
    enabled: isEnabled,
    userId,
    timestamp: new Date().toISOString(),
  });

  return { feature: featureName, enabled: isEnabled };
}
```

### Frontend Analytics

```typescript
function Dashboard() {
  const { isEnabled } = useFeatureFlags();

  useEffect(() => {
    // Track which features are enabled for this user
    const enabledFeatures = ['newUI', 'darkMode', 'aiSuggestions'].filter(
      (f) => isEnabled(f),
    );

    analytics.track('user_features_loaded', {
      enabledFeatures,
      timestamp: new Date().toISOString(),
    });
  }, [isEnabled]);

  return <div>Dashboard</div>;
}
```

### Metrics to Track

```
- Feature adoption rate (% of users seeing feature)
- Error rates by feature
- Performance metrics by feature variant
- User engagement by variant (for A/B tests)
- Rollout progress (% of target users reached)
```

---

## 11. Common Pitfalls

### Pitfall 1: Flag Proliferation
**Problem:** Too many flags, impossible to manage

**Solution:**
- Archive old flags
- Use naming conventions
- Set expiration dates
- Regular cleanup

### Pitfall 2: Inconsistent State
**Problem:** Flag evaluated differently across requests

**Solution:**
- Use consistent hashing
- Cache flag values per session
- Evaluate server-side, return to client

### Pitfall 3: Performance Impact
**Problem:** Slow flag evaluations

**Solution:**
- Cache flag values
- Use in-memory flag storage
- Batch flag requests
- Optimize hash functions

### Pitfall 4: Dependency Hell
**Problem:** Flags depend on each other, hard to manage

**Solution:**
- Keep flags independent
- Use explicit dependencies in metadata
- Document flag relationships

---

## 12. Quick Reference

### Backend Setup
```typescript
// 1. Create service
// 2. Inject into controller
// 3. Use in routes/middleware
// 4. Call isEnabled(name, context)

const isEnabled = featureFlagService.isEnabled('newCheckout', {
  userId: user.id,
  userRole: user.role,
});
```

### Frontend Setup
```typescript
// 1. Wrap app in FeatureFlagProvider
// 2. Use useFeatureFlags hook
// 3. Use FeatureGate component or isEnabled()

const { isEnabled } = useFeatureFlags();
{isEnabled('newCheckout') && <NewCheckout />}
```

### Progressive Rollout
```
Week 1: 5%
Week 2: 25%
Week 3: 50%
Week 4: 100%
Then: Remove flag from code
```

---

## Key Takeaways

1. **Feature flags enable safe releases** - Deploy code, enable gradually
2. **Multiple strategies available** - Boolean, percentage, user-targeted, role-based
3. **Backend evaluation is safer** - Server controls what users see
4. **Frontend caches for performance** - Fetch once, use in multiple places
5. **Consistent hashing matters** - Same user always sees same variation
6. **Monitor carefully** - Watch metrics during rollout
7. **Kill switch essential** - Ability to disable instantly
8. **Clean up is important** - Remove old flags to avoid technical debt
9. **Test both paths** - Test with flag enabled and disabled
10. **Start simple** - Boolean flags are often sufficient

