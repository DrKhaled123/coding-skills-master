# Code Quality Standards & Mandatory Structure

## 🎯 Purpose

Standards for writing clean, maintainable, production-ready code the first time. These are not suggestions — they are requirements.

> **VIP EXPERT NOTE — The Cost of Technical Debt:**
> Every shortcut now costs 10x to fix later. A function with unclear naming,
> missing types, or swallowed errors will generate 3-5 bug reports over its
> lifetime. Writing it right the first time is not "slower" — it's faster
> when measured across the project lifecycle.

---

## 🔷 FUNCTION DESIGN

### The Single Responsibility Principle (Hard Rule)

**One function does exactly one thing. If you need "and" to describe it, split it.**

```typescript
// ❌ WRONG: Does three things (fetch, transform, render)
async function loadAndDisplayUserProfile(userId: string) {
  const response = await fetch(`/api/users/${userId}`);
  const raw = await response.json();
  const user = { // transformation
    displayName: `${raw.first_name} ${raw.last_name}`,
    email: raw.email_address,
    joinedDate: new Date(raw.created_at).toLocaleDateString(),
  };
  document.getElementById('profile-name')!.textContent = user.displayName;
  document.getElementById('profile-email')!.textContent = user.email;
}

// ✅ CORRECT: Three functions, each with one job
async function fetchUserProfile(userId: string): Promise<RawApiUser> {
  const response = await fetch(`/api/users/${userId}`);
  if (!response.ok) throw new ApiError(response.status, `/api/users/${userId}`);
  return response.json();
}

function transformUser(raw: RawApiUser): UserProfile {
  return {
    displayName: `${raw.first_name} ${raw.last_name}`,
    email: raw.email_address,
    joinedDate: new Date(raw.created_at).toLocaleDateString(),
  };
}

function renderUserProfile(profile: UserProfile, container: HTMLElement): void {
  container.querySelector('[data-name]')!.textContent = profile.displayName;
  container.querySelector('[data-email]')!.textContent = profile.email;
}

// Orchestration at top level
async function loadAndDisplayUserProfile(userId: string): Promise<void> {
  const raw = await fetchUserProfile(userId);
  const profile = transformUser(raw);
  renderUserProfile(profile, document.getElementById('profile')!);
}
```

### Function Complexity Limits

```typescript
// ❌ WRONG: Cognitive complexity too high — multiple nested conditions
function processOrder(order: Order): Result {
  if (order.items.length > 0) {
    if (order.paymentMethod) {
      if (validatePayment(order.paymentMethod)) {
        if (order.shippingAddress) {
          if (validateAddress(order.shippingAddress)) {
            return fulfill(order);
          } else {
            return { error: 'Invalid address' };
          }
        } else {
          return { error: 'Missing address' };
        }
      } else {
        return { error: 'Invalid payment' };
      }
    } else {
      return { error: 'Missing payment' };
    }
  } else {
    return { error: 'Empty order' };
  }
}

// ✅ CORRECT: Linear flow — early returns reduce nesting
type ValidationResult = 
  | { valid: true; order: ValidatedOrder }
  | { valid: false; error: string };

function validateOrder(order: Order): ValidationResult {
  if (order.items.length === 0) {
    return { valid: false, error: 'Empty order' };
  }
  if (!order.paymentMethod) {
    return { valid: false, error: 'Missing payment' };
  }
  if (!validatePayment(order.paymentMethod)) {
    return { valid: false, error: 'Invalid payment' };
  }
  if (!order.shippingAddress) {
    return { valid: false, error: 'Missing address' };
  }
  if (!validateAddress(order.shippingAddress)) {
    return { valid: false, error: 'Invalid address' };
  }
  return { valid: true, order: order as ValidatedOrder };
}

function processOrder(order: Order): Result {
  const validation = validateOrder(order);
  if (!validation.valid) {
    return { error: validation.error };
  }
  return fulfill(validation.order);
}
```

### Guard Clauses Pattern (Mandatory)

**All validation at the top. No nested if-else trees.**

```typescript
// ❌ WRONG: Deep nesting, hard to follow happy path
function processOrder(order: Order | null, user: User | null) {
  if (order) {
    if (user) {
      if (user.isActive) {
        if (order.items.length > 0) {
          if (order.total > 0) {
            // actual logic buried at level 5
            return chargeAndFulfill(order, user);
          } else {
            return { error: 'Order total must be positive' };
          }
        } else {
          return { error: 'Order must have items' };
        }
      } else {
        return { error: 'User is not active' };
      }
    } else {
      return { error: 'User not found' };
    }
  } else {
    return { error: 'Order not found' };
  }
}

// ✅ CORRECT: Guard clauses — failures first, happy path last
function processOrder(order: Order | null, user: User | null): OrderResult {
  if (!order) return { error: 'Order not found' };
  if (!user) return { error: 'User not found' };
  if (!user.isActive) return { error: 'User is not active' };
  if (order.items.length === 0) return { error: 'Order must have items' };
  if (order.total <= 0) return { error: 'Order total must be positive' };

  // Happy path — clean, no nesting
  return chargeAndFulfill(order, user);
}
```

### Function Parameters

```typescript
// ❌ WRONG: Multiple parameters — easy to confuse order
function createEvent(title: string, date: Date, location: string, isPublic: boolean, maxAttendees: number) {}
// Caller: createEvent('Party', date, 'NYC', true, 50) — is isPublic before or after location?

// ✅ CORRECT: Options object for 3+ parameters
interface CreateEventOptions {
  title: string;
  date: Date;
  location: string;
  isPublic: boolean;
  maxAttendees: number;
}

function createEvent(options: CreateEventOptions): Event {
  const { title, date, location, isPublic, maxAttendees } = options;
  // ...
}

// Caller — self-documenting, order doesn't matter:
createEvent({
  title: 'Party',
  date: new Date('2024-12-31'),
  location: 'NYC',
  isPublic: true,
  maxAttendees: 50,
});

// ✅ CORRECT: Destructuring with defaults
interface SendEmailOptions {
  to: string;
  subject: string;
  body: string;
  from?: string;
  cc?: string[];
  bcc?: string[];
  attachments?: Attachment[];
  priority?: 'high' | 'normal' | 'low';
}

function sendEmail({
  to,
  subject,
  body,
  from = 'noreply@company.com',
  cc = [],
  bcc = [],
  attachments = [],
  priority = 'normal',
}: SendEmailOptions): Promise<void> {
  // Implementation
}
```

### Pure Functions & Side Effects

```typescript
// ❌ WRONG: Side effects make function unpredictable
let totalCalls = 0;
function calculatePrice(base: number, tax: number): number {
  totalCalls++; // Hidden side effect
  return base * (1 + tax);
}

// ❌ WRONG: Mutates input
function addTimestamp<T>(record: T): T {
  record.timestamp = Date.now(); // Mutation!
  return record;
}

// ✅ CORRECT: Pure function — same input, same output, no side effects
function calculatePrice(base: number, tax: number): number {
  return base * (1 + tax);
}

// ✅ CORRECT: Returns new object instead of mutating
function addTimestamp<T>(record: T): T & { timestamp: number } {
  return { ...record, timestamp: Date.now() };
}

// Side effects only at system boundaries (I/O, API calls, DOM)
async function saveUser(user: User): Promise<void> {
  // Side effect is expected and explicit
  await db.users.insert(user);
  await cache.invalidate(`user:${user.id}`);
}
```

> **VIP EXPERT NOTE:** Functions are the building blocks of your application.
> A function that does one thing well is reusable, testable, and composable.
> When reviewing code, ask: "Can I describe what this function does in one 
> sentence without using 'and' or 'or'?" If not, it needs to be split.

---

## 🔷 NAMING STANDARDS

### Variables and Functions

```typescript
// ❌ WRONG: Abbreviations, generic names, acronyms
const d = new Date();
const usr = getUser();
const arr = [1, 2, 3];
const res = await api.get('/users');
let flag = false;
const tmp = process(data);
const data2 = transform(data);
const handleClick = () => {};

// ✅ CORRECT: Descriptive, intention-revealing names
const currentDate = new Date();
const authenticatedUser = await getAuthenticatedUser();
const activeUserIds = [1, 2, 3];
const usersResponse = await api.get('/users');
let hasValidationErrors = false;
const processedPaymentResult = processPayment(paymentData);
const enrichedUserData = transformUserData(rawUserData);
const handleSubmitButtonClick = () => {};

// DOMAIN-SPECIFIC: Use ubiquitous language from the domain
// ❌ WRONG:
const item = await db.get(id);
const total = calculate(item.price, item.qty);

// ✅ CORRECT:
const product = await productRepository.findById(id);
const orderTotal = calculateOrderTotal(product.unitPrice, product.quantity);
```

### Boolean Naming

```typescript
// ❌ WRONG: Unclear boolean names
const active = true;
const visible = false;
const valid = check(input);
const done = false;

// ✅ CORRECT: Prefix with is/has/can/should/will/did
const isActive = true;
const isVisible = false;
const isValid = validateEmailFormat(input);
const hasPermission = checkPermission(user, resource);
const canEdit = user.role === 'admin';
const shouldRedirect = !isAuthenticated;
const willExpireSoon = daysUntilExpiry < 7;
const didSubmit = submitStatus === 'success';

// Functions returning booleans
function isEmpty(value: unknown): boolean {}
function hasRole(user: User, role: string): boolean {}
function canAccess(user: User, resource: Resource): boolean {}
function shouldRetry(error: Error): boolean {}
```

### Constants

```typescript
// ❌ WRONG: Magic numbers scattered in code
if (retries >= 3) { ... }
setTimeout(callback, 30000);
if (password.length < 8) { ... }
const apiUrl = 'https://api.example.com/v1';

// ✅ CORRECT: Named constants in one place
const MAX_RETRY_ATTEMPTS = 3;
const REQUEST_TIMEOUT_MS = 30_000; // underscore separator for readability
const MIN_PASSWORD_LENGTH = 8;
const SUPPORTED_FILE_TYPES = ['jpg', 'png', 'webp'] as const;
const API_BASE_URL = 'https://api.example.com/v1';

// Group related constants in an object or enum
const HTTP_STATUS = {
  OK: 200,
  CREATED: 201,
  NO_CONTENT: 204,
  BAD_REQUEST: 400,
  UNAUTHORIZED: 401,
  FORBIDDEN: 403,
  NOT_FOUND: 404,
  UNPROCESSABLE: 422,
  INTERNAL_SERVER_ERROR: 500,
} as const;

const CACHE_DURATION = {
  SHORT: 60_000,      // 1 minute
  MEDIUM: 300_000,    // 5 minutes
  LONG: 3_600_000,    // 1 hour
  PERMANENT: 86_400_000, // 1 day
} as const;

// Business domain constants
const SUBSCRIPTION_TIERS = {
  FREE: { maxProjects: 1, maxStorageMB: 100 },
  PRO: { maxProjects: 10, maxStorageMB: 1000 },
  ENTERPRISE: { maxProjects: Infinity, maxStorageMB: 10000 },
} as const;
```

### Function Naming Conventions

```typescript
// ACTION + SUBJECT pattern
getUserById()           // fetch by ID
getUsersByRole()        // fetch multiple by criteria
findUserByEmail()       // search with potential null result
fetchUserProfile()      // async data retrieval
loadUserSettings()      // async with caching implications

formatCurrency()        // transform to currency string
parseDateString()       // parse from string
transformApiResponse()  // convert from one format to another
normalizeEmail()        // standardize format

validateEmailFormat()   // check + return bool
assertValidUser()       // throw if invalid
ensureAuthenticated()   // guard/throw pattern

sendWelcomeEmail()      // action with side effect
createUserAccount()     // mutating operation
deleteExpiredSessions() // batch operation

calculateTaxAmount()    // derive a value
computeStatistics()     // complex calculation
derivePermissions()    // calculate from other data

handleSubmit()          // event handler
onUserClick()           // callback prop
processPayment()        // business logic handler
```

> **VIP EXPERT NOTE:** Naming is the most important code documentation. 
> Code is read 10x more than it's written. Spend time finding the right name — 
> it's worth it. When stuck, use the "say it out loud" test: if you can't 
> describe what the function does using its name in a sentence, rename it.

---

## 🔷 ERROR HANDLING ARCHITECTURE

### Error Type Hierarchy

```typescript
// Base application error
class AppError extends Error {
  constructor(
    message: string,
    public readonly code: string,
    public readonly statusCode: number = 500,
    public readonly context?: Record<string, unknown>,
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }

  toJSON() {
    return {
      name: this.name,
      code: this.code,
      message: this.message,
      statusCode: this.statusCode,
    };
  }
}

// Domain-specific error types
class ValidationError extends AppError {
  constructor(
    message: string,
    public readonly fields: Record<string, string>,
  ) {
    super(message, 'VALIDATION_ERROR', 422);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string | number) {
    super(`${resource} with id=${id} not found`, 'NOT_FOUND', 404, { resource, id });
  }
}

class AuthorizationError extends AppError {
  constructor(action: string, resource: string) {
    super(`Not authorized to ${action} ${resource}`, 'UNAUTHORIZED', 403);
  }
}

class ConflictError extends AppError {
  constructor(resource: string, reason: string) {
    super(`${resource} conflict: ${reason}`, 'CONFLICT', 409);
  }
}

class RateLimitError extends AppError {
  constructor(retryAfter: number) {
    super('Rate limit exceeded', 'RATE_LIMIT', 429, { retryAfter });
  }
}

// Usage: callers can catch specific types
try {
  await updateUser(userId, data);
} catch (error) {
  if (error instanceof NotFoundError) {
    return { status: 404, body: error.message };
  }
  if (error instanceof ValidationError) {
    return { status: 422, body: error.fields };
  }
  if (error instanceof AuthorizationError) {
    return { status: 403, body: 'Forbidden' };
  }
  // Unknown error — log full context, return safe response
  logger.error('Unexpected error in updateUser', { error, userId });
  return { status: 500, body: 'Internal server error' };
}
```

### Result Pattern

```typescript
// ✅ Result type pattern — never throws, returns typed success or failure
type Result<T, E extends Error = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

// Helper functions for creating results
function ok<T>(value: T): Result<T, never> {
  return { ok: true, value };
}

function err<E extends Error>(error: E): Result<never, E> {
  return { ok: false, error };
}

// Usage in async functions
async function safeParseJson<T>(text: string): Promise<Result<T, SyntaxError>> {
  try {
    return ok(JSON.parse(text) as T);
  } catch (error) {
    return err(error as SyntaxError);
  }
}

async function fetchUserSafe(id: string): Promise<Result<User, NotFoundError | NetworkError>> {
  try {
    const response = await fetch(`/api/users/${id}`);
    if (response.status === 404) {
      return err(new NotFoundError('User', id));
    }
    if (!response.ok) {
      return err(new NetworkError(`HTTP ${response.status}`));
    }
    return ok(await response.json());
  } catch (error) {
    return err(new NetworkError(String(error)));
  }
}

// Caller never needs try/catch
const result = await safeParseJson<User>(responseText);
if (!result.ok) {
  logger.warn('Failed to parse user JSON', { error: result.error.message });
  return null;
}
const user = result.value; // TypeScript knows this is User

// With pattern matching utility
function matchResult<T, E, R>(
  result: Result<T, E>,
  handlers: {
    ok: (value: T) => R;
    err: (error: E) => R;
  }
): R {
  return result.ok ? handlers.ok(result.value) : handlers.err(result.error);
}
```

### Error Boundaries and Recovery

```typescript
// React Error Boundary
class ErrorBoundary extends React.Component<
  { fallback: React.ReactNode; children: React.ReactNode },
  { hasError: boolean; error?: Error }
> {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    logger.error('React error boundary caught error', { error, errorInfo });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}

// API Error Handler Middleware
function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      error: {
        code: err.code,
        message: err.message,
        ...(process.env.NODE_ENV === 'development' && { stack: err.stack }),
      },
    });
    return;
  }

  // Unexpected error — don't leak details
  logger.error('Unhandled error', { error: err, path: req.path });
  res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
    },
  });
}
```

> **VIP EXPERT NOTE:** Error handling is not an afterthought — it's part of 
> your API design. Use custom error types for domain-specific failures, 
> never swallow errors silently, and always provide context for debugging.
> The Result pattern eliminates the need for try/catch in most business logic.

---

## 🔷 MODULE & FILE ORGANIZATION

### File Structure Template

```
src/
├── components/              # UI components (React)
│   ├── UserProfile/
│   │   ├── UserProfile.tsx         # Component
│   │   ├── UserProfile.test.tsx    # Tests (co-located)
│   │   ├── UserProfile.types.ts    # Component-specific types
│   │   ├── UserProfile.styles.ts   # Styled components/CSS-in-JS
│   │   ├── useUserProfile.ts       # Component-specific hook
│   │   └── index.ts                # Public API (barrel export)
│   └── shared/                     # Shared/reusable components
│       ├── Button/
│       ├── Input/
│       └── Card/
├── hooks/                   # Custom React hooks
│   ├── useLocalStorage.ts
│   ├── useDebounce.ts
│   └── useAuth.ts
├── services/                # External API/DB communication
│   ├── api/
│   │   ├── client.ts
│   │   ├── users.ts
│   │   └── orders.ts
│   └── database/
│       ├── connection.ts
│       └── repositories/
├── utils/                   # Pure utility functions
│   ├── formatters.ts
│   ├── validators.ts
│   └── date-utils.ts
├── types/                   # Shared TypeScript types
│   ├── domain.ts            # Business entities
│   ├── api.ts               # API contracts
│   └── global.d.ts          # Global declarations
├── constants/               # App-wide constants
│   ├── config.ts
│   ├── routes.ts
│   └── messages.ts
├── errors/                  # Error class definitions
│   └── AppError.ts
├── lib/                     # Third-party library configurations
│   ├── db.ts
│   └── redis.ts
└── middleware/              # Express/Fastify middleware
    ├── auth.ts
    ├── errorHandler.ts
    └── rateLimiter.ts
```

### Import Order Convention

```typescript
// 1. Node/platform built-ins
import { readFile } from 'fs/promises';
import path from 'path';
import { URL } from 'url';

// 2. External packages (node_modules)
import React, { useState, useEffect } from 'react';
import { z } from 'zod';
import axios from 'axios';

// 3. Internal absolute imports (aliases)
import { UserService } from '@/services/UserService';
import { AppError } from '@/errors/AppError';
import { formatDate } from '@/utils/formatters';

// 4. Internal relative imports
import { UserProfileHeader } from './UserProfileHeader';
import { useUserData } from './useUserData';
import type { UserProfileProps } from './UserProfile.types';

// Blank line between each group
// Within groups: alphabetical order by import source
```

### Barrel Exports (Use Carefully)

```typescript
// index.ts — only export the PUBLIC API of a module
// ✅ CORRECT:
export { UserProfile } from './UserProfile';
export type { UserProfileProps } from './UserProfile.types';

// DON'T export internal helpers, test utilities, or types meant only for the module
// ❌ WRONG:
// export { UserProfileInternal } from './UserProfileInternal';
// export { testUtils } from './test-utils';

// Organize exports by category
// components/index.ts
export { Button } from './Button';
export { Input } from './Input';
export { Card } from './Card';

// Export variants
export { Button, ButtonVariants } from './Button';
export type { ButtonProps, ButtonSize } from './Button.types';
```

### Feature-Based Organization (Alternative)

```
src/
├── features/
│   ├── auth/
│   │   ├── components/
│   │   │   ├── LoginForm.tsx
│   │   │   └── SignupForm.tsx
│   │   ├── hooks/
│   │   │   ├── useAuth.ts
│   │   │   └── useLogin.ts
│   │   ├── services/
│   │   │   └── authApi.ts
│   │   ├── types/
│   │   │   └── auth.ts
│   │   └── utils/
│   │       └── tokenStorage.ts
│   ├── orders/
│   │   ├── components/
│   │   ├── hooks/
│   │   └── services/
│   └── users/
│       ├── components/
│       ├── hooks/
│       └── services/
├── shared/
│   ├── components/
│   ├── hooks/
│   ├── utils/
│   └── types/
└── app/
    ├── App.tsx
    └── routes.tsx
```

> **VIP EXPERT NOTE:** Consistent organization reduces cognitive load. 
> Every developer should know where to find any piece of code within 10 seconds.
> Co-locate tests and types with their implementations. Use barrel exports 
> sparingly — they can create circular dependencies and slow down builds.

---

## 🔷 COMMENTS & DOCUMENTATION

### What to Comment (and what not to)

```typescript
// ❌ WRONG: Comment explains WHAT (the code already says that)
// increment the count by 1
count++;

// ❌ WRONG: Outdated comment (worse than no comment)
// multiply by 2 to get the result
return value * 3; // someone changed the code but not the comment

// ❌ WRONG: Redundant comments on obvious code
// Get the user
const user = getUser();

// Check if user is active
if (user.isActive) { ... }

// ✅ CORRECT: Comment explains WHY (the code can't say this)
// We sleep for 100ms here because the upstream API has a rate limit of
// 10 requests/second and bursting causes silent failures on their end.
// See: https://docs.partner-api.com/rate-limiting
await sleep(100);

// ✅ CORRECT: Explain non-obvious algorithm or business rule
// VIES VAT validation requires the country code prefix for EU countries
// but NOT for UK after Brexit. The API changed behavior in 2021-01.
const vatNumber = isEuCountry(country)
  ? `${country}${rawVatNumber}`
  : rawVatNumber;

// ✅ CORRECT: Document workarounds and hacks
// HACK: Safari 14 has a bug where flex-basis: auto doesn't work correctly
// with overflow: hidden. We use min-width: 0 as a workaround.
// See: https://bugs.webkit.org/show_bug.cgi?id=210682
const safariFix = { minWidth: 0 };

// ✅ CORRECT: Document intent for complex conditionals
// Check if user can delete: must be owner OR admin, but not if order
// is already shipped or more than 30 days old
const canDelete = (isOwner || isAdmin) && !isShipped && daysSinceCreation < 30;
```

### JSDoc for Public APIs

```typescript
/**
 * Calculates the pro-rated subscription amount for a plan change.
 *
 * @param currentPlan - The user's existing subscription plan
 * @param newPlan - The plan they're switching to
 * @param daysRemaining - Days left in the current billing period
 * @returns Amount in cents to charge (positive) or credit (negative)
 *
 * @throws {ValidationError} If daysRemaining is negative or > billing period
 *
 * @example
 * // User upgrades with 15 days remaining in a 30-day cycle
 * const charge = calculateProration(basicPlan, proPlan, 15);
 * // Returns 2500 (= (50-0) * (15/30) * 100 cents)
 */
function calculateProration(
  currentPlan: SubscriptionPlan,
  newPlan: SubscriptionPlan,
  daysRemaining: number,
): number {
  // ...
}

/**
 * Hook for managing form state with validation.
 *
 * @template T - The shape of the form values
 * @param options - Configuration options
 * @param options.initialValues - Starting values for the form
 * @param options.validate - Function to validate form values
 * @param options.onSubmit - Called when form is submitted with valid data
 *
 * @returns Form state and handlers
 *
 * @example
 * const { values, errors, handleSubmit } = useForm({
 *   initialValues: { email: '' },
 *   validate: (values) => ({ email: isEmail(values.email) ? null : 'Invalid email' }),
 *   onSubmit: async (values) => await api.login(values),
 * });
 */
function useForm<T extends Record<string, unknown>>(options: UseFormOptions<T>): UseFormReturn<T> {
  // ...
}
```

### README Standards

```markdown
# Feature Name

Brief description of what this module does.

## Usage

```typescript
import { myFunction } from './feature';

const result = myFunction(input);
```

## API Reference

### `myFunction(input: InputType): OutputType`

Description of what it does.

- `input`: Description of parameter
- Returns: Description of return value

## Architecture Decisions

### Why X instead of Y?

Brief explanation of architectural choices.
```

> **VIP EXPERT NOTE:** Comments are a liability — they can become outdated 
> and misleading. The best code is self-documenting through clear naming 
> and structure. Reserve comments for: business logic rationale, workarounds, 
> and external references. If you find yourself writing long comments to 
> explain code, refactor the code instead.

---

## 🔷 PERFORMANCE STANDARDS

### Avoid N+1 Queries

```typescript
// ❌ WRONG: N+1 query — 1 query to get users + N queries for each user's profile
async function getUsersWithProfiles(userIds: string[]) {
  const users = await db.users.findMany({ where: { id: { in: userIds } } });
  const result = [];
  for (const user of users) {
    const profile = await db.profiles.findUnique({ where: { userId: user.id } }); // N queries!
    result.push({ ...user, profile });
  }
  return result;
}

// ✅ CORRECT: Batch query — 2 queries total regardless of N
async function getUsersWithProfiles(userIds: string[]) {
  const [users, profiles] = await Promise.all([
    db.users.findMany({ where: { id: { in: userIds } } }),
    db.profiles.findMany({ where: { userId: { in: userIds } } }),
  ]);

  const profileMap = new Map(profiles.map(p => [p.userId, p]));
  return users.map(user => ({ ...user, profile: profileMap.get(user.id) ?? null }));
}

// ✅ CORRECT: Use include/join
async function getUsersWithProfiles(userIds: string[]) {
  return db.users.findMany({
    where: { id: { in: userIds } },
    include: { profile: true }, // Single query with JOIN
  });
}

// DataLoader pattern for GraphQL
const userLoader = new DataLoader<string, User>(async (ids) => {
  const users = await db.users.findMany({ where: { id: { in: ids } } });
  return ids.map(id => users.find(u => u.id === id) ?? new Error(`User ${id} not found`));
});
```

### Memoization

```typescript
// ✅ CORRECT: Memoize expensive pure computations
const expensiveComputed = useMemo(
  () => computeStatistics(largeDataset),
  [largeDataset], // only recompute when dataset changes
);

// ✅ CORRECT: useCallback for stable function references
const handleSubmit = useCallback(
  (event: React.FormEvent) => {
    event.preventDefault();
    onSubmit(formData);
  },
  [onSubmit, formData], // stable reference when deps unchanged
);

// ✅ CORRECT: React.memo for pure components
const ExpensiveList = React.memo(function ExpensiveList({ items, onSelect }) {
  return (
    <ul>
      {items.map(item => (
        <ListItem key={item.id} item={item} onClick={onSelect} />
      ))}
    </ul>
  );
});

// ✅ CORRECT: useMemo for derived data
const filteredAndSorted = useMemo(() => {
  return items
    .filter(item => item.status === 'active')
    .sort((a, b) => b.createdAt - a.createdAt);
}, [items]);

// ❌ WRONG: Over-memoization (premature optimization)
const value = useMemo(() => a + b, [a, b]); // Addition is faster than memo lookup
```

### Code Splitting

```typescript
// ✅ CORRECT: Lazy load routes
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  );
}

// ✅ CORRECT: Dynamic imports for heavy libraries
async function generateReport() {
  const { generatePDF } = await import('./pdf-generator'); // Heavy library loaded on demand
  return generatePDF(data);
}
```

> **VIP EXPERT NOTE:** Performance optimization follows the 80/20 rule: 
> 80% of slowness comes from 20% of code. Profile before optimizing. 
> Use React DevTools Profiler to identify actual bottlenecks. 
> Premature optimization is the root of all evil — but so is ignoring 
> obvious performance anti-patterns like N+1 queries.

---

## 🔷 FILE SIZE LIMITS (Hard Rules)

| Type | Maximum | Action if Exceeded | Rationale |
|------|---------|-------------------|-----------|
| Single file | 300 lines | Split by responsibility | Cognitive load limit |
| Single function | 40 lines | Extract sub-functions | Single responsibility |
| Single component | 150 lines | Decompose into children | Component complexity |
| Single test file | 500 lines | Split by describe block | Test organization |
| Import block | 30 lines | Use barrel exports | Readability |
| Type definition | 100 lines | Split related types | Maintainability |
| Class | 200 lines | Apply composition | Class complexity |
| Hook | 80 lines | Extract logic to utils | Hook responsibility |

### Exception Handling

```typescript
// If a file MUST exceed limits, document why at the top:
/**
 * FILE SIZE EXCEPTION: This file is 450 lines due to auto-generated
 * GraphQL type definitions. Do not manually edit — regenerate via
 * `npm run generate:types` instead.
 * 
 * Exception approved by: [Name] on [Date]
 * Review date: [Date + 6 months]
 */
```

---

## 🔷 SOLID PRINCIPLES IN PRACTICE

### S — Single Responsibility (Already Covered Above)

A class or module should have one, and only one, reason to change.

### O — Open/Closed Principle

```typescript
// ❌ WRONG: Adding a new payment method requires modifying existing code
function processPayment(method: string, amount: number) {
  if (method === 'credit_card') { /* ... */ }
  else if (method === 'paypal') { /* ... */ }
  else if (method === 'crypto') { /* ... */ }  // Added later — touches existing code
}

// ✅ CORRECT: Open for extension, closed for modification
interface PaymentProcessor {
  readonly type: string;
  process(amount: number): Promise<PaymentResult>;
  validate(): boolean;
}

class CreditCardProcessor implements PaymentProcessor {
  readonly type = 'credit_card';
  validate() { /* validate card */ return true; }
  async process(amount: number): Promise<PaymentResult> { /* ... */ }
}

class PayPalProcessor implements PaymentProcessor {
  readonly type = 'paypal';
  validate() { return true; }
  async process(amount: number): Promise<PaymentResult> { /* ... */ }
}

// Adding crypto = add new class, don't touch existing code:
class CryptoProcessor implements PaymentProcessor {
  readonly type = 'crypto';
  validate() { /* validate wallet */ return true; }
  async process(amount: number): Promise<PaymentResult> { /* ... */ }
}

// Registry pattern:
class PaymentProcessorRegistry {
  private processors = new Map<string, PaymentProcessor>();
  
  register(processor: PaymentProcessor) {
    this.processors.set(processor.type, processor);
  }
  
  get(type: string): PaymentProcessor {
    const processor = this.processors.get(type);
    if (!processor) throw new Error(`Unknown payment method: ${type}`);
    return processor;
  }
}

const registry = new PaymentProcessorRegistry();
registry.register(new CreditCardProcessor());
registry.register(new PayPalProcessor());
registry.register(new CryptoProcessor());

function processPayment(method: string, amount: number): Promise<PaymentResult> {
  const processor = registry.get(method);
  if (!processor.validate()) {
    throw new ValidationError(`Invalid ${method} details`);
  }
  return processor.process(amount);
}
```

### L — Liskov Substitution

```typescript
// Subtypes must be substitutable for their base types
interface Animal {
  makeSound(): string;
}

class Dog implements Animal {
  makeSound() { return 'woof'; }
}

class Cat implements Animal {
  makeSound() { return 'meow'; }
}

// Any function expecting Animal works with any subtype
function announce(animal: Animal) {
  console.log(animal.makeSound());
}

announce(new Dog()); // 'woof'
announce(new Cat()); // 'meow'

// ❌ WRONG: Violation — subtype changes behavior
class SilentCat implements Animal {
  makeSound() { 
    throw new Error('Cats are silent'); // Breaks substitution!
  }
}
```

### I — Interface Segregation

```typescript
// ❌ WRONG: Fat interface forces implementations to have unnecessary methods
interface Worker {
  work(): void;
  eat(): void;
  sleep(): void;
}

class Robot implements Worker {
  work() { /* OK */ }
  eat() { throw new Error('Robots don\'t eat'); } // Violation!
  sleep() { throw new Error('Robots don\'t sleep'); } // Violation!
}

// ✅ CORRECT: Split into focused interfaces
interface Workable {
  work(): void;
}

interface Feedable {
  eat(): void;
}

interface Restable {
  sleep(): void;
}

class Human implements Workable, Feedable, Restable {
  work() { /* ... */ }
  eat() { /* ... */ }
  sleep() { /* ... */ }
}

class Robot implements Workable {
  work() { /* ... */ }
}
```

### D — Dependency Inversion

```typescript
// ❌ WRONG: Business logic depends on concrete implementation
class OrderService {
  private db = new PostgresDatabase();        // Hard dependency
  private mailer = new SendGridMailer();       // Hard dependency

  async createOrder(data: OrderInput): Promise<Order> {
    const order = await this.db.insert('orders', data);
    await this.mailer.send(order.userId, 'Order confirmed');
    return order;
  }
}
// Cannot test without real Postgres and SendGrid!

// ✅ CORRECT: Depend on abstractions (interfaces), inject implementations
interface Database {
  insert<T>(table: string, data: unknown): Promise<T>;
  findById<T>(table: string, id: string): Promise<T | null>;
}

interface EmailService {
  send(to: string, subject: string, body: string): Promise<void>;
}

class OrderService {
  constructor(
    private readonly db: Database,           // Abstraction
    private readonly mailer: EmailService,   // Abstraction
  ) {}

  async createOrder(data: OrderInput): Promise<Order> {
    const order = await this.db.insert<Order>('orders', data);
    await this.mailer.send(order.userId, 'Order confirmed', orderTemplate(order));
    return order;
  }
}

// In production:
const orderService = new OrderService(new PostgresDatabase(), new SendGridMailer());

// In tests:
const orderService = new OrderService(new InMemoryDatabase(), new MockMailer());
// No real DB, no real email — fast, isolated, reliable tests!
```

> **VIP EXPERT NOTE:** SOLID principles aren't academic exercises — they directly 
> prevent maintenance nightmares. Start with Single Responsibility and Dependency 
> Inversion; they provide the most immediate value. Don't over-abstract, but 
> design for the changes you know are coming.

---

## 🔷 CLEAN ARCHITECTURE LAYERS

```
┌─────────────────────────────────────────────────────────────┐
│  PRESENTATION (UI, API Routes, Controllers)              │
│  • Handles HTTP/WebSocket, renders views                 │
│  • Calls use cases, never accesses DB directly            │
│  • Transforms input/output for transport                  │
├─────────────────────────────────────────────────────────────┤
│  APPLICATION (Use Cases, Services)                        │
│  • Orchestrates business operations                       │
│  • Contains NO framework-specific code                    │
│  • Defines repository interfaces                          │
├─────────────────────────────────────────────────────────────┤
│  DOMAIN (Entities, Value Objects, Business Rules)         │
│  • Pure logic, no dependencies on anything external        │
│  • This is the most stable, testable layer                │
│  • Defines core business invariants                        │
├─────────────────────────────────────────────────────────────┤
│  INFRASTRUCTURE (Database, Email, File System, APIs)      │
│  • Implements interfaces defined in Application layer      │
│  • Framework/library specific code lives here              │
│  • External service adapters                               │
└─────────────────────────────────────────────────────────────┘

Dependency rule: Inner layers NEVER import from outer layers.
  Presentation → Application → Domain ← Infrastructure
  (Infrastructure implements Domain interfaces)
```

### Code Structure Example:

```
src/
├── presentation/         # Express routes, React components
│   ├── routes/
│   │   ├── userRoutes.ts
│   │   └── orderRoutes.ts
│   ├── middleware/
│   │   ├── auth.ts
│   │   └── validation.ts
│   └── controllers/
│       ├── UserController.ts
│       └── OrderController.ts
├── application/          # Use cases, DTOs, mappers
│   ├── use-cases/
│   │   ├── CreateOrder.ts
│   │   ├── GetUserProfile.ts
│   │   └── CancelOrder.ts
│   ├── dto/
│   │   ├── CreateOrderRequest.ts
│   │   └── OrderResponse.ts
│   └── services/
│       └── OrderService.ts
├── domain/               # Entities, value objects, business rules
│   ├── entities/
│   │   ├── User.ts
│   │   ├── Order.ts
│   │   └── Product.ts
│   ├── value-objects/
│   │   ├── Money.ts
│   │   ├── Email.ts
│   │   └── Address.ts
│   └── interfaces/       # Repository interfaces (not implementations)
│       ├── UserRepository.ts
│       └── OrderRepository.ts
└── infrastructure/       # Database, email, external API implementations
    ├── database/
    │   ├── connection.ts
    │   └── repositories/
    │       ├── PrismaUserRepository.ts
    │       └── PrismaOrderRepository.ts
    ├── email/
    │   └── SendGridEmailService.ts
    └── external-apis/
        └── PaymentGateway.ts
```

> **VIP EXPERT NOTE:** Clean Architecture adds ceremony but pays dividends 
> in testability and adaptability. The Domain layer should be pure TypeScript 
> with zero dependencies. You should be able to test core business logic 
> without a database, HTTP server, or any framework. This is the key to 
> fast, reliable tests.

---

## 🔷 CODE REVIEW PRIORITIES

> **Review in this order** (most impactful issues first):

```
Priority 1 — CORRECTNESS (Does it work?)
  □ Logic is correct for all inputs (trace with values)
  □ Edge cases handled (null, empty, max, min, 0, NaN)
  □ Error paths produce correct behavior
  □ Async operations properly awaited
  □ Race conditions considered

Priority 2 — SECURITY (Is it safe?)
  □ No injection vectors (SQL, XSS, command)
  □ No exposed secrets or PII in logs
  □ Auth/authz on all protected endpoints
  □ Input validation on all user data
  □ Rate limiting for sensitive endpoints

Priority 3 — RELIABILITY (Will it break?)
  □ Errors handled, not swallowed
  □ State mutations are safe (no races)
  □ Tests exist for critical paths
  □ Environment variables validated at startup
  □ Circuit breakers for external calls

Priority 4 — MAINTAINABILITY (Can others understand it?)
  □ Clear naming, no abbreviations
  □ Functions < 40 lines, files < 300 lines
  □ No magic numbers
  □ Comments explain WHY, not WHAT
  □ No duplicated code

Priority 5 — PERFORMANCE (Is it fast enough?)
  □ No N+1 queries
  □ No blocking operations on main thread
  □ Expensive computations memoized
  □ Appropriate data structures used
```

### Review Checklist

Before submitting code for review:

```
□ All functions have explicit return types
□ All parameters have explicit types
□ No `any` types without justification
□ No magic numbers/strings — use constants
□ All errors handled explicitly
□ No silent failures
□ No nested conditionals > 2 levels
□ Functions are < 40 lines
□ File is < 300 lines
□ Tests cover happy path and error paths
□ No console.log in production code
□ No hardcoded secrets or credentials
□ Imports are organized correctly
□ Comments explain WHY, not WHAT
□ No commented-out code
□ Git commits are atomic with clear messages
```

> **VIP EXPERT NOTE:** Code review is not about finding fault — it's about 
> collective ownership and knowledge sharing. The reviewer learns the code, 
> the author learns from feedback. Never approve code you don't understand. 
> Ask questions. If something is unclear to you, it will be unclear to 
> future maintainers.

---

## 🔷 TESTING STANDARDS

### Test Structure (Arrange-Act-Assert)

```typescript
describe('calculateDiscount', () => {
  it('should apply 10% discount for premium customers', () => {
    // Arrange
    const order = createOrder({ total: 100, customerTier: 'premium' });
    
    // Act
    const result = calculateDiscount(order);
    
    // Assert
    expect(result).toBe(10);
  });
  
  it('should return 0 for standard customers', () => {
    const order = createOrder({ total: 100, customerTier: 'standard' });
    
    const result = calculateDiscount(order);
    
    expect(result).toBe(0);
  });
});
```

### Test Coverage Requirements

| Category | Minimum Coverage | Notes |
|----------|-----------------|-------|
| Business Logic | 90% | Core calculations, decisions |
| Utilities | 80% | Pure functions, helpers |
| API Handlers | 70% | Request/response handling |
| UI Components | 60% | Critical user flows |
| Infrastructure | 50% | Database, external APIs |

> **VIP EXPERT NOTE:** High coverage doesn't guarantee quality, but low 
> coverage guarantees risk. Focus on behavior, not implementation details. 
> Test what the code does, not how it does it. Mock external dependencies, 
> not internal implementation.

---

*This module provides mandatory code quality standards. All code must meet these requirements.*
